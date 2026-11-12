# Bold BI Upgrade Center on ECS

The customer-facing guide lives at [docs/ecs-upgrade-guide.md](../../../../docs/ecs-upgrade-guide.md), and the design/specification document lives at [docs/ecs-upgrade-center-multi-container-spec.md](../../../../docs/ecs-upgrade-center-multi-container-spec.md). This folder holds the task definition templates those documents reference.

Pick the subfolder that matches your Bold BI storage:

- `EFS/` for EFS-only deployments.
- `S3/` for S3-only deployments (recommended).
- `EFS+S3/` for hybrid deployments.

Each folder contains `task-definition-upgrade-center.json` and `task-definition-playwright.json`. Replace the placeholders as described in the guide, then register the runner task first and the Upgrade Center task second.
