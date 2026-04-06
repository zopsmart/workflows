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
- Test and lint for Go and Node projects

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
# .github/workflows/ci.yaml
name: CI/CD

on:
  push:
    branches: [development]
  pull_request:
  release:
    types: [published]

jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: go
    secrets: inherit

  stage:
    if: github.ref == 'refs/heads/development'
    needs: test
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

### test-and-lint.yaml

Runs tests and linting for Go or Node projects. Dispatches to the appropriate language-specific workflow based on `LANGUAGE`.

**What it does:**
1. Detects language (`go` or `node`) from the required `LANGUAGE` input
2. For Go: runs `golangci-lint`, executes tests with optional coverage threshold, spins up any required service containers, and optionally runs Postman integration tests
3. For Node: installs dependencies, runs ESLint and Prettier checks, and executes the test suite

#### Required Inputs

| Input | Description |
|-------|-------------|
| `LANGUAGE` | Build language: `go` or `node` |

#### Go-Specific Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `GO_VERSION` | `1.22` | Go version |
| `LINTER_VERSION` | `v1.54.2` | golangci-lint version |
| `LINTER_TIMEOUT` | `8m` | Linter timeout |
| `TESTCOVERAGE_THRESHOLD` | `0` | Minimum test coverage %. Fails if not met |
| `ADD_SCHEMA` | `false` | Run a DB schema setup command before tests |
| `SCHEMA_COMMAND` | | Command to load DB schema |
| `EXTRA_DEPENDENCIES` | `false` | Enable step to install extra dependencies |
| `DEPENDENCIES_COMMAND` | | Commands to install extra dependencies |
| `MODULES` | | Space-separated list of sub-module dirs to test separately |

#### Go Service Toggles

Spin up sidecar containers for integration tests:

| Input | Default | Description |
|-------|---------|-------------|
| `MYSQL_ENABLE` | `false` | Start a MySQL container |
| `POSTGRES_ENABLE` | `false` | Start a PostgreSQL container |
| `REDIS_ENABLE` | `false` | Start a Redis container |
| `ZIPKIN_ENABLE` | `false` | Start a Zipkin container |
| `ELASTIC_SEARCH_ENABLE` | `false` | Start an Elasticsearch container |
| `KAFKA_ENABLE` | `false` | Start a Kafka container |
| `MONGO_ENABLE` | `false` | Start a MongoDB container |
| `MSSQL_ENABLE` | `false` | Start an MSSQL container |
| `DYNAMODB_ENABLE` | `false` | Start a DynamoDB Local container |
| `CASSANDRA_ENABLE` | `false` | Start a Cassandra container |

#### Go Postman / Integration Test Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `POSTMAN_ENABLED` | `false` | Run Postman integration tests |
| `APP_NAME` | | Postman collection filename (without extension) |
| `SERVER_COMMAND` | | Commands to run before starting the server for Postman tests |

#### Node-Specific Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `NODE_VERSION` | `18` | Node version |
| `NODE_PACKAGE_MANAGER` | `yarn` | Package manager: `npm` or `yarn` |
| `DEPENDENCIES_FLAG` | | Extra flags for `npm install` / `yarn install` |
| `TEST_COMMAND` | | Command to run Node tests |
| `LINTER_CHECKS` | `true` | Run ESLint |
| `PRETTIER_CHECKS` | `true` | Run Prettier |
| `ENABLE_TESTS` | `true` | Run the test suite |
| `USE_GAR_PKG` | `false` | Fetch packages from Google Artifact Registry instead of GitHub Packages |

#### Secrets

| Secret | Description |
|--------|-------------|
| `PAT` | GitHub PAT for private Go/Node packages (optional) |
| `NPM_TOKEN` | NPM token for private npm packages (Node only, optional) |
| `REGISTRY_CREDENTIALS` | GCP service-account JSON for GAR package access (Node only, optional) |

---

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
| `USE_GAR_PKG` | `false` | Fetch packages from Google Artifact Registry instead of GitHub Packages |

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

### Test on Pull Request, Deploy on Merge

```yaml
name: CI/CD

on:
  pull_request:
  push:
    branches: [development]
  release:
    types: [published]

jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: go
    secrets: inherit

  stage:
    if: github.ref == 'refs/heads/development'
    needs: test
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

### Go with Services and Coverage Threshold

```yaml
jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: go
      TESTCOVERAGE_THRESHOLD: '80'
      POSTGRES_ENABLE: true
      REDIS_ENABLE: true
    secrets: inherit
```

### Node with Custom Test Command

```yaml
jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: node
      NODE_PACKAGE_MANAGER: npm
      TEST_COMMAND: 'npm run test:ci'
      PRETTIER_CHECKS: false
    secrets: inherit
```

### Go with Postman Integration Tests

```yaml
jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: go
      POSTMAN_ENABLED: true
      APP_NAME: my-service
      SERVER_COMMAND: './main &'
      POSTGRES_ENABLE: true
    secrets: inherit
```

### Go with Sub-modules

```yaml
jobs:
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: go
      MODULES: 'pkg/auth pkg/billing pkg/notifications'
    secrets: inherit
```

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
  test:
    uses: zopsmart/workflows/.github/workflows/test-and-lint.yaml@main
    with:
      LANGUAGE: node
      NODE_PACKAGE_MANAGER: yarn
      TEST_COMMAND: 'yarn test --watchAll=false'
    secrets: inherit

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