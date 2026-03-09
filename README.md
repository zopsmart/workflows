# Reusable Deployment Workflows

Reusable GitHub Actions workflows for building, pushing, and deploying containerized applications to Kubernetes — across any cloud.

**One workflow file. Any registry. Any cluster. Auto-detected everything.**

## Features

- Multi-cloud support (GCP, AWS, Azure, self-hosted)
- Auto-detected language, registry type, and cluster auth
- Stage + production deployment with image retagging
- ConfigMap management from env files
- Helm and kubectl deployment methods
- Config-only updates (no rebuild when only env changes)

## Supported Platforms

| Registry | Cluster |
|----------|---------|
| Google Artifact Registry (GAR) | GKE |
| Google Container Registry (GCR) | EKS |
| Amazon ECR | AKS |
| Azure Container Registry (ACR) | Any (kubeconfig) |
| GitHub Container Registry (GHCR) | |
| Docker Hub | |

## Quick Start

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [development]
  release:
    types: [published]

jobs:
  stage:
    if: github.ref == 'refs/heads/development'
    uses: zopsmart/workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: 'go build -o main ./cmd/...'
    secrets: inherit

  prod:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: zopsmart/workflows/.github/workflows/prod-deploy.yaml@main
    with:
      SVC_NAME: my-service
    secrets: inherit
```

Set your registry and cluster details via [repository variables and secrets](#configuration), then `secrets: inherit` passes them through automatically.

See [`examples/`](examples/) for cloud-specific setups (GAR+GKE, ECR+EKS, ACR+AKS, GHCR, Docker Hub).

## Workflows

### stage-deploy.yaml

Builds, pushes, and deploys to staging on push to `development`.

**What it does:**
1. Auto-detects language from build command
2. Checks for code vs config-only changes
3. Builds application (Go/Node/generic)
4. Builds and pushes Docker image
5. Deploys to Kubernetes cluster
6. Updates ConfigMap from env file

#### Required Inputs

| Input | Description |
|-------|-------------|
| `SVC_NAME` | Service name (Kubernetes deployment/cronjob name) |
| `BUILD_COMMAND` | Build command (e.g., `go build -o main ./cmd/...` or `yarn build`) |

#### Optional Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `LANGUAGE` | Auto-detected | Build language: `go`, `node`, or `generic` |
| `DOCKER_FILE_PATH` | `.` | Path to Dockerfile directory |
| `BUILD_ARGUMENTS` | | Docker build arguments |
| `GO_VERSION` | `1.20` | Go version (when `LANGUAGE=go`) |
| `NODE_VERSION` | `18` | Node version (when `LANGUAGE=node`) |
| `REGISTRY_TYPE` | Auto-detected | Registry provider: `gar`, `gcr`, `ecr`, `acr`, `dockerhub`, `ghcr`, `custom` |
| `IMAGE_REGISTRY` | From `vars.*` | Registry URL |
| `REGISTRY_PROJECT` | From `vars.*` | Project/namespace within registry |
| `REGISTRY_REPO` | From `vars.*` | Repository name within project |
| `CLUSTER_PROJECT` | From `vars.*` | GCP project (required for GKE) |
| `CLUSTER_NAME` | From `vars.*` | Kubernetes cluster name |
| `CLUSTER_REGION` | From `vars.*` | Cluster region |
| `AZURE_RESOURCE_GROUP` | From `vars.*` | Azure resource group (required for AKS) |
| `NAMESPACE` | `{APP_NAME}-stage` | Kubernetes namespace |
| `TYPE` | `deployment` | Workload type: `deployment` or `cron` |
| `DEPLOY_METHOD` | `kubectl` | Deploy method: `kubectl` or `helm` |
| `ENV_FILE_PATH` | `configs/.stage.env` | Path to env file (empty string skips configmap) |
| `REACT_APP` | `false` | Enable React-specific configmap format |

### prod-deploy.yaml

Retags the staging image and deploys to production on release tag.

**What it does:**
1. Validates semantic version tag
2. Retags SHA image with release version
3. Deploys to production cluster
4. Updates ConfigMap from env file

#### Required Inputs

| Input | Description |
|-------|-------------|
| `SVC_NAME` | Service name (Kubernetes deployment/cronjob name) |

#### Optional Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `REGISTRY_TYPE` | Auto-detected | Registry provider |
| `IMAGE_REGISTRY` | From `vars.*` | Registry URL |
| `REGISTRY_PROJECT` | From `vars.*` | Project/namespace within registry |
| `REGISTRY_REPO` | From `vars.*` | Repository name within project |
| `CLUSTER_PROJECT` | From `vars.*` | GCP project (required for GKE) |
| `CLUSTER_NAME` | From `vars.*` | Kubernetes cluster name |
| `CLUSTER_REGION` | From `vars.*` | Cluster region |
| `AZURE_RESOURCE_GROUP` | From `vars.*` | Azure resource group (required for AKS) |
| `NAMESPACE` | `{APP_NAME}` | Kubernetes namespace |
| `TYPE` | `deployment` | Workload type: `deployment` or `cron` |
| `DEPLOY_METHOD` | `kubectl` | Deploy method: `kubectl` or `helm` |
| `ENV_FILE_PATH` | `configs/.prod.env` | Path to env file |
| `REACT_APP` | `false` | Enable React-specific configmap format |

## Configuration

### Repository Variables

Set in **Settings > Secrets and variables > Actions > Variables**:

| Variable | Description | Example |
|----------|-------------|---------|
| `IMAGE_REGISTRY` | Registry URL | `us-docker.pkg.dev` |
| `REGISTRY_PROJECT` | Project/namespace | `my-gcp-project` |
| `REGISTRY_REPO` | Repository name | `docker-registry` |
| `CLUSTER_PROJECT` | GCP project (GKE) | `my-gcp-project` |
| `CLUSTER_NAME` | Cluster name | `my-cluster` |
| `CLUSTER_REGION` | Cluster region | `us-central1` |
| `AZURE_RESOURCE_GROUP` | Azure RG (AKS) | `my-resource-group` |
| `APP_NAME` | Application name | `my-service` |
| `STAGE_NAMESPACE` | Staging namespace | `my-app-stage` |
| `PROD_NAMESPACE` | Production namespace | `my-app` |

### Repository Secrets

Set in **Settings > Secrets and variables > Actions > Secrets**:

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (GCP SA JSON, AWS JSON, Azure JSON, or token) |
| `STAGE_CLUSTER_CREDENTIALS` | Stage cluster auth |
| `PROD_CLUSTER_CREDENTIALS` | Prod cluster auth |
| `CLUSTER_CREDENTIALS` | Fallback cluster auth (used if env-specific not provided) |
| `PAT` | GitHub PAT for private dependencies (optional) |
| `NPM_TOKEN` | NPM token for private packages (optional) |

## Common Patterns

### ConfigMap-only Updates

When only the env file changes (no code changes), the workflow updates only the ConfigMap without rebuilding or redeploying:

```
Push with config changes only -> check-changes detects -> update-configmap-only job runs
```

### Helm Deployments

```yaml
jobs:
  stage:
    uses: zopsmart/workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: 'go build -o main ./cmd/...'
      DEPLOY_METHOD: helm
      HELM_VALUES_PATH: ./helm/values-stage.yaml
    secrets: inherit
