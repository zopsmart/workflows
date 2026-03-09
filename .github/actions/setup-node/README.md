# Setup Node Build

Sets up Node environment, installs dependencies, and builds the application.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `node_version` | No | `18` | Node version |
| `build_command` | Yes | - | Build command to run |
| `dependencies_flag` | No | - | Extra yarn install flags |
| `npm_token` | No | - | NPM token for private packages |

## Usage

### Basic Build

```yaml
- uses: actions/checkout@v4

- uses: zopsmart/workflows/.github/actions/setup-node@main
  with:
    build_command: 'yarn build'
```

### With Private Packages

```yaml
- uses: zopsmart/workflows/.github/actions/setup-node@main
  with:
    build_command: 'yarn build'
    npm_token: ${{ secrets.NPM_TOKEN }}
```

The token configures npm registry for `@zopsmart` scoped packages:
```
yarn add @zopsmart/private-package
```

### Custom Node Version

```yaml
- uses: zopsmart/workflows/.github/actions/setup-node@main
  with:
    node_version: '20'
    build_command: 'yarn build'
```

### With Yarn Install Flags

```yaml
- uses: zopsmart/workflows/.github/actions/setup-node@main
  with:
    build_command: 'yarn build'
    dependencies_flag: '--frozen-lockfile --ignore-scripts'
```

### npm Instead of yarn

```yaml
- uses: zopsmart/workflows/.github/actions/setup-node@main
  with:
    build_command: 'npm run build'
```

Note: Dependencies are always installed with `yarn install`. If you need npm for dependencies, use a custom workflow.

## What It Does

1. **Setup Node** - Installs specified Node version
2. **Configure npm** - Sets up registry for `@zopsmart` scoped packages
3. **Install dependencies** - Runs `yarn install` (with optional flags)
4. **Build** - Runs your build command

## Private Package Registry

The action configures npm to use GitHub Packages for `@zopsmart` scoped packages:

```
@zopsmart:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=<NPM_TOKEN>
```

## Build Output

Build output location depends on your build configuration. Typically:

- **React/Vue/Angular**: `build/` or `dist/`
- **Next.js**: `.next/`

Example Dockerfile:
```dockerfile
FROM nginx:alpine
COPY build/ /usr/share/nginx/html/
```
