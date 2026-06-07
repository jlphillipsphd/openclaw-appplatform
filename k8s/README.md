# OpenClaw — Kubernetes Manifests

Deploys the OpenClaw gateway container (built from the project `Dockerfile`) to Kubernetes, mirroring the configuration of the DigitalOcean App Platform worker defined in `app.yaml`.

## Files

| File | Description |
|------|-------------|
| `namespace.yaml` | Creates the `openclaw` namespace |
| `secret.yaml` | Sensitive environment variables (API keys, tokens, passwords) |
| `configmap-env.yaml` | Non-sensitive environment variables and feature flags |
| `configmap-gateway.yaml` | Overrides the default openclaw gateway config (`openclaw.default.json`) mounted into the container. Keeps `bind: loopback` (gateway only listens on `127.0.0.1`) — Tailscale serve acts as the reverse proxy, accepting connections from your tailnet and forwarding them to localhost. Also defines model providers. |
| `pvc.yaml` | 10 GiB `ReadWriteOnce` PersistentVolumeClaim for `/data` — stores openclaw state, the user workspace, and Tailscale node identity |
| `deployment.yaml` | Single-replica Deployment. Runs as root (required by the s6-overlay init system), uses a `Recreate` strategy (the RWO PVC cannot be mounted by two pods simultaneously), and enforces a 2 GiB memory minimum. |
| `kustomization.yaml` | Kustomize root — apply all resources with a single command |

## Before You Deploy

### 1. Set your container image

Edit `deployment.yaml` and replace the placeholder with your actual image reference:

```yaml
image: REPLACE_ME
# e.g. ghcr.io/your-org/openclaw-appplatform:latest
```

If your image is in a private registry, also add `imagePullSecrets` to the pod spec.

### 2. Populate secrets

Edit `secret.yaml` and fill in plaintext values under `stringData` (Kubernetes base64-encodes them automatically on apply).

| Key | Required | Description |
|-----|----------|-------------|
| `OPENCLAW_GATEWAY_TOKEN` | **Yes** | Token used to authenticate with the Control UI. Choose any strong random string. |
| `TS_AUTHKEY` | When `TAILSCALE_ENABLE=true` | Auth key from [Tailscale admin](https://login.tailscale.com/admin/settings/keys) |
| `MTSU_API_KEY` | When using the `mtsu` model provider | API key for `https://openai.cs.mtsu.edu/api` |
| `GRADIENT_API_KEY` | Optional | API key for the DigitalOcean AI inference provider |
| `RESTIC_SPACES_ACCESS_KEY_ID` | When `ENABLE_SPACES=true` | DigitalOcean Spaces access key |
| `RESTIC_SPACES_SECRET_ACCESS_KEY` | When `ENABLE_SPACES=true` | DigitalOcean Spaces secret key |
| `RESTIC_PASSWORD` | When `ENABLE_SPACES=true` | Encryption password for restic backups. If left empty the container generates one and saves it to `/data/.openclaw/.restic-password` — record it if you ever need to restore from backup outside the container. |

### 3. Review feature flags

Feature flags live in `configmap-env.yaml`. The current configuration has the following features enabled:

| Flag | Value | Notes |
|------|-------|-------|
| `TAILSCALE_ENABLE` | `true` | Tailscale and ngrok are mutually exclusive — keep `ENABLE_NGROK=false` |
| `ENABLE_SPACES` | `true` | Restic backups to DigitalOcean Spaces every 30 seconds |
| `ENABLE_UI` | `true` | Serves the OpenClaw Control UI |
| `ENABLE_NGROK` | `false` | |
| `SSH_ENABLE` | `false` | |

Other values you may want to review:

- **`STABLE_HOSTNAME`** / **`TS_HOSTNAME`** — the node name that appears in your Tailscale admin console and is used as the restic backup path prefix. Both default to `openclaw`.
- **`RESTIC_SPACES_ENDPOINT`** — defaults to `tor1.digitaloceanspaces.com`. Change to match your Spaces region.
- **`RESTIC_SPACES_BUCKET`** — defaults to `openclaw-backup`.

### 4. Check your StorageClass

`pvc.yaml` does not specify a `storageClassName`, so the cluster's default StorageClass is used. If your cluster has no default, or you want to use a specific one (e.g. `do-block-storage`, `gp2`, `standard`), uncomment and set the field at the bottom of `pvc.yaml`:

```yaml
storageClassName: "your-storage-class"
```

## Deploying

Apply everything at once with Kustomize (built into `kubectl` since v1.14):

```bash
kubectl apply -k k8s/
```

Check that the pod starts up:

```bash
kubectl -n openclaw get pods -w
```

The startup probe gives the container up to ~5 minutes to initialise (Node.js and s6 services take a moment). Once the pod is `Running` and `Ready`, access the Control UI via Tailscale:

- Open `https://<hostname>.tailnet-name.ts.net` in your browser (where `<hostname>` is the value of `TS_HOSTNAME`, defaulting to `openclaw`)
- Enter your `OPENCLAW_GATEWAY_TOKEN` when prompted

Tailscale serve acts as a reverse proxy on the tailnet, forwarding HTTPS traffic to the gateway on `localhost:18789`. No Kubernetes Service or port-forward is required.

## Updating Configuration

Config changes (feature flags, model list, etc.) require the pod to be restarted to pick them up, since the gateway config is seeded from `openclaw.default.json` at startup:

```bash
# Apply updated manifests then restart
kubectl apply -k k8s/
kubectl -n openclaw rollout restart deployment/openclaw
```

## Tearing Down

```bash
kubectl delete -k k8s/
```

> **Note:** This does **not** delete the PersistentVolumeClaim (and its underlying volume) by default, preserving your workspace data. To delete it explicitly:
> ```bash
> kubectl -n openclaw delete pvc openclaw-data
> ```
