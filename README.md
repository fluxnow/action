# FluxNow Deploy Action

Build, push, and deploy your app to [FluxNow](https://fluxnow.dev) — or promote staging to production.

## Features

- **Build & Deploy** — Dockerfile or [Railpack](https://railpack.io) (auto-detect)
- **Promote** — promote staging image tag to production with one command
- **Monorepo** — per-service `values-path`, `sibling-values` for full preview environments
- **Service Refs** — Railway-style cross-service hostname resolution via `fluxnow.yaml`
- **Preview Environments** — automatic per-PR preview with ArgoCD sync
- **Harbor Registry** — secure token exchange, no registry credentials in your repo

## Quick Start

### Single-app repository

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      deployments: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - uses: fluxnow/action@v1
        with:
          fluxnow-token: ${{ secrets.FLUXNOW_TOKEN }}
```

### Monorepo (multiple services)

Each service has its own workflow with `paths` filter and `sibling-values` for preview fallback:

```yaml
# .github/workflows/frontend.yml
name: "Frontend: Build & Deploy"
on:
  push:
    branches: [main]
    paths: [frontend/**, deploy/uca-web/**]
  pull_request:
    paths: [frontend/**, deploy/uca-web/**]
    types: [opened, synchronize, reopened]

jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      deployments: write
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: fluxnow/action@v1
        with:
          fluxnow-token: ${{ secrets.FLUXNOW_TOKEN }}
          dockerfile: frontend/Dockerfile
          context: ./frontend
          values-path: deploy/uca-web/values.yaml

  preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: fluxnow/action@v1
        with:
          fluxnow-token: ${{ secrets.FLUXNOW_TOKEN }}
          dockerfile: frontend/Dockerfile
          context: ./frontend
          values-path: deploy/uca-web/values.yaml
          sibling-values: deploy/uca-backend/values.yaml
```

### Promote staging to production

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy to Production
on:
  workflow_dispatch:

jobs:
  promote:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      deployments: write
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: fluxnow/action@v1
        with:
          fluxnow-token: ${{ secrets.FLUXNOW_TOKEN }}
          mode: promote
          target-env: prod
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `fluxnow-token` | **Required.** FluxNow authentication token from `FLUXNOW_TOKEN` repository secret. Used to exchange for short-lived Harbor registry credentials. | — |
| `mode` | Operation mode. `build-and-deploy` builds the image, pushes to registry, and updates manifests. `promote` copies the current staging image tag to a target environment (e.g. prod). | `build-and-deploy` |
| `target-env` | Target environment name for `promote` mode. The action reads the staging tag from `values-path` and writes it to `deploy/values-{target-env}.yaml`. | `prod` |
| `build-strategy` | How to build the container image. `auto` uses Dockerfile if present, otherwise Railpack. `dockerfile` forces Docker build. `railpack` forces zero-config Railpack build. | `auto` |
| `dockerfile` | Path to the Dockerfile, relative to the repository root. Only used when `build-strategy` is `dockerfile` or `auto` with a Dockerfile present. | `Dockerfile` |
| `context` | Docker build context directory. For monorepos, set to the service subdirectory (e.g. `./backend`). | `.` |
| `values-path` | Path to the Helm staging values file. The action reads `image.repository` from this file (monorepo image routing) and writes the new `image.tag` after build. For monorepos: `deploy/{service}/values.yaml`. | `deploy/values.yaml` |
| `sibling-values` | Comma-separated paths to sibling service values files. **Monorepo only.** When a PR only changes one service, the action retags each sibling's current staging image with the PR-specific tag using `crane`. This prevents `ImagePullBackOff` on unchanged services in preview. Example: `deploy/uca-backend/values.yaml`. | — |
| `fluxnow-api-url` | FluxNow API base URL. Override for self-hosted FluxNow installations. | `https://onboarding.openfab.dev` |

## Outputs

| Output | Description |
|--------|-------------|
| `image-tag` | Short SHA image tag that was built (e.g. `sha-abc1234`) or promoted. |
| `image-uri` | Full image URI including registry, project, and repository (e.g. `harbor.openfab.dev/cust-acme/myapp`). Only set in `build-and-deploy` mode. |
| `deployment-url` | URL of the deployed environment (e.g. `https://acme-myapp.openfab.dev` for staging, `https://acme-myapp-pr-42.openfab.dev` for preview). |

## `deploy/values.yaml` Reference

The Helm values file that FluxNow reads and updates. Here's what each field does:

```yaml
# Service identity — must match fluxnow.yaml metadata.name
appName: myapp
customer: acme

# Container port your app listens on
port: 3000

# Container image — tag is auto-updated by the action on each push to main
image:
  repository: myapp    # Optional, for monorepos (overrides default repo name)
  tag: sha-abc1234     # Updated automatically by the action

# Database — set to true if your app needs a PostgreSQL database
# FluxNow provisions a PG18 branch per PR for preview environments
db:
  enabled: true

# Custom environment variables injected into the pod
# For monorepos: use API_BACKEND_HOST from refs (see Service Refs below)
env:
  - name: LOG_LEVEL
    value: "info"
  - name: API_BACKEND_HOST              # Resolved from refs for staging
    value: "acme-api.openfab.dev"

# Per-environment resource secrets from OpenBao (e.g. bot tokens, API keys)
# Each entry creates an ExternalSecret that bulk-extracts all keys from the path
resourceSecrets:
  - name: telegram-pool
    path: telegram-pool

# Individual secret key mappings from OpenBao
secrets:
  - name: cookie_secret
    key: app/cookie-secret

# Custom domains (prod only) — auto-provisions TLS via cert-manager
customDomains:
  - myapp.com

# Health check probes (override defaults for slow-starting apps)
probes:
  readiness:
    path: /healthz
    initialDelaySeconds: 10
  liveness:
    path: /healthz
    initialDelaySeconds: 15
```

## Service Refs (Railway-style)

For monorepos where a frontend needs to reach a backend, declare `refs` in `fluxnow.yaml`:

```yaml
# fluxnow.yaml
kind: Monorepo
apps:
  - name: uca-backend
    path: backend
    spec:
      runtime: python
      port: 8080

  - name: uca-web
    path: frontend
    spec:
      runtime: node
      port: 80
    refs:
      API_BACKEND_HOST: uca-backend    # Platform resolves this per environment
```

The platform automatically resolves `API_BACKEND_HOST` to the correct hostname:

| Environment | `API_BACKEND_HOST` value | Source |
|---|---|---|
| **Staging** | `acme-uca-backend.openfab.dev` | Baked into `deploy/uca-web/values.yaml` |
| **Preview** | `acme-uca-backend-pr-{N}.openfab.dev` | Injected by ApplicationSet template |
| **Production** | `api.myapp.com` | Set manually in `deploy/uca-web/values-prod.yaml` |

Your app builds the full URL from the hostname:

```sh
# docker-entrypoint.sh
cat > /usr/share/nginx/html/config.json <<JSON
{"apiBaseUrl": "https://${API_BACKEND_HOST:-fallback.example.com}/api/v1"}
JSON
```

This decouples infrastructure from application URL paths — the platform provides the hostname, your app owns the protocol and path.

## How `sibling-values` works

In a monorepo, a frontend-only PR triggers only the frontend workflow. The backend preview pod fails with `ImagePullBackOff` because no PR-specific backend image exists.

With `sibling-values`, the action retags each sibling's current staging image with the PR-specific tag using [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane). The ArgoCD ApplicationSet finds images for all services and the preview works end-to-end.

**Note:** Resource provisioning (databases, Telegram bots, secrets) is handled independently by the FluxNow onboarding service. When a PR is opened, the `pull_request` webhook triggers provisioning for **all** services defined in `fluxnow.yaml` — regardless of which files changed. `sibling-values` only solves the container image availability problem.

## How it works

1. Exchanges `FLUXNOW_TOKEN` for short-lived registry credentials
2. Auto-detects build strategy (Dockerfile or Railpack)
3. Builds and pushes image to Harbor registry
4. Retags sibling staging images for preview (if `sibling-values` set)
5. Creates GitHub Deployment with environment URL
6. On push to main: updates `values.yaml` with new image tag (with retry for concurrent monorepo pushes)
7. On PR: adds `preview` label — ArgoCD syncs the preview environment

## Security

- Fork PRs are automatically skipped
- Registry credentials are short-lived (per-run exchange)
- Passwords masked in workflow logs
- No registry credentials stored in customer repos

## License

MIT
