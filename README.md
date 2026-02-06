# Reusable GitHub Actions Workflows

Centralised CI/CD workflows shared across [National Tutoring Observatory](https://github.com/National-Tutoring-Observatory) repositories.

This repository extracts common deployment jobs into **reusable workflows** invoked via `workflow_call`, keeping deployment logic consistent, reviewable, and testable in staging before production.

---

## Workflows

| Workflow | File | Purpose |
|----------|------|---------|
| Resolve Release | [`resolve_release.yml`](.github/workflows/resolve_release.yml) | Validates semver tags and ensures tagged commits originate from the base branch |
| ECS Preflight | [`ecs_preflight.yml`](.github/workflows/ecs_preflight.yml) | Waits for an ECS service to reach a stable state before deploying |
| ECS Build Image | [`ecs_build_image.yml`](.github/workflows/ecs_build_image.yml) | Builds and pushes a container image to Amazon ECR |
| ECS Deploy Service | [`ecs_deploy_service.yml`](.github/workflows/ecs_deploy_service.yml) | Updates an ECS service with a new task definition and image |
| Deployment Summary | [`deployment_summary.yml`](.github/workflows/deployment_summary.yml) | Generates a formatted deployment report in the workflow summary |

---

## Usage

Calling repositories reference these workflows with `uses:`:

```yaml
jobs:
  preflight:
    uses: National-Tutoring-Observatory/workflows/.github/workflows/ecs_preflight.yml@main
    with:
      environment: staging
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

All workflows are environment-agnostic. Environment-specific values are resolved through GitHub environment variables (`vars.*`) configured in the calling repository.

---

## Workflow reference

### `resolve_release.yml`

Validates that the triggering tag matches `vX.Y.Z` and that the tagged commit exists on the base branch. Outputs the version string without the leading `v`.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `base-branch` | No | `main` | Branch the tagged commit must belong to |

#### Outputs

| Name | Description |
|------|-------------|
| `version` | Release version without leading `v` (e.g. `1.2.3`) |

#### Permissions

`contents: read`

---

### `ecs_preflight.yml`

Polls an ECS service until it has a single deployment with a `COMPLETED` rollout state. Prevents deployments when in-progress or failed rollouts are active.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `environment` | **Yes** | — | GitHub environment name (e.g. `staging`, `production`) |
| `aws-region` | No | `us-east-1` | AWS region |
| `max-wait` | No | `600` | Maximum wait time in seconds |
| `sleep-interval` | No | `30` | Seconds between polls |

#### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `AWS_ROLE_ARN` | **Yes** | IAM role ARN assumed via OIDC |

#### Environment variables (caller)

| Variable | Description |
|----------|-------------|
| `ECS_CLUSTER` | ECS cluster name |
| `ECS_SERVICE` | ECS service name |

#### Permissions

`id-token: write`, `contents: read`

---

### `ecs_build_image.yml`

Authenticates to ECR, builds a Docker image, and pushes it. Supports custom Dockerfiles, image tag suffixes (for multi-service builds), and custom tags.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `environment` | **Yes** | — | GitHub environment name |
| `aws-region` | No | `us-east-1` | AWS region |
| `dockerfile` | No | `Dockerfile` | Path to the Dockerfile |
| `image-suffix` | No | `""` | Suffix appended to the image tag (e.g. `-worker`) |
| `output-name` | **Yes** | — | Name of the output image variable |
| `tag` | No | Short commit SHA | Custom image tag |

#### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `AWS_ROLE_ARN` | **Yes** | IAM role ARN assumed via OIDC |

#### Outputs

| Name | Description |
|------|-------------|
| `image` | Fully qualified ECR image URI |

#### Environment variables (caller)

| Variable | Description |
|----------|-------------|
| `ECR_REPOSITORY` | ECR repository name |

#### Permissions

`id-token: write`, `contents: read`

---

### `ecs_deploy_service.yml`

Fetches the current task definition, swaps the container image, registers a new revision, and updates the ECS service. Automatically rolls back to the previous task definition on failure.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `environment` | **Yes** | — | GitHub environment name |
| `aws-region` | No | `us-east-1` | AWS region |
| `image` | **Yes** | — | Full container image URI |
| `service` | **Yes** | — | Service type: `web` or `worker` |

#### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `AWS_ROLE_ARN` | **Yes** | IAM role ARN assumed via OIDC |

#### Environment variables (caller)

| Variable | Used by |
|----------|---------|
| `ECS_CLUSTER` | `web` |
| `ECS_SERVICE` | `web` |
| `ECS_CONTAINER_NAME` | `web` |
| `ECS_WORKER_CLUSTER` | `worker` |
| `ECS_WORKER_SERVICE` | `worker` |
| `ECS_WORKER_CONTAINER_NAME` | `worker` |

#### Permissions

`id-token: write`, `contents: read`

---

### `deployment_summary.yml`

Queries the GitHub Actions API for job timings and writes a Markdown summary to `$GITHUB_STEP_SUMMARY`. Runs on both success and failure so deployment reports are always visible.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `environment` | **Yes** | — | Environment name |
| `web-image` | **Yes** | — | Deployed web container image URI |
| `worker-image` | **Yes** | — | Deployed worker container image URI |
| `app-url` | **Yes** | — | Application URL |

#### Permissions

`actions: read`

---

## Repository guidelines

**Belongs here** — reusable, environment-agnostic workflows parameterised via inputs and secrets.

**Does not belong here** — application code, environment-specific workflows (`staging.yml`, `production.yml`), secrets, or repository-specific logic. Those stay in calling repositories.

---

## License

[MIT](LICENSE)