```

### Multi-environment Setup

```yaml
# STAGE_CLUSTER_CREDENTIALS and PROD_CLUSTER_CREDENTIALS are
# resolved automatically per workflow — just set both secrets.

jobs:
  stage:
    uses: zopsmart/workflows/.github/workflows/stage-deploy.yaml@main
    secrets: inherit  # Uses STAGE_CLUSTER_CREDENTIALS

  prod:
    uses: zopsmart/workflows/.github/workflows/prod-deploy.yaml@main
    secrets: inherit  # Uses PROD_CLUSTER_CREDENTIALS
```

### React/Frontend Apps

```yaml
jobs:
  stage:
    uses: zopsmart/workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-frontend
      BUILD_COMMAND: 'yarn build'
      REACT_APP: true  # Generates browser-compatible ConfigMap
    secrets: inherit
```

## Composite Actions

Standalone actions for use in custom workflows:

| Action | Description |
|--------|-------------|
| [`docker-build-push`](.github/actions/docker-build-push/) | Build and push Docker image to any registry |
| [`registry-login`](.github/actions/registry-login/) | Authenticate to container registry (GAR/GCR/ECR/ACR/DockerHub/GHCR) |
| [`cluster-auth`](.github/actions/cluster-auth/) | Authenticate to Kubernetes cluster (GKE/EKS/AKS/kubeconfig) |
| [`resolve-image-path`](.github/actions/resolve-image-path/) | Resolve registry URL and construct image path |
| [`generate-configmap`](.github/actions/generate-configmap/) | Generate ConfigMap YAML from env file |
| [`apply-configmap`](.github/actions/apply-configmap/) | Download and apply ConfigMap from artifact |
| [`validate-tag`](.github/actions/validate-tag/) | Validate semver tag is latest and ahead |
| [`setup-go`](.github/actions/setup-go/) | Set up Go environment with caching |
| [`setup-node`](.github/actions/setup-node/) | Set up Node environment with caching |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for repo structure, architecture, and how to extend.
