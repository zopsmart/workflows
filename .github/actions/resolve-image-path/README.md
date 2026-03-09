# Resolve Image Path

Resolves container image registry URL and builds the full image path.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry_type` | Yes | - | Registry provider: `gar`, `gcr`, `ecr`, `acr`, `dockerhub`, `ghcr`, `custom` |
| `image_registry` | Conditional | - | Registry URL. Required for `gar`/`ecr`/`acr`/`custom` |
| `registry_project` | No | - | Project/account/namespace within registry |
| `registry_repo` | No | - | Repository name within project |
| `svc_name` | Yes | - | Service name |
| `image_tag` | No | - | Image tag (omit for base path only) |

## Outputs

| Output | Description |
|--------|-------------|
| `image_registry` | Resolved registry URL |
| `image_base` | Image path without tag |
| `image` | Full image path with tag (empty if no tag) |

## Image Path Construction

The image path is constructed based on which components are provided:

```
IMAGE_REGISTRY / REGISTRY_PROJECT / REGISTRY_REPO / SVC_NAME : TAG
```

| Components Provided | Result |
|---------------------|--------|
| All | `us-docker.pkg.dev/my-project/docker/api:v1` |
| Registry + Project | `gcr.io/my-project/api:v1` |
| Registry only | `docker.io/api:v1` |

## Registry Defaults

| Registry Type | Default URL |
|---------------|-------------|
| `gcr` | `gcr.io` |
| `ghcr` | `ghcr.io` |
| `dockerhub` | `docker.io` |

## Usage

### Full Path (GAR)

```yaml
- uses: zopsmart/workflows/.github/actions/resolve-image-path@main
  id: image
  with:
    registry_type: gar
    image_registry: us-docker.pkg.dev
    registry_project: my-gcp-project
    registry_repo: docker-registry
    svc_name: api
    image_tag: v1.0.0

# Outputs:
# image_registry: us-docker.pkg.dev
# image_base: us-docker.pkg.dev/my-gcp-project/docker-registry/api
# image: us-docker.pkg.dev/my-gcp-project/docker-registry/api:v1.0.0
```

### Without Repository (GCR)

```yaml
- uses: zopsmart/workflows/.github/actions/resolve-image-path@main
  id: image
  with:
    registry_type: gcr
    registry_project: my-gcp-project
    svc_name: api
    image_tag: latest

# Outputs:
# image_registry: gcr.io
# image_base: gcr.io/my-gcp-project/api
# image: gcr.io/my-gcp-project/api:latest
```

### Base Path Only (No Tag)

```yaml
- uses: zopsmart/workflows/.github/actions/resolve-image-path@main
  id: image
  with:
    registry_type: ecr
    image_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    svc_name: api

# Outputs:
# image_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
# image_base: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api
# image: (empty)
```

## Examples by Registry

| Registry | image_registry | registry_project | registry_repo | Result |
|----------|----------------|------------------|---------------|--------|
| GAR | `us-docker.pkg.dev` | `my-project` | `docker` | `us-docker.pkg.dev/my-project/docker/api:v1` |
| GCR | _(auto)_ | `my-project` | | `gcr.io/my-project/api:v1` |
| ECR | `123456.dkr.ecr.us-east-1.amazonaws.com` | | | `123456.dkr.ecr.us-east-1.amazonaws.com/api:v1` |
| ACR | `myregistry.azurecr.io` | | | `myregistry.azurecr.io/api:v1` |
| Docker Hub | _(auto)_ | `username` | | `docker.io/username/api:v1` |
| GHCR | _(auto)_ | `owner` | | `ghcr.io/owner/api:v1` |
