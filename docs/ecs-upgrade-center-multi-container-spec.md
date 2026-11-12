# ECS Upgrade Center Multi-Container Support Specification

## 1. Purpose

This document explains how Bold Upgrade Center supports Bold BI multi-container upgrades on Amazon ECS Fargate. It is intended for engineering, DevOps, QA, and support teams who need to understand:

- How Upgrade Center works end to end.
- How ECS support is achieved for a Bold BI multi-container deployment.
- What configuration, AWS resources, IAM permissions, and validation are required.
- What happens during upgrade, failure, recovery, and rollback.

The operational deployment guide is [Upgrade Center on ECS](ecs-upgrade-guide.md). This specification describes the design behind that guide.

## 2. Current problem

Bold BI ECS multi-container deployments run the product as separate ECS services, usually one service per product component:

| Upgrade Center component key | Bold BI service area | Required |
| --- | --- | --- |
| `id-web-deployment` | Identity Provider web | Yes |
| `id-api-deployment` | Identity Provider API | Yes |
| `id-ums-deployment` | User Management Server | Yes |
| `bi-web-deployment` | BI web application | Yes |
| `bi-api-deployment` | BI API | Yes |
| `bi-jobs-deployment` | BI jobs | Yes |
| `bi-dataservice-deployment` | BI designer/data service | Yes |
| `bold-etl-deployment` | ETL service | No |
| `bold-ai-deployment` | AI service | No |
| `bold-mcp-deployment` | MCP service | No |

A normal ECS deployment upgrade is not only an image tag replacement. Upgrade Center must also:

1. Identify the current installed version.
2. Show available target versions.
3. Analyze database schema changes for the selected release.
4. Back up impacted database tables before product services are changed.
5. Run pre-upgrade validation.
6. Change the ECS task definitions and ECS services safely.
7. Run post-upgrade validation.
8. Roll back the database and ECS services if the upgrade fails.
9. Persist enough state to recover if Upgrade Center restarts during an operation.

## 3. Solution summary

Upgrade Center runs as a separate ECS service in the same AWS environment as Bold BI. It does not become part of the Bold BI product task definitions. It orchestrates the upgrade from outside the product services.

At a high level, ECS support is achieved by adding an ECS execution backend to the existing Upgrade Center workflow:

```text
Administrator browser
        |
        v
Application Load Balancer path rule
        |
        v
Upgrade Center ECS service
        |
        +-- Reads Bold BI configuration from Standalone storage mode
        +-- Reads current version and release metadata
        +-- Reads master and tenant database information
        +-- Backs up impacted schema tables
        +-- Starts one-off ECS Playwright validation tasks
        +-- Registers new ECS task-definition revisions with target images
        +-- Updates Bold BI ECS services to the new revisions
        +-- Persists rollback context and operation history
```

The same user-facing Upgrade Center screens are used for Kubernetes, Docker, and ECS. The selected execution environment decides which backend updates the product containers.

## 4. Main design decisions

### 4.1 Upgrade Center is a separate service

Upgrade Center is deployed as its own Fargate service. This keeps orchestration logic, IAM permissions, validation reports, and recovery state separate from Bold BI runtime containers.

Benefits:

- Product containers do not need AWS SDK upgrade logic.
- Upgrade Center can be stopped or updated independently.
- ECS task roles can grant Upgrade Center only the permissions needed for upgrade orchestration.
- Rollback and history state can be persisted outside the in-memory process.

### 4.2 Standalone integration mode is used for ECS

For ECS, Upgrade Center uses `Standalone` integration mode with `BoldBI` deployment mode. In this mode Upgrade Center reads the Bold BI runtime configuration from the same storage backend used by the product deployment.

Supported storage layouts:

| Layout | Product configuration storage | Playwright shared state storage |
| --- | --- | --- |
| EFS | EFS | EFS |
| S3 | S3 | S3 pre-signed URL handoff |
| EFS+S3 | S3 for product configuration | EFS for validation state |

### 4.3 ECS service mappings are explicit

