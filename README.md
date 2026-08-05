# gh-extensions-actions

Reusable GitHub Actions and workflows for building and publishing Podman Desktop extension OCI images.

## Actions

| Action | Description |
|---|---|
| [`oci-build`](#oci-build) | Build an extension OCI image using Buildah |
| [`oci-publish`](#oci-publish) | Publish an extension OCI image to a container registry |
| [`hermeto-prefetch`](#hermeto-prefetch) | Pre-fetch dependencies for hermetic builds using [Hermeto](https://hermetoproject.github.io/hermeto/latest/) |

## `oci-build`

Checks out the repository, builds an OCI image with Buildah (`--squash`), and uploads the resulting `oci-image.tar` as a GitHub Actions artifact.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `ref` | no | — | The branch, tag or SHA to checkout |
| `containerfile` | no | `./Containerfile` | Path to the Containerfile |
| `skip-checkout` | no | `false` | Skip the checkout step (use when source is already checked out and modified, e.g. by `hermeto-prefetch`) |
| `extra-args` | no | — | Additional newline-separated arguments for `buildah-build` (e.g. volume mounts, `--network none`) |

### Outputs

| Output | Description |
|---|---|
| `artifact-id` | Unique identifier for the uploaded artifact |
| `artifact-name` | Name of the uploaded artifact (`artifact`) |

### Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: podman-desktop/gh-extensions-actions/.github/actions/oci-build@main
        id: build
        with:
          ref: ${{ github.sha }}
```

## `oci-publish`

Downloads a previously built OCI artifact, loads it with Podman, tags it, pushes it to a container registry, and generates an artifact attestation.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `run-id` | no | — | Workflow run ID where the artifact was uploaded |
| `artifact-id` | no | — | ID of the artifact to publish |
| `artifact-name` | no | `artifact` | Name of the artifact to publish |
| `github-token` | no | — | GitHub token for downloading artifacts |
| `registry` | no | `ghcr.io` | Container registry to publish to |
| `registry-username` | yes | — | Registry username |
| `registry-password` | yes | — | Registry password |
| `image-name` | no | `${{ github.repository }}` | Image name |
| `tags` | yes | — | Whitespace-separated list of tags |

### Usage

```yaml
jobs:
  publish:
    runs-on: ubuntu-24.04
    steps:
      - uses: podman-desktop/gh-extensions-actions/.github/actions/oci-publish@main
        with:
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ github.token }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          tags: latest ${{ github.sha }}
```

## `hermeto-prefetch`

Pre-fetches project dependencies for hermetic (network-isolated) builds using [Hermeto](https://hermetoproject.github.io/hermeto/latest/). Runs the three-step Hermeto workflow (`fetch-deps`, `generate-env`, `inject-files`) and produces pre-formatted `extra-args` for the `oci-build` action.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `packages` | yes | — | JSON package config for `hermeto fetch-deps` (single object or array, e.g. `[{"type":"pnpm","path":"."}]`) |
| `source` | no | `.` | Path to the source directory |
| `output-dir` | no | `./hermeto-output` | Host path for Hermeto output |
| `container-output-dir` | no | `/cachi2/output` | Mount target inside the build container (defaults to `/cachi2/output` for Containerfile compatibility with Konflux/Cachi2) |
| `hermeto-image` | no | `ghcr.io/hermetoproject/hermeto:latest` | Hermeto container image |

### Outputs

| Output | Description |
|---|---|
| `output-dir` | Absolute path to the Hermeto output directory |
| `env-file` | Absolute path to the generated environment file |
| `sbom` | Absolute path to the generated SBOM (`bom.json`) |
| `extra-args` | Pre-formatted extra-args for `oci-build` (volume mounts + `--network none`) |

### Usage

See [Hermetic Builds](#hermetic-builds) for a complete example.

## Reusable Workflows

### `publish-oci-pr`

Publishes an OCI image built from a pull request. Downloads the artifact from a previous workflow run, pushes it to the registry with the commit SHA as the tag, and creates a GitHub check run with the image reference.

#### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `run-id` | yes | — | Workflow run ID where the artifact was uploaded |
| `head-sha` | yes | — | Commit SHA to tag the image with |
| `artifact-name` | no | `artifact` | Name of the artifact to publish |
| `image-name` | no | — | Image name (defaults to repository name) |
| `registry` | no | `ghcr.io` | Container registry |
| `registry-username` | no | `${{ github.repository_owner }}` | Registry username |
| `image-name-suffix` | no | `pr` | Suffix appended to the image name |

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `registry-password` | yes | Registry password |

#### Usage

```yaml
name: Build & Publish PR

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: podman-desktop/gh-extensions-actions/.github/actions/oci-build@main
        id: build
        with:
          ref: ${{ github.event.pull_request.head.sha }}

  publish:
    needs: build
    permissions:
      contents: read
      id-token: write
      attestations: write
      actions: read
      packages: write
      checks: write
    uses: podman-desktop/gh-extensions-actions/.github/workflows/publish-oci-pr.yml@main
    with:
      run-id: ${{ github.run_id }}
      head-sha: ${{ github.event.pull_request.head.sha }}
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

## Hermetic Builds

Hermetic builds run with network isolation (`--network none`), ensuring all dependencies are pre-fetched and verified before the build starts. This prevents supply-chain attacks, guarantees reproducibility, and produces accurate SBOMs.

The `hermeto-prefetch` action handles the pre-fetching using [Hermeto](https://hermetoproject.github.io/hermeto/latest/). It modifies `.npmrc` and the lockfile in-place so that `pnpm install --frozen-lockfile` resolves dependencies from the mounted volume instead of the network. This allows a **single Containerfile** to work for both hermetic and non-hermetic builds.

### Complete hermetic build workflow

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.sha }}

      - uses: podman-desktop/gh-extensions-actions/.github/actions/hermeto-prefetch@main
        id: hermeto
        with:
          packages: '[{"type":"generic","path":"."},{"type":"pnpm","path":"."}]'

      - uses: podman-desktop/gh-extensions-actions/.github/actions/oci-build@main
        id: build
        with:
          skip-checkout: true
          extra-args: ${{ steps.hermeto.outputs.extra-args }}

      # Optional: upload the SBOM
      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: ${{ steps.hermeto.outputs.sbom }}
```

Checkout is done explicitly before `hermeto-prefetch` because `inject-files` modifies source files in-place. The `oci-build` step skips its own checkout to preserve those modifications.

### Single Containerfile pattern

The same Containerfile works for non-hermetic builds (dependencies fetched from the network), hermetic builds via Hermeto (GitHub Actions), and hermetic builds via Cachi2 (Konflux). The `container-output-dir` defaults to `/cachi2/output` for path compatibility with Konflux pipelines.

```dockerfile
FROM node:24-slim AS builder

# Install pnpm: use prefetched tarball if available, otherwise corepack
RUN if ls /cachi2/output/deps/generic/pnpm-*.tgz 1>/dev/null 2>&1; then \
      npm install -g /cachi2/output/deps/generic/pnpm-*.tgz; \
    else \
      corepack enable; \
    fi

WORKDIR /app
COPY . .

# Source hermeto env if present (required for pnpm >= 11.3), then install
RUN if [ -f /tmp/hermeto.env ]; then . /tmp/hermeto.env; fi && \
    pnpm install --frozen-lockfile

RUN pnpm build

FROM scratch
COPY --from=builder /app/packages/backend/dist/ /extension/dist
COPY --from=builder /app/packages/backend/package.json /extension/
COPY --from=builder /app/LICENSE /extension/
```

### Prerequisites for hermetic mode

The consumer repository must include an `artifacts.lock.yaml` for the pnpm binary (fetched via Hermeto's generic backend):

```yaml
metadata:
  version: "1.0"
artifacts:
  - download_url: https://registry.npmjs.org/pnpm/-/pnpm-10.12.1.tgz
    checksum: "sha256:<checksum>"
    filename: pnpm-10.12.1.tgz
```

The `packages` input must include both the generic fetcher (for this tarball) and the pnpm fetcher (for project dependencies):

```json
[{"type": "generic", "path": "."}, {"type": "pnpm", "path": "."}]
```

For more details, see the [Hermeto documentation](https://hermetoproject.github.io/hermeto/latest/) and the [pnpm backend reference](https://hermetoproject.github.io/hermeto/latest/pnpm/).

## License

Apache-2.0
