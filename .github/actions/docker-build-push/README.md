# Docker Build and Push

Logs in to a container registry and builds/pushes a Docker image.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry_type` | Yes | - | Registry provider: `gar`, `gcr`, `ecr`, `acr`, `dockerhub`, `ghcr`, `custom` |
| `image_registry` | Conditional | - | Registry URL. Required for `gar`/`ecr`/`acr`/`custom` |
| `registry_project` | No | - | Project/account/namespace within registry |
| `registry_repo` | No | - | Repository name within project |
| `svc_name` | Yes | - | Service name |
| `image_tag` | Yes | - | Image tag |
| `dockerfile_path` | No | `.` | Path to Dockerfile directory |
| `build_arguments` | No | - | Docker build arguments |
| `registry_credentials` | Yes | - | Registry auth credentials |
| `registry_username` | Conditional | - | Username for `dockerhub`/`ghcr`/`custom` |
| `aws_access_key_id` | Conditional | - | AWS access key ID for ECR |
| `aws_region` | No | - | AWS region for ECR |
| `azure_client_id` | Conditional | - | Azure client ID for ACR |
| `azure_tenant_id` | Conditional | - | Azure tenant ID for ACR |
| `github_actor` | No | - | Fallback username for `dockerhub`/`ghcr`/`custom` |
| `github_token` | No | - | Fallback credentials for GHCR |

## Outputs

| Output | Description |
|--------|-------------|
| `image` | Full image path with tag |

## Usage

### Google Artifact Registry (GAR)

```yaml
- uses: zopsmart/workflows/.github/actions/docker-build-push@main
  with:
    registry_type: gar
    image_registry: us-docker.pkg.dev
    registry_project: my-gcp-project
    registry_repo: docker-registry
    svc_name: my-service
    image_tag: ${{ github.sha }}
    registry_credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

### Amazon ECR

```yaml
- uses: zopsmart/workflows/.github/actions/docker-build-push@main
  with:
    registry_type: ecr
    image_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    svc_name: my-service
    image_tag: ${{ github.sha }}
    registry_credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

### GitHub Container Registry (GHCR)

```yaml
- uses: zopsmart/workflows/.github/actions/docker-build-push@main
  with:
    registry_type: ghcr
    registry_project: ${{ github.repository_owner }}
    svc_name: my-service
    image_tag: ${{ github.sha }}
    registry_credentials: ${{ secrets.GITHUB_TOKEN }}
    github_actor: ${{ github.actor }}
```

### With Build Arguments

```yaml
- uses: zopsmart/workflows/.github/actions/docker-build-push@main
  with:
    registry_type: gar
    image_registry: us-docker.pkg.dev
    registry_project: my-gcp-project
    registry_repo: docker-registry
    svc_name: my-service
    image_tag: ${{ github.sha }}
    dockerfile_path: ./docker
    build_arguments: |
      BUILD_VERSION=${{ github.sha }}
      NODE_ENV=production
    registry_credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

## How It Works

This action combines three steps:

1. **Resolve image path** - Uses `resolve-image-path` action to construct the full image URL
2. **Login to registry** - Uses `registry-login` action to authenticate
3. **Build and push** - Uses `docker/build-push-action` to build and push the image

## Dockerfile Location

The action looks for `Dockerfile` in the `dockerfile_path` directory:

```yaml
dockerfile_path: .           # ./Dockerfile
dockerfile_path: ./docker    # ./docker/Dockerfile
dockerfile_path: ./services/api  # ./services/api/Dockerfile
```

## Output Usage

```yaml
- uses: zopsmart/workflows/.github/actions/docker-build-push@main
  id: build
  with:
    # ... inputs ...

- name: Deploy
  run: |
    echo "Deploying image: ${{ steps.build.outputs.image }}"
    # Output: us-docker.pkg.dev/my-project/docker/my-service:abc123
```