Upgrade Center does not discover arbitrary ECS services by name pattern. Each product component key must be mapped to an approved ECS service name and container name.

Example mapping shape:

```text
UpgradeCenter__Ecs__Services__bi-web-deployment__Service   = <bi-web-ecs-service-name>
UpgradeCenter__Ecs__Services__bi-web-deployment__Container = <bi-web-container-name>
```

Why this is required:

- Prevents accidental updates to unrelated services.
- Allows customers to use their own ECS service names.
- Ensures the exact container inside each task definition is updated.
- Allows optional services such as ETL, AI, and MCP to be omitted when not installed.

### 4.4 Upgrade Center creates new task-definition revisions

ECS service task definitions are immutable. Upgrade Center therefore does not edit existing task definitions. It:

1. Describes the current task definition of each mapped ECS service.
2. Copies the task definition.
3. Changes only the mapped container image.
4. Registers a new task-definition revision.
5. Updates the ECS service to use that new revision.

All other task-definition settings are preserved, including CPU, memory, environment variables, secrets, log configuration, volumes, mount points, IAM roles, Service Connect settings, and target group settings.

### 4.5 Rollback context is saved before services are changed

Before stopping or updating product services, Upgrade Center persists rollback context containing:

- ECS cluster name.
- Component key.
- ECS service name.
- Container name.
- Previous task-definition ARN.
- Target task-definition ARN.
- Previous desired count.
- Autoscaling suspension state when applicable.
- Previous product image snapshot.
- Database backup mapping when schema backup was required.

This makes rollback possible even if the process fails after the new task definitions are registered.

### 4.6 A maintenance-window confirmation is required

ECS upgrades intentionally stop the selected product services before starting the new revisions. Upgrade Center does not automatically configure customer-facing ALB maintenance rules or block writes. Operators must confirm that customer writes are blocked before the upgrade starts.

This is enforced by the `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` setting.

## 5. Upgrade Center functional flow

### 5.1 Dashboard and release selection

1. Administrator opens Upgrade Center through the ALB path rule.
2. Upgrade Center authenticates the administrator.
3. It reads runtime configuration from Standalone storage.
4. It obtains the current Bold BI version from the product version API, with configuration fallback for non-Docker execution modes.
5. It calls the Bold BI release metadata API to list available versions.
6. The dashboard shows current version, available releases, active operation status, history, and rollback availability.

### 5.2 Confirmation and impact analysis

When an administrator selects a target release:

1. Upgrade Center discovers the master database from product configuration.
2. It decrypts the master database connection string using the Bold BI key material.
3. It reads `boldtc_tenantinfo` to discover tenant databases for the selected product.
4. It calls the release metadata API for database scripts between the current and target versions.
5. It extracts impacted tables, created tables, and dropped tables.
6. The confirmation page shows schema-impact details and warnings.

### 5.3 Standard release upgrade workflow

After confirmation, Upgrade Center creates a job and runs these stages:

| Stage | What happens |
| --- | --- |
| Preflight | Validates ECS settings, service mappings, image metadata, allowed registries, runner task definition, and maintenance-window confirmation. |
| Database backup | Discovers master and tenant databases, analyzes schema impact, and backs up affected existing tables. |
| Pre-upgrade validation | Starts a one-off Playwright Fargate task and records the validation result. |
| ECS image upgrade | Registers target task-definition revisions and updates ECS services. |
| Post-upgrade validation | Starts another one-off Playwright Fargate task against the upgraded product. |
| Cleanup | Removes temporary validation tasks, temporary task definitions, temporary Secrets Manager entries, and shared state. |
| Completion | Updates operation history and keeps the rollback point available. |

### 5.4 Custom patch upgrade workflow

Custom patch upgrades are intended for support-provided image tags. The flow is similar to standard releases, with these differences:

- The target image mapping is built from the custom patch input.
- If only a tag is provided, Upgrade Center keeps the current repository and image name and replaces the tag.
- If a repository is provided, Upgrade Center keeps the current image name and combines it with the provided repository and tag.
- Image registry and repository allow-lists still apply.
- Database schema backup is skipped because custom patch image upgrades are not release schema migrations.

