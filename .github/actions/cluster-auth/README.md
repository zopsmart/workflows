# Cluster Auth

Authenticates to Kubernetes clusters with auto-detection of auth method.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | Yes | - | Environment name (stage, prod) |
| `credentials` | Yes | - | Cluster credentials (GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `cluster_name` | Conditional | - | Cluster name (required for GKE/EKS/AKS) |
| `cluster_region` | Conditional | - | Cluster region (required for GKE/EKS) |
| `cluster_project` | Conditional | - | GCP project (required for GKE) |
| `azure_resource_group` | Conditional | - | Azure resource group (required for AKS) |
| `aws_access_key_id` | No | - | AWS access key ID override |
| `aws_region` | No | - | AWS region override |
| `azure_client_id` | No | - | Azure client ID override |
| `azure_tenant_id` | No | - | Azure tenant ID override |

## Outputs

| Output | Description |
|--------|-------------|
| `auth_method` | Detected auth method: `gke`, `eks`, `aks`, `kubeconfig` |
| `kubeconfig_path` | Path to kubeconfig file |

## Auth Method Auto-Detection

The action automatically detects the auth method from credential format:

| Credential Format | Detected Method |
|-------------------|-----------------|
| GCP Service Account JSON (`type: service_account`) | `gke` |
| AWS JSON (`aws_secret_access_key` present) | `eks` |
| Azure JSON (`clientSecret` present) | `aks` |
| Base64-encoded kubeconfig | `kubeconfig` |

If detection fails, it infers from inputs:
- `cluster_project` set → GKE
- `aws_access_key_id` set → EKS
- `azure_client_id` set → AKS
- Default → kubeconfig

## Usage

### GKE (Google Kubernetes Engine)

```yaml
- uses: zopsmart/workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-cluster
    cluster_region: us-central1
    cluster_project: my-gcp-project
```

**Credentials format:**
```json
{
  "type": "service_account",
  "project_id": "my-project",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "client_email": "sa@my-project.iam.gserviceaccount.com"
}
```

### EKS (Amazon Elastic Kubernetes Service)

```yaml
- uses: zopsmart/workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-eks-cluster
    cluster_region: us-east-1
```

**Credentials format:**
```json
{
  "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE",
  "aws_secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "aws_region": "us-east-1"
}
```

### AKS (Azure Kubernetes Service)

```yaml
- uses: zopsmart/workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-aks-cluster
    azure_resource_group: my-resource-group
```

**Credentials format:**
```json
{
  "clientId": "12345678-1234-1234-1234-123456789abc",
  "clientSecret": "abc123~secret",
  "tenantId": "87654321-4321-4321-4321-cba987654321"
}
```

### Kubeconfig (Any Cluster)

```yaml
- uses: zopsmart/workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
```

**Credentials format:** Base64-encoded kubeconfig
```bash
# Generate with:
cat ~/.kube/config | base64 -w 0
```

## Split Credentials

You can provide some values via inputs and others via JSON:

```yaml
- uses: zopsmart/workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.AWS_SECRET_ACCESS_KEY }}  # Just the secret
    cluster_name: my-eks-cluster
    cluster_region: us-east-1
    aws_access_key_id: AKIAIOSFODNN7EXAMPLE  # Provided separately
```

## Credential Resolution Order

For EKS and AKS, values are resolved in this order:

1. Input parameter (highest priority)
2. JSON credential field
3. Default/fallback

Example for `aws_region`:
1. `aws_region` input
2. `aws_region` field in credentials JSON
3. `cluster_region` input (fallback)
