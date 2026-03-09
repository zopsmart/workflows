# Contributing

Guide for maintaining and extending the deployment workflows.

## Repo Structure

```
.github/
  workflows/
    stage-deploy.yaml          # Entry point: stage deployment
    prod-deploy.yaml           # Entry point: prod deployment
    _build.yaml                # Internal: build app + push Docker image
    _retag-image.yaml          # Internal: retag existing image
    _deploy.yaml               # Internal: deploy to Kubernetes
    _update-configmap.yaml     # Internal: update ConfigMap only
    _check-changes.yaml        # Internal: detect code vs config changes
    _detect-language.yaml      # Internal: auto-detect language
    _resolve-registry-config.yaml  # Internal: resolve registry settings
    _resolve-deploy-config.yaml    # Internal: resolve cluster settings
  actions/
    docker-build-push/         # Build + push image (any registry)
    registry-login/            # Registry auth (GAR/GCR/ECR/ACR/DockerHub/GHCR)
    cluster-auth/              # Cluster auth (GKE/EKS/AKS/kubeconfig)
    resolve-image-path/        # Construct image path from components
    generate-configmap/        # Generate ConfigMap YAML from env file
    apply-configmap/           # Apply ConfigMap from artifact
    validate-tag/              # Validate semver tag
    setup-go/                  # Go environment setup
    setup-node/                # Node environment setup
examples/                      # Cloud-specific caller workflow examples
```

## Naming Conventions

- **`_` prefix** = internal building block (not called directly by consumers)
- **No prefix** = public entry point workflow (called by consumer repos)
- Each composite action has its own directory with `action.yml` + `README.md`

## Architecture

### How Workflows Compose

Consumer repos call **entry points** (`stage-deploy.yaml`, `prod-deploy.yaml`), which orchestrate **building blocks** (prefixed with `_`), which use **composite actions** for atomic operations.

```
Consumer repo
  └─► Entry point (stage-deploy / prod-deploy)
        ├─► _resolve-registry-config  →  resolve-image-path action
        ├─► _resolve-deploy-config
        ├─► _check-changes            →  generate-configmap action
        ├─► _build                    →  setup-go/setup-node, docker-build-push, registry-login
        ├─► _retag-image              →  registry-login
        ├─► _deploy                   →  cluster-auth, apply-configmap
        └─► _update-configmap         →  cluster-auth, apply-configmap
```

### Stage Deploy Flow

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            STAGE DEPLOY FLOW                                      │
│                                                                                   │
│  ┌──────────────────┐                                                            │
│  │ resolve-config   │  Detect language, resolve registry/cluster vars            │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │ _check-changes   │  Detect code vs env changes, generate ConfigMap artifact   │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ├─── code changed? ───┐                                                │
│           │                     ▼                                                │
│           │          ┌──────────────────┐                                        │
│           │          │   BUILD PHASE    │                                        │
│           │          │  ┌────────────┐  │                                        │
│           │          │  │  _build    │  │  LANGUAGE=go: go mod download → build │
│           │          │  │            │  │  LANGUAGE=node: yarn install → build  │
│           │          │  │            │  │  LANGUAGE=generic: build only         │
│           │          │  └─────┬──────┘  │                                        │
│           │          │        │         │                                        │
│           │          │        ▼         │                                        │
│           │          │  ┌────────────┐  │                                        │
│           │          │  │docker-build│  │  Multi-registry: GAR/GCR/ECR/ACR/     │
│           │          │  │   -push    │  │  DockerHub/GHCR/Custom                │
│           │          │  └─────┬──────┘  │                                        │
│           │          └────────┼─────────┘                                        │
│           │                   │                                                   │
│           │                   ▼                                                   │
│           │          ┌──────────────────┐                                        │
│           │          │    _deploy       │  Deploy + apply ConfigMap artifact     │
│           │          └──────────────────┘                                        │
│           │                                                                       │
│           │ (env changed only)                                                    │
│           │          ┌──────────────────┐                                        │
│           └─────────►│_update-configmap │  Apply ConfigMap artifact only         │
│                      └──────────────────┘                                        │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Prod Deploy Flow

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PROD DEPLOY FLOW                                      │
│                                                                                   │
│  ┌──────────────────┐                                                            │
│  │ resolve-config   │  Extract release version, resolve vars                     │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │  validate-tag    │  Validate semver tag is latest and ahead                   │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ├───────────────────┬───────────────────┐                              │
│           │                   │                   │                              │
│           ▼                   ▼                   │                              │
│  ┌──────────────────┐  ┌──────────────────┐      │                              │
│  │  _retag-image    │  │ _check-changes   │      │                              │
│  │ SHA → v1.0.0     │  │ Generate ConfigMap│      │                              │
│  └────────┬─────────┘  │    artifact       │      │                              │
│           │            └────────┬──────────┘      │                              │
│           │                     │                 │                              │
│           ▼                     ▼                 │                              │
│  ┌──────────────────────────────────────────┐    │                              │
│  │              _deploy                      │◄───┘                              │
│  │  Deploy to prod + apply ConfigMap artifact│                                   │
│  └──────────────────────────────────────────┘                                   │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## How to Extend

### Adding a New Registry

1. Add detection pattern to `.github/actions/resolve-image-path/action.yml`
2. Add login logic to `.github/actions/registry-login/action.yml`
3. Add URL pattern to registry type auto-detection in `_resolve-registry-config.yaml`
4. Add an example to `examples/`
5. Update the supported platforms table in `README.md`

### Adding a New Cluster Provider

1. Add auth logic to `.github/actions/cluster-auth/action.yml`
2. Add detection pattern to the auth method auto-detection
3. Add credential resolution to `_resolve-deploy-config.yaml`
4. Add an example to `examples/`
5. Update the supported platforms table in `README.md`

### Adding a New Language

1. Add detection pattern to `_detect-language.yaml`
2. Create a `setup-<lang>` composite action in `.github/actions/`
3. Add build logic to `_build.yaml`
4. Update language detection docs in `README.md`

## PR Guidelines

- Test with at least one cloud provider before submitting
- Internal workflows (`_` prefix) should not be called directly by consumers
- Keep composite actions self-contained with their own `README.md`
- Update examples if inputs/behavior changes
- Use `vars.*` and `secrets.*` for configuration — avoid hardcoded values