## 6. ECS image upgrade algorithm

The ECS backend follows this sequence:

1. Validate the configured ECS cluster.
2. Validate every required component mapping.
3. Read each mapped ECS service.
4. Confirm each service is active, uses the ECS deployment controller, has a single active deployment, and is running.
5. Read each service's task definition.
6. Locate the configured container in the task definition.
7. Validate the target image against the allowed registry and repository settings.
8. Copy the task definition and replace only the mapped container image.
9. Register the copied task definition as a new revision.
10. Read current desired count and autoscaling state.
11. Persist rollback context before service changes.
12. Suspend service autoscaling when needed.
13. Stop all selected old services by setting desired count to zero.
14. Start new revisions in rollout groups.
15. Wait for ECS to report the expected task definition, running count, no pending tasks, one deployment, and completed rollout state.
16. Restore autoscaling.
17. Mark the stage complete.

### 6.1 Rollout grouping

ECS service rollout uses two groups:

1. Identity services (`id-*`).
2. Remaining Bold BI services.

Services inside a group can start in parallel. Groups run sequentially so the shared Identity services are available before the BI services are validated.

## 7. Playwright validation design

Upgrade Center uses one-off ECS Fargate tasks for validation instead of running browser tests inside the Upgrade Center container.

### 7.1 Runner task definition

The runner task definition is registered separately and referenced by Upgrade Center. It must use:

- Fargate compatibility.
- `awsvpc` networking.
- AWS logs.
- A known runner container name, usually `playwright`.
- No task-level entrypoint override for the base runner template.

### 7.2 Runtime secret handoff

For each validation stage, Upgrade Center creates a short-lived Secrets Manager secret containing runtime environment values such as:

- Upload URLs for report and result files.
- Upload token.
- Admin credentials injected from the Upgrade Center runtime.
- Product base URL and validation mode.
- Shared-state access values.

The validation task receives these values through ECS container secrets. The temporary secret is deleted during cleanup.

### 7.3 Validation state options

| State mode | How it works |
| --- | --- |
| S3 | Upgrade Center creates pre-signed URLs for the runner to get and put validation state. This avoids requiring an EFS mount for validation state. |
| EFS | Upgrade Center and runner share an EFS directory mounted at the configured shared-state path. |

### 7.4 Result handling

The runner writes a structured validation summary to logs and uploads reports/results back to Upgrade Center. Upgrade Center reads CloudWatch logs, validates the summary, saves the report, and checks the configured pass-rate threshold.

## 8. Rollback design

Rollback can run automatically after a failed upgrade or manually from the History tab when a rollback point is available.

Rollback sequence:

1. Stop affected ECS services.
2. Restore database backups when the standard release upgrade created schema backups.
3. Update each affected ECS service to the recorded previous task-definition revision.
4. Wait for ECS service health.
5. Restore autoscaling.
6. Mark rollback complete or failed.

If database restore fails, Upgrade Center leaves services stopped and keeps the backup information for manual recovery. This avoids starting product services against a partially restored schema.

## 9. Restart and recovery behavior

Upgrade Center persists operational state in the Bold BI master database. If Upgrade Center restarts during a job, it can recover only stages that have enough durable context.

Recoverable stages include:

- ECS image upgrade after rollback context was persisted.
- Post-upgrade validation.
- Cleanup.
- Automatic rollback.

Recovery safety rules:

- Upgrade Center verifies the persisted ECS cluster and service mapping still match current configuration.
- It verifies that the service task definition is either the previous revision or the target revision recorded by Upgrade Center.
- If an external deployment changed a service to another revision, automatic changes are blocked.
- Operators must not switch execution environments during an active operation.

## 10. Required AWS resources

