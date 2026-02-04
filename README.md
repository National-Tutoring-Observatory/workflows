# GitHub Workflows

This repository contains **reusable GitHub Actions workflows** shared across National Tutoring Observatory repositories.

Its purpose is to centralise deployment and CI/CD logic so that:
- workflows are **consistent across environments**
- duplication is reduced
- changes can be tested safely in staging before being released to production

---

## Why this repository exists

Historically, deployment workflows lived inside individual application repositories.  
Over time this led to:

- duplicated logic across staging and production
- environment-specific drift
- harder reviews and riskier changes

This repository solves that by extracting common jobs into **reusable workflows** that can be called via `workflow_call`.

---

## Workflow overview

| Workflow | File | Description |
|--------|------|-------------|
| ECS preflight   | [`ecs_preflight.yml`](.github/workflows/ecs_preflight.yml) | Waits for an ECS service to reach a stable state before continuing a deployment. Ensures no in-progress or failed rollouts are active. |
| ECS Build Image | [`ecs_build_image.yml`](.github/workflows/ecs_build_image.yml) | Builds and pushes a container image to AWS ECR. |

> Additional workflows will be added incrementally as existing pipelines are refactored.

---

## What belongs here

This repository should contain **only reusable workflows**, for example:

- ECS preflight checks
- Container build workflows
- ECS deployment steps

Each workflow should be:
- environment-agnostic
- fully parameterised via inputs
- free of hard-coded repository details
- safe to reuse across staging and production

---

## What does *not* belong here

- Application code
- Environment-specific workflows (e.g. `staging.yml`, `production.yml`)
- Secrets
- Repository-specific logic

Those remain in the calling repositories.

---

## How these workflows are used

Other repositories call workflows from this repo using `uses:`.

Example:

```yaml
jobs:
  preflight:
    uses: National-Tutoring-Observatory/workflows/.github/workflows/ecs_preflight.yml@main
    with:
      environment: staging
      ecs-cluster: nto-staging
      ecs-service: nto-api-staging
    secrets:
      AWS_ROLE_ARN: ${{ vars.AWS_ROLE_ARN }}
```
