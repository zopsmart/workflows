# Validate Tag

Validates that a semantic version tag is the latest and ahead of previous tags.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `tag` | Yes | - | Tag to validate (e.g., `v1.2.3`) |

## Outputs

| Output | Description |
|--------|-------------|
| `valid` | `true` if tag passes validation |
| `previous_tag` | Previous tag used for comparison |
| `error` | Error message if validation failed |

## Usage

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required for tag history

- uses: zopsmart/workflows/.github/actions/validate-tag@main
  id: validate
  with:
    tag: ${{ github.ref_name }}

- name: Fail if invalid
  if: steps.validate.outputs.valid != 'true'
  run: |
    echo "::error::${{ steps.validate.outputs.error }}"
    exit 1
```

## Validation Rules

1. **Semantic version**: Tag must be semantically greater than the highest existing tag
2. **Commit ancestry**: Previous tag's commit must be an ancestor of current tag

## Examples

| Scenario | Result |
|----------|--------|
| `v1.0.1` after `v1.0.0` | Valid |
| `v1.0.0` when `v1.0.1` exists | Invalid - not semantically ahead |
| `v1.0.2` on commit behind `v1.0.1` | Invalid - commit not ahead |
| First tag (no previous) | Valid |

## Why These Rules?

Prevents common mistakes:
- Accidentally reusing old version numbers
- Tagging commits that aren't on the main branch
- Creating tags on stale branches

## Prerequisites

- Repository must be checked out with full history (`fetch-depth: 0`)
- Tags must follow `v*` pattern (e.g., `v1.0.0`, `v2.1.3`)

## Complete Workflow Example

```yaml
name: Production Deploy

on:
  push:
    tags:
      - 'v*'

jobs:
  validate:
    runs-on: ubuntu-latest
    outputs:
      valid: ${{ steps.validate.outputs.valid }}
      previous_tag: ${{ steps.validate.outputs.previous_tag }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: zopsmart/workflows/.github/actions/validate-tag@main
        id: validate
        with:
          tag: ${{ github.ref_name }}

      - name: Fail if invalid
        if: steps.validate.outputs.valid != 'true'
        run: |
          echo "::error::${{ steps.validate.outputs.error }}"
          exit 1

  deploy:
    needs: validate
    if: needs.validate.outputs.valid == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "Deploying ${{ github.ref_name }}..."
```

## Semantic Version Comparison

The action compares versions using `sort -V`, which handles semantic versioning correctly:

- `v1.2.10` > `v1.2.9` (numeric comparison, not string)
- `v2.0.0` > `v1.99.99`
- `v1.0.0-beta` < `v1.0.0`