| Resource | Purpose |
| --- | --- |
| ECS cluster | Runs Bold BI services, Upgrade Center service, and validation tasks. |
| Upgrade Center ECS service | Hosts the Upgrade Center web application. |
| Playwright ECS task definition | Template for one-off validation tasks. |
| Application Load Balancer rule | Routes Upgrade Center paths to the Upgrade Center service. |
| CloudWatch log groups | Stores Upgrade Center and Playwright validation logs. |
| Secrets Manager admin secret | Supplies administrator credentials to Upgrade Center/runner without plain-text task environment values. |
| Temporary Secrets Manager secrets | Short-lived validation runtime handoff created by Upgrade Center. |
| S3 and/or EFS | Product configuration and validation shared state, depending on deployment layout. |
| Database | Stores Bold BI product data, Upgrade Center history, operational state, and backups. |

## 11. Required IAM permissions

Upgrade Center task role needs permissions for:

- Describing ECS services, task definitions, and tasks.
- Registering and deregistering task definitions.
- Updating ECS services.
- Running and stopping validation tasks.
- Passing only approved task roles to ECS tasks.
- Reading CloudWatch logs for validation tasks.
- Creating, reading, tagging, and deleting temporary Secrets Manager secrets under the Upgrade Center prefix.
- Reading the configured admin secret when used by the task definition.
- Accessing S3 or EFS for product configuration and validation state as required by the selected storage layout.
- Reading and restoring the Bold BI database through the configured product database user.

The Playwright execution role needs standard ECS task execution permissions. The Playwright task role needs only the storage permissions required by the selected validation-state mode.

## 12. Required Upgrade Center configuration

Minimum ECS-specific settings:

| Setting | Purpose |
| --- | --- |
| `UpgradeCenter__ExecutionEnvironment=ECS` | Selects the ECS backend. |
| `UpgradeCenter__IntegrationMode=Standalone` | Uses Standalone configuration reading for ECS. |
| `UpgradeCenter__DeploymentMode=BoldBI` | Enables Bold BI product definitions. |
| `UpgradeCenter__Ecs__Region` | AWS region for ECS/API clients. |
| `UpgradeCenter__Ecs__Cluster` | ECS cluster to manage. |
| `UpgradeCenter__Ecs__RunnerTaskDefinition` | Playwright runner task definition family/revision. |
| `UpgradeCenter__Ecs__RunnerContainer` | Runner container name inside the runner task definition. |
| `UpgradeCenter__Ecs__Subnets__*` | Subnets used by validation tasks. |
| `UpgradeCenter__Ecs__SecurityGroups__*` | Security groups used by validation tasks. |
| `UpgradeCenter__Ecs__UploadBaseUrl` | External Upgrade Center base URL used by runner upload callbacks. |
| `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` | Operator confirmation that customer writes are blocked. |
| `UpgradeCenter__Ecs__Services__*` | ECS service/container mapping for every installed component. |
| `UpgradeCenter__Kubernetes__AllowedImageRegistryHosts__*` | Registry allow-list reused by the image validator. |
| `UpgradeCenter__Kubernetes__AllowedImageRepositories__*` | Repository allow-list reused by the image validator. |

The `Kubernetes` configuration prefix is still used for image registry allow-list settings because the image validator contract is shared across execution backends.

## 13. Templates provided in this repository

The ECS multi-container deployment contains Upgrade Center templates under:

```text
deploy/ECS-deployment-manifest/Multi container/UpgradeCenter/
```

The folders are organized by storage layout:

| Folder | Use when |
| --- | --- |
| `EFS/` | Bold BI product configuration and validation state both use EFS. |
| `S3/` | Bold BI product configuration and validation state both use S3. |
| `EFS+S3/` | Product configuration uses S3 and validation state uses EFS. |

Each folder contains:

| Template | Purpose |
| --- | --- |
| `task-definition-upgrade-center.json` | ECS task definition for the Upgrade Center service. |
| `task-definition-playwright.json` | ECS task definition template for one-off Playwright validation tasks. |

## 14. What needs to be done to enable ECS multi-container support

### 14.1 Product deployment requirements

