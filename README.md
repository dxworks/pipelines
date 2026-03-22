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
| `image-ref` | string | *required* | Docker image to scan. When `build-context` is set, used as the tag for the built image. |
| `build-context` | string | `''` | Path to Docker build context (e.g. `.`). When set, builds the image instead of pulling it. |
| `build-setup` | string | `''` | Shell commands to run before `docker build` (e.g. `gradle clean build`). |
| `java-version` | string | `''` | Java version to set up before build (empty to skip). |
| `node-version` | string | `''` | Node.js version to set up before build (empty to skip). |
| `severity` | string | `MEDIUM,HIGH,CRITICAL` | Severity levels to report |
| `fail-on-findings` | boolean | `true` | Fail the workflow if vulnerabilities are found |
| `post-pr-comment` | boolean | `false` | Post a sticky PR comment with results |

**Pull and scan a published image:**
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

**Build from Dockerfile and scan (e.g. on PRs):**
```yaml
jobs:
  trivy-image:
    uses: dxworks/pipelines/.github/workflows/trivy-image-scan.yml@v1
    with:
      image-ref: my-app:${{ github.sha }}
      build-context: .
      build-setup: gradle clean build
      java-version: '21'
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

---

## Release Workflows

### `release-gate.yml`
Quality gate that must pass before any artifacts are published. Runs build, tests, and Trivy security scan.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `java-version` | string | `''` | Java version to set up (empty to skip) |
| `node-version` | string | `''` | Node.js version to set up (empty to skip) |
| `build-script` | string | `./scripts/build.sh` | Path to build script |
| `trivy-severity` | string | `HIGH,CRITICAL` | Trivy severity threshold |

### `release-archive.yml`
Builds the project and creates a release archive (ZIP). Uses convention-based scripts from the calling repo.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | *required* | Semver version string |
| `java-version` | string | `''` | Java version (empty to skip) |
| `node-version` | string | `''` | Node.js version (empty to skip) |
| `build-script` | string | `./scripts/build.sh` | Path to build script |
| `prepare-script` | string | `./scripts/prepare-release.sh` | Path to release preparation script (receives version as $1) |
| `archive-name` | string | `release-archive` | Name for the uploaded artifact |

### `release-npm.yml`
Publishes an npm package to GitHub Packages and/or npmjs.org. Uses OIDC Trusted Publishers for npmjs.org — no secrets needed.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | *required* | Semver version string |
| `node-version` | string | `'20'` | Node.js version |
| `build-script` | string | `''` | Path to build script (empty to skip) |
| `publish-github-packages` | boolean | `true` | Publish to GitHub Packages |
| `publish-npmjs` | boolean | `true` | Publish to npmjs.org via OIDC |

Requires `id-token: write` permission for npm provenance.

### `release-docker.yml`
Builds and pushes a multi-arch Docker image (linux/amd64 + linux/arm64 by default).

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | *required* | Semver version string |
| `image-name` | string | *required* | Docker image name (e.g. `dxworks/insider`) |
| `platforms` | string | `linux/amd64,linux/arm64` | Target platforms |
| `build-script` | string | `''` | Build script to run before docker build |
| `push-dockerhub` | boolean | `true` | Push to Docker Hub |
| `push-ghcr` | boolean | `false` | Push to GitHub Container Registry |

Secrets: `dockerhub-username`, `dockerhub-token` (when pushing to Docker Hub).

### `release-github.yml`
Creates a GitHub Release with assets. Auto-generates release notes from PRs/commits if no notes file is provided. Uses `gh` CLI — no third-party actions.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | *required* | Semver version string |
| `tag` | string | *required* | Git tag name |
| `release-name` | string | `''` | Release name (empty to use tag) |
| `artifact-name` | string | `release-archive` | Artifact to download |
| `asset-name` | string | *required* | Name for the release asset file |
| `release-notes-file` | string | `''` | Path to notes file (empty to auto-generate) |
| `prerelease` | boolean | `false` | Mark as pre-release |
| `draft` | boolean | `false` | Create as draft |

Idempotent: re-running uploads the asset to the existing release.

### Full release example (Java + npm + Docker)

```yaml
on:
  push:
    tags: ['v*', '!**-voyager']

jobs:
  parse-tag:
    runs-on: ubuntu-latest
    outputs:
      semver: ${{ steps.parse.outputs.semver }}
    steps:
      - id: parse
        run: echo "semver=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

  gate:
    needs: [parse-tag]
    uses: dxworks/pipelines/.github/workflows/release-gate.yml@v1
    with:
      java-version: '21'

  archive:
    needs: [gate, parse-tag]
    uses: dxworks/pipelines/.github/workflows/release-archive.yml@v1
    with:
      version: ${{ needs.parse-tag.outputs.semver }}
      java-version: '21'

  npm:
    needs: [gate, parse-tag]
    uses: dxworks/pipelines/.github/workflows/release-npm.yml@v1
    with:
      version: ${{ needs.parse-tag.outputs.semver }}
    permissions:
      contents: read
      packages: write
      id-token: write

  docker:
    needs: [gate, parse-tag]
    uses: dxworks/pipelines/.github/workflows/release-docker.yml@v1
    with:
      version: ${{ needs.parse-tag.outputs.semver }}
      image-name: dxworks/my-app
      java-version: '21'
      build-script: ./scripts/build.sh
    secrets:
      dockerhub-username: ${{ secrets.DOCKERHUB_USERNAME }}
      dockerhub-token: ${{ secrets.DOCKERHUB_TOKEN }}

  github-release:
    needs: [parse-tag, archive, npm, docker]
    uses: dxworks/pipelines/.github/workflows/release-github.yml@v1
    with:
      version: ${{ needs.parse-tag.outputs.semver }}
      tag: ${{ github.ref_name }}
      release-name: 'My App ${{ needs.parse-tag.outputs.semver }}'
      asset-name: my-app.zip
    permissions:
      contents: write
```

---

## Versioning

- `@v1` — pinned to a major version tag. Non-breaking improvements are included automatically when the tag is moved forward.
- `@main` — always the latest. Convenient but may break if a breaking change is pushed.

**Recommendation:** Use `@v1`. Breaking changes will go to `@v2`.

## Requirements

- The consuming repo must have **GitHub Advanced Security** enabled (free for public repos) for the Security tab integration.
- This repo must remain **public** for cross-repo reusable workflow calls on GitHub Free plans.
