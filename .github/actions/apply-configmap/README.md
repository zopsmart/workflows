# Apply ConfigMap

Downloads a ConfigMap artifact and applies it to a Kubernetes cluster.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `artifact_name` | Yes | - | Name of artifact containing ConfigMap YAML |
| `namespace` | Yes | - | Kubernetes namespace |

## Outputs

None

## Usage

```yaml
- uses: zopsmart/workflows/.github/actions/apply-configmap@main
  with:
    artifact_name: configmap-my-service
    namespace: production
```

## Prerequisites

- kubectl must be authenticated to the cluster
- Artifact must exist from a previous job in the same workflow run

## Behavior

1. Downloads the artifact to current directory
2. Finds the first `.yaml` file
3. Applies with `kubectl apply --force`

## Example Workflow

```yaml
jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: zopsmart/workflows/.github/actions/generate-configmap@main
        id: generate
        with:
          svc_name: my-service
          env_file_path: ./configs/stage.env

      - uses: actions/upload-artifact@v4
        with:
          name: configmap-my-service
          path: ${{ steps.generate.outputs.configmap_file }}

  apply:
    needs: generate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Authenticate to cluster first
      - uses: zopsmart/workflows/.github/actions/cluster-auth@main
        with:
          environment: stage
          credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
          cluster_name: my-cluster
          cluster_region: us-central1
          cluster_project: my-project

      # Then apply the configmap
      - uses: zopsmart/workflows/.github/actions/apply-configmap@main
        with:
          artifact_name: configmap-my-service
          namespace: my-app-stage
```

## Notes

- The action uses `--force` flag to handle ConfigMap updates
- Only the first `.yaml` file in the artifact is applied
- Ensure the namespace exists before applying