- Deploy Bold BI multi-container services on ECS Fargate.
- Keep each Bold BI component as an ECS service with a stable service name.
- Use a supported shared configuration storage layout: EFS, S3, or EFS+S3.
- Ensure product services are healthy before enabling Upgrade Center.
- Ensure the product database user can perform backup and restore operations required by Upgrade Center.

### 14.2 Upgrade Center deployment tasks

- Build and publish the Upgrade Center image with ECS support enabled.
- Register the Playwright runner task definition.
- Register the Upgrade Center task definition.
- Create or update IAM roles and policies.
- Create CloudWatch log groups.
- Create or reuse the required S3 bucket and/or EFS access point.
- Create an ALB target group and path rule for Upgrade Center.
- Configure all component-to-service mappings.
- Configure image registry/repository allow-lists.
- Configure the administrator secret through Secrets Manager references.

### 14.3 Upgrade operation tasks

Before each upgrade:

- Take a full managed database snapshot.
- Block customer writes while keeping Upgrade Center reachable.
- Confirm the maintenance window setting.
- Verify ECS service mappings and optional service omissions.
- Verify runner networking to the ALB and AWS APIs.
- Verify target images exist and are accessible from Fargate.

After each upgrade:

- Review the post-upgrade validation report.
- Confirm product services are healthy.
- Disable the maintenance-window confirmation setting.
- Reopen customer traffic.
- Keep rollback records until the upgrade is accepted.

## 15. Safety and security controls

| Control | Reason |
| --- | --- |
| Explicit service mappings | Prevents accidental updates to unmanaged services. |
| Allowed component keys only | Prevents arbitrary ECS service targeting through configuration. |
| Registry and repository allow-lists | Prevents unapproved images from being deployed. |
| Rollback context persisted before service changes | Ensures rollback data exists before downtime begins. |
| External-deployment detection | Blocks automatic recovery if another actor changed ECS services. |
| Temporary Secrets Manager entries | Avoids long-lived validation credentials. |
| Upload token validation | Prevents unauthenticated report/result uploads. |
| Maintenance-window confirmation | Prevents accidental upgrades while customer writes are active. |
| Database backups before image rollout | Allows schema rollback when release migrations fail after service changes. |

## 16. Limitations and assumptions

- ECS support is intended for Bold BI Standalone integration mode.
- Upgrade Center supports one active upgrade or rollback at a time.
- ECS services must use the standard ECS rolling deployment controller.
- Blue/green CodeDeploy ECS deployments are not part of this design.
- Upgrade Center does not automatically add or remove ALB maintenance rules.
- Operators remain responsible for blocking customer writes during the maintenance window.
- Required components must be mapped exactly once.
- Optional components should be mapped only when installed.
- Custom patch upgrades skip schema backup and should be used only when the patch does not require release schema migrations.

## 17. Validation checklist

Use this checklist during QA or implementation review:

- Dashboard loads and shows the current Bold BI version.
- Available release list is populated from release metadata.
- Confirmation page shows database impact for standard releases.
- Required ECS mappings are validated and missing optional mappings are allowed.
- Standard release creates database backup when affected tables exist.
- Pre-upgrade Playwright validation starts as an ECS task and uploads results.
- Upgrade Center registers new task-definition revisions without changing unrelated task settings.
- ECS services stop old tasks, start target revisions, and reach healthy state.
- Post-upgrade Playwright validation succeeds and report is available.
- Cleanup removes temporary runner task definitions, temporary secrets, and shared state.
- Failed image rollout triggers automatic rollback.
- Failed post-upgrade validation triggers automatic rollback.
- Manual rollback restores previous task-definition revisions and database backup.
- Upgrade Center restart during image rollout can reconcile or safely block if external changes are detected.
- S3-only validation works without requiring EFS.

## 18. Expected outcome

After ECS multi-container support is configured, administrators can use Upgrade Center to upgrade Bold BI ECS deployments through the UI. Upgrade Center coordinates database backup, validation, ECS task-definition revision management, service rollout, report collection, history, and rollback from one place while preserving the customer's existing ECS service settings.
