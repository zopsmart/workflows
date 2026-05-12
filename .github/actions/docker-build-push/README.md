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
| `cache_scope` | No | `svc_name` | Scope key for the BuildKit GHA cache backend. Defaults to `svc_name` so each service has its own cache. Pass an explicit value to share a cache (e.g. multi-arch builds of the same service). Set to empty string to disable caching. |

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

This action combines four steps:

1. **Resolve image path** - Uses `resolve-image-path` action to construct the full image URL
2. **Login to registry** - Uses `registry-login` action to authenticate
3. **Set up Buildx** - Configures BuildKit so the GHA cache backend is available
4. **Build and push** - Uses `docker/build-push-action` to build and push the image, wired to `type=gha` `cache-from`/`cache-to` scoped by service

## Layer caching

The action exports BuildKit layer cache to the GitHub Actions cache backend
(`type=gha`) and restores from it on the next run. By default the cache is
scoped per service (`cache_scope` defaults to `svc_name`), so api/worker/
orchestrator builds in the same repo do not evict each other.

On a typical code-only change this saves the base-image `apk add`/`apt-get`
layers, `go mod download`, and lets the build's `--mount=type=cache`
mounts (e.g. `/root/.cache/go-build`, `/go/pkg/mod`) survive across runs —
turning multi-minute cold builds into well under a minute.

Things to know:

- The GHA cache is per branch with fallback to the default branch, per
  GitHub's standard cache scoping rules. The first run on a new branch
  warms from the default branch's cache.
- Total cache is capped at 10 GB per repo; least-recently-used entries
  are evicted automatically.
- Pass `cache_scope: ''` to disable caching for a specific build (e.g.
  if you intentionally want a from-scratch image).
- Pass a custom `cache_scope` to share cache across multiple builds of
  the same content (e.g. `cache_scope: my-service-multiarch`).

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
