# Registry Login

Authenticates to container registries with auto-detection of credential format.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry_type` | Yes | - | Registry provider: `gar`, `gcr`, `ecr`, `acr`, `dockerhub`, `ghcr`, `custom` |
| `image_registry` | Conditional | - | Registry URL. Required for `gar`/`ecr`/`acr`/`custom`. Auto-defaults for others |
| `credentials` | Conditional | - | Registry auth (format varies by provider) |
| `username` | Conditional | - | Username for `dockerhub`/`ghcr`/`custom` |
| `aws_access_key_id` | Conditional | - | AWS access key ID for ECR |
| `aws_region` | No | - | AWS region for ECR (auto-extracted from URL) |
| `azure_client_id` | Conditional | - | Azure client ID for ACR |
| `azure_tenant_id` | Conditional | - | Azure tenant ID for ACR |
| `github_actor` | No | - | Fallback username for `dockerhub`/`ghcr`/`custom` |
| `github_token` | No | - | Fallback credentials for GHCR |

## Outputs

| Output | Description |
|--------|-------------|
| `registry` | Resolved registry URL |

## Registry Defaults

| Registry Type | Default URL |
|---------------|-------------|
| `gcr` | `gcr.io` |
| `ghcr` | `ghcr.io` |
| `dockerhub` | `docker.io` |
| `gar` | _(required)_ |
| `ecr` | _(required)_ |
| `acr` | _(required)_ |
| `custom` | _(required)_ |

## Usage

### Google Artifact Registry (GAR)

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: gar
    image_registry: us-docker.pkg.dev
    credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

**Credentials:** GCP Service Account JSON with Artifact Registry Writer role

### Google Container Registry (GCR)

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: gcr
    credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

**Credentials:** GCP Service Account JSON

### Amazon ECR

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: ecr
    image_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

**Credentials:**
```json
{
  "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE",
  "aws_secret_access_key": "wJalrXUtnFEMI/..."
}
```

Region is auto-extracted from the registry URL.

### Azure Container Registry (ACR)

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: acr
    image_registry: myregistry.azurecr.io
    credentials: ${{ secrets.REGISTRY_CREDENTIALS }}
```

**Credentials:**
```json
{
  "clientId": "12345678-1234-...",
  "clientSecret": "abc123~secret",
  "tenantId": "87654321-4321-..."
}
```

### Docker Hub

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: dockerhub
    credentials: ${{ secrets.DOCKERHUB_TOKEN }}
    username: myusername
```

**Credentials:** Docker Hub access token

### GitHub Container Registry (GHCR)

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: ghcr
    credentials: ${{ secrets.GITHUB_TOKEN }}
    github_actor: ${{ github.actor }}
```

**Credentials:** GitHub PAT with `packages:write` scope (or `GITHUB_TOKEN`)

### Custom Registry

```yaml
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: custom
    image_registry: registry.example.com
    credentials: ${{ secrets.REGISTRY_PASSWORD }}
    username: myuser
```

## Split Credentials

For ECR and ACR, you can provide credentials as JSON or split across inputs:

```yaml
# ECR with split credentials
- uses: zopsmart/workflows/.github/actions/registry-login@main
  with:
    registry_type: ecr
    image_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    credentials: ${{ secrets.AWS_SECRET_ACCESS_KEY }}  # Just the secret
    aws_access_key_id: ${{ vars.AWS_ACCESS_KEY_ID }}   # Key ID from vars
```
