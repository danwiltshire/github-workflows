# Reusable workflows

| Workflow                                      | Description                                                      |
| --------------------------------------------- | ---------------------------------------------------------------- |
| [Build image](#build-image)                   | Builds, scans, and publishes Docker images to Amazon ECR.        |
| [Update GitOps values](#update-gitops-values) | Updates application environment values in the GitOps repository. |

## Build image

Builds and scans a Docker image for vulnerabilities, and publishes it to Amazon ECR for pull requests and pushes to `main`.

Trivy reports high and critical vulnerabilities without failing the build for findings; scan errors still fail the job. The full report is available in the job log, and pull requests receive a Markdown table showing up to 20 findings.

The comment is updated on subsequent scans, including scans of other images in the same pull request, so only one image's report is shown at a time.

Callers must grant `pull-requests: write` and `issues: read` to post and update comments. Fork pull requests generally receive a read-only token and cannot post comments unless repository settings allow write tokens; they also need access to the AWS credentials required for publishing.

### Inputs

| Input            | Type   | Required | Default                   | Description                               |
| ---------------- | ------ | -------- | ------------------------- | ----------------------------------------- |
| `architecture`   | string | No       | `amd64`                   | Runner architecture: `amd64` or `arm64`.  |
| `context`        | string | No       | `.`                       | Docker build context.                     |
| `file`           | string | No       | `Dockerfile`              | Path to the Dockerfile.                   |
| `build-args`     | string | No       | Empty                     | Newline-separated Docker build arguments. |
| `tag-prefix`     | string | No       | Empty                     | Prefix for all image tags, for example `ui`; a hyphen is added automatically. |
| `repository-uri` | string | No       | `vars.ECR_REPOSITORY_URI` | ECR image repository URI.                 |

### Outputs

| Output    | Description                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------- |
| `sha-tag` | Image tag containing the seven-character commit SHA and optional prefix; for pull requests, this is the head commit SHA. |

### Usage

```yaml
name: Build

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write
  issues: read
  pull-requests: write

jobs:
  build:
    uses: danwiltshire/github-workflows/.github/workflows/build-image.yml@main
    with:
      architecture: arm64
      tag-prefix: ui
      build-args: |
        VERSION=1.2.3
    secrets: inherit

  show-tag:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - run: echo "$IMAGE_TAG"
        env:
          IMAGE_TAG: ${{ needs.build.outputs.sha-tag }}
```

For example, `tag-prefix: ui` produces `ui-abcdef1` and, on pushes to `main`, `ui-latest`. Use a different prefix such as `api` for another image in the same repository.

## Update GitOps values

Merges YAML values into an application's environment file in `danwiltshire/helm-charts` and commits the change to its default branch.

### Inputs

| Input         | Type   | Required | Default | Description                                                                                 |
| ------------- | ------ | -------- | ------- | ------------------------------------------------------------------------------------------- |
| `application` | string | Yes      | —       | Application directory under `apps/`, for example `k8s-books`.                               |
| `environment` | string | Yes      | —       | Environment filename under `apps/<application>/envs/`, without `.yaml`, for example `prod`. |
| `values`      | string | Yes      | —       | Non-empty YAML mapping to merge into the file; quote values that must be strings.           |

### Outputs

| Output | Description                            |
| ------ | -------------------------------------- |
| None   | This workflow does not define outputs. |

### Usage

```yaml
name: Build and promote

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write
  issues: read
  pull-requests: write

jobs:
  build:
    uses: danwiltshire/github-workflows/.github/workflows/build-image.yml@main
    with:
      architecture: arm64
    secrets: inherit

  promote-to-prod:
    uses: danwiltshire/github-workflows/.github/workflows/update-gitops-values.yml@main
    if: github.event_name == 'push'
    needs: build
    secrets: inherit
    with:
      application: ${{ github.event.repository.name }}
      environment: prod
      values: |
        imageTag: "${{ needs.build.outputs.sha-tag }}"
```
