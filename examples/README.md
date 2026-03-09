# Deployment Workflow Examples

## Quick Start

Copy an example matching your infrastructure and customize:

| Example | Registry | Cluster |
|---------|----------|---------|
| [gar-gke.yaml](gar-gke.yaml) | Google Artifact Registry | GKE |
| [ecr-eks.yaml](ecr-eks.yaml) | Amazon ECR | EKS |
| [acr-aks.yaml](acr-aks.yaml) | Azure Container Registry | AKS |
| [ghcr-kubeconfig.yaml](ghcr-kubeconfig.yaml) | GitHub Container Registry | Any (kubeconfig) |
| [dockerhub.yaml](dockerhub.yaml) | Docker Hub | Any (kubeconfig) |

## Language Support

Language is auto-detected from `BUILD_COMMAND`:

| Language | BUILD_COMMAND |
|----------|---------------|
| Go | `go build -o main ./cmd/...` |
| Node | `yarn build` or `npm run build` |

For React/frontend apps, add `REACT_APP: true` to generate browser-compatible ConfigMaps.

## Configuration

See the [root README](../README.md) for full configuration reference.
