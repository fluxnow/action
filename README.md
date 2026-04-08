# FluxNow Deploy Action

Build, push, and deploy your app to [FluxNow](https://fluxnow.dev) — or promote staging to production.

## Features

- **Build & Deploy** — Dockerfile or [Railpack](https://railpack.io) (auto-detect)
- **Promote** — promote staging image tag to production with one command
- **Monorepo** — per-service `values-path` and `sibling-values` for full preview environments
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
# .github/workflows/backend.yml
name: "Backend: Build & Deploy"
on:
  push:
    branches: [main]
    paths: [backend/**, deploy/uca-backend/**]
  pull_request:
    paths: [backend/**, deploy/uca-backend/**]
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
          dockerfile: backend/Dockerfile
          context: ./backend
          values-path: deploy/uca-backend/values.yaml

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
          dockerfile: backend/Dockerfile
          context: ./backend
          values-path: deploy/uca-backend/values.yaml
          sibling-values: deploy/uca-web/values.yaml
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
| `fluxnow-token` | FluxNow authentication token (`FLUXNOW_TOKEN` secret) | **required** |
| `mode` | `build-and-deploy` or `promote` | `build-and-deploy` |
| `target-env` | Target environment for promote mode (e.g. `prod`) | `prod` |
| `build-strategy` | `auto`, `dockerfile`, or `railpack` | `auto` |
| `dockerfile` | Path to Dockerfile | `Dockerfile` |
| `context` | Build context directory | `.` |
| `values-path` | Path to staging values file | `deploy/values.yaml` |
| `sibling-values` | Comma-separated sibling service values paths (monorepo preview fallback) | |
| `fluxnow-api-url` | FluxNow API base URL (self-hosted override) | `https://onboarding.openfab.dev` |

## Outputs

| Output | Description |
|--------|-------------|
| `image-tag` | Image tag (built or promoted) |
| `image-uri` | Full image URI |
| `deployment-url` | URL of the deployed environment |

## How `sibling-values` works

In a monorepo, a frontend-only PR triggers only the frontend workflow. The backend preview pod fails with `ImagePullBackOff` because no PR-specific backend image exists.

With `sibling-values`, the action retags each sibling's current staging image with the PR-specific tag using [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane). The ArgoCD ApplicationSet finds images for all services and the preview works end-to-end.

## License

MIT
