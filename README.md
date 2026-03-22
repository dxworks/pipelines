# dxworks/pipelines

Reusable GitHub Actions workflows for the DXWorks organisation.

Each repo picks only the workflows it needs — no repo is forced to use all of them.

## Available workflows

### `trivy-fs-scan.yml`
Scans source code dependencies for known vulnerabilities and uploads results to the GitHub Security tab.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `severity` | string | `MEDIUM,HIGH,CRITICAL` | Severity levels to report |
| `fail-on-findings` | boolean | `true` | Fail the workflow if vulnerabilities are found |

```yaml
jobs:
  trivy-fs:
    uses: dxworks/pipelines/.github/workflows/trivy-fs-scan.yml@v1
    permissions:
      contents: read
      security-events: write
```

### `trivy-image-scan.yml`
Scans a Docker image for OS and application vulnerabilities. Optionally posts a PR comment with a summary table and link to the Security tab.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `image-ref` | string | *required* | Docker image to scan (e.g. `dxworks/insider:latest`) |
| `severity` | string | `MEDIUM,HIGH,CRITICAL` | Severity levels to report |
| `fail-on-findings` | boolean | `true` | Fail the workflow if vulnerabilities are found |
| `post-pr-comment` | boolean | `false` | Post a sticky PR comment with results |

```yaml
jobs:
  trivy-image:
    uses: dxworks/pipelines/.github/workflows/trivy-image-scan.yml@v1
    with:
      image-ref: dxworks/my-app:latest
      post-pr-comment: true
    permissions:
      contents: read
      security-events: write
      pull-requests: write
```

### `trivy-daily-scan.yml`
Scheduled daily scan for dependencies and (optionally) a Docker image. Results go to the GitHub Security tab — you get notified when new CVEs appear.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `image-ref` | string | `''` | Docker image to scan (empty to skip image scan) |
| `severity` | string | `MEDIUM,HIGH,CRITICAL` | Severity levels to report |

```yaml
name: Daily Security Scan

on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  daily-scan:
    uses: dxworks/pipelines/.github/workflows/trivy-daily-scan.yml@v1
    with:
      image-ref: dxworks/my-app:latest
    permissions:
      contents: read
      security-events: write
```

## Versioning

- `@v1` — pinned to a major version tag. Non-breaking improvements are included automatically when the tag is moved forward.
- `@main` — always the latest. Convenient but may break if a breaking change is pushed.

**Recommendation:** Use `@v1`. Breaking changes will go to `@v2`.

## Requirements

- The consuming repo must have **GitHub Advanced Security** enabled (free for public repos) for the Security tab integration.
- This repo must remain **public** for cross-repo reusable workflow calls on GitHub Free plans.
