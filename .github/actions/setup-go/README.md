# Setup Go Build

Sets up Go environment, downloads dependencies, and builds the application.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `go_version` | No | `1.20` | Go version |
| `build_command` | Yes | - | Build command to run |
| `docker_file_path` | No | `.` | Dockerfile directory (build runs here) |
| `artifact_path` | No | `main` | Built artifact path (relative to docker_file_path) |
| `extra_dependencies` | No | `false` | Enable extra dependencies step |
| `dependencies_command` | No | - | Commands to install extra dependencies |
| `pat` | No | - | GitHub PAT for private repos |

## Usage

### Basic Build

```yaml
- uses: actions/checkout@v4

- uses: zopsmart/workflows/.github/actions/setup-go@main
  with:
    build_command: 'go build -o main ./cmd/...'
```

### With Private Dependencies

```yaml
- uses: zopsmart/workflows/.github/actions/setup-go@main
  with:
    build_command: 'go build -o main ./cmd/...'
    pat: ${{ secrets.PAT }}
```

The PAT configures Git to authenticate when fetching private Go modules:
```
go get github.com/myorg/private-module
```

### Custom Go Version

```yaml
- uses: zopsmart/workflows/.github/actions/setup-go@main
  with:
    go_version: '1.21'
    build_command: 'go build -o main ./cmd/...'
```

### Extra Dependencies

For builds requiring additional system packages or tools:

```yaml
- uses: zopsmart/workflows/.github/actions/setup-go@main
  with:
    build_command: 'go build -o main ./cmd/...'
    extra_dependencies: true
    dependencies_command: |
      apt-get update && apt-get install -y libvips-dev
```

### Custom Dockerfile Location

When Dockerfile is in a subdirectory:

```yaml
- uses: zopsmart/workflows/.github/actions/setup-go@main
  with:
    build_command: 'go build -o main ./cmd/...'
    docker_file_path: './services/api'
    artifact_path: 'main'
```

The action:
1. Changes to `docker_file_path` directory
2. Runs the build command
3. Moves the artifact back to the root directory

## What It Does

1. **Setup Go** - Installs specified Go version
2. **Configure Git** - Sets up PAT for private repos (if provided)
3. **Download dependencies** - Runs `go mod download`
4. **Extra dependencies** - Runs custom commands (if enabled)
5. **Build** - Runs your build command

## Build Output

The built binary is placed in the repository root, ready for Docker build:

```dockerfile
# Dockerfile
FROM alpine:latest
COPY main /app/main
CMD ["/app/main"]
```
