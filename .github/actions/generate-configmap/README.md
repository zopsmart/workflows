# Generate ConfigMap

Generates a Kubernetes ConfigMap YAML file from an environment file.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `svc_name` | Yes | - | Service/ConfigMap name |
| `env_file_path` | Yes | - | Path to env file |
| `react_app` | No | `false` | Enable React-specific format |

## Outputs

| Output | Description |
|--------|-------------|
| `configmap_file` | Path to generated ConfigMap YAML |

## Usage

```yaml
- uses: zopsmart/workflows/.github/actions/generate-configmap@main
  with:
    svc_name: my-service
    env_file_path: ./configs/stage.env
```

## Env File Format

- Lines with `KEY=value` format
- Comments (lines starting with `#`) are skipped
- Empty lines are skipped

**Example env file:**
```
# Database configuration
DB_HOST=localhost
DB_PORT=5432

# API settings
API_URL=https://api.example.com
```

## Output Formats

**Standard** (`react_app: false`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-service
data:
  DB_HOST: localhost
  DB_PORT: 5432
  API_URL: https://api.example.com
```

**React** (`react_app: true`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-service
data:
  environment.js: |
    if (typeof window !== 'undefined')
    window.env = {
      DB_HOST: "localhost",
      DB_PORT: "5432",
      API_URL: "https://api.example.com",
    }
```

## React App Usage

For React/frontend applications, use `react_app: true` to generate a ConfigMap that can be mounted as a JavaScript file:

```yaml
- uses: zopsmart/workflows/.github/actions/generate-configmap@main
  with:
    svc_name: my-frontend
    env_file_path: ./configs/stage.env
    react_app: true
```

Mount the ConfigMap in your Kubernetes deployment:

```yaml
volumes:
  - name: config
    configMap:
      name: my-frontend
containers:
  - name: app
    volumeMounts:
      - name: config
        mountPath: /usr/share/nginx/html/environment.js
        subPath: environment.js
```

Access in your React app:
```javascript
const apiUrl = window.env?.API_URL || 'http://localhost:3000';
```
