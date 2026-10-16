# Bold BI Upgrade Center on ECS

Use the existing Bold Upgrade Center application with `ExecutionEnvironment=ECS`, `IntegrationMode=Standalone`, and `DeploymentMode=BoldBI`. This initial ECS implementation does not enable Reports or Common. Kubernetes defaults and manifests are unchanged.

## Before Deployment

1. Finish the existing BI ECS installation described in [deployment.md](../deployment.md). Keep the existing VPC, database, ALB, product services, storage settings, and Secrets.
2. Build/publish Upgrade Center from the updated `bold-common-upgrade-center` source. Reuse the existing release API's `/version/json/{version}/k8s` records, product image tags, and version-specific `playwrightRunnerImage`. No ECS-specific release data, separate Playwright image build, or automation-repository edit is required.
3. Verify the existing runner images on Fargate. They must contain `/app/docker-entrypoint.js` and support the existing pre/post/cleanup and upload protocol. S3-only state transfer also needs Node.js 18 or later with global `fetch` and `AbortSignal.timeout`. Upgrade Center supplies the S3 bootstrap through the task startup command and calls the original entrypoint in the same image. Leave template `entryPoint` unset; EFS uses the original image entrypoint unchanged. Missing compatible image mappings still block the upgrade before backup.
4. Choose the storage folder below. Replace every `<placeholder>` in the two JSON task definitions. Enter the actual ECS service and container names for each component mapping. The `*-deployment` dictionary keys identify product components; they are not assumed to be ECS service names. Remove mappings for optional AI/MCP/ETL services that are not installed. Required BI and IDP mappings must remain.
5. Create private CloudWatch log groups with retention. Configure the task/execution roles below. Use the existing administrator Secret; map its actual JSON keys to `BOLD_ADMIN_USERNAME` and `BOLD_ADMIN_PASSWORD` in the Upgrade Center task.

| Folder | Configuration access | Reports and authentication keys | Pre/post/cleanup shared state |
| --- | --- | --- | --- |
| EFS | Existing product EFS mounted read-only | Isolated Upgrade Center EFS access point | Same isolated EFS access point, per-job directory |
| S3 | Existing S3 bucket/root | Dedicated Upgrade Center object prefix | Per-job S3 state object; no EFS required |
| EFS+S3 | Existing S3 bucket/root | Isolated Upgrade Center EFS access point | Same isolated EFS access point, per-job directory |

For EFS layouts, reuse the existing filesystem and create an Upgrade Center access point rooted at `/upgrade-center`, with a POSIX owner/write permissions suitable for the runner user (for example UID/GID 1001). Configure root-directory creation permissions on the access point. Do not point this access point at product configuration or reuse the product's authentication-key directory. EFS transit encryption is enabled. Allow the task security groups to reach EFS mount targets on TCP 2049. The S3 layout has no EFS volume or mount.

For S3 layouts, reuse `BOLD_SERVICES_STORAGETYPE=3`, bucket, root, and region. The examples use IAM task-role credentials, so no AWS access keys are required in the JSON. If the existing installation uses static keys, inject the matching existing Secrets Manager values through `secrets`, not literal environment values. The hybrid product manifests have different PV flags per component; use the actual IDP configuration source rather than copying an arbitrary service's flag.

## Roles And Network Access

Grant least-privilege access to the approved cluster, services, task families, roles, prefixes, and log groups. Do not use AdministratorAccess.

| Role | Required access |
| --- | --- |
| Upgrade Center task role | ECS DescribeServices/DescribeTaskDefinition/RegisterTaskDefinition/UpdateService/RunTask/DescribeTasks/StopTask/DeregisterTaskDefinition, tag access as required, and `iam:PassRole` only for approved product/runner roles. Application Auto Scaling DescribeScalableTargets/RegisterScalableTarget for the mapped services. CloudWatch Logs GetLogEvents. |
| Upgrade Center task role, storage | Read product S3 configuration/private keys; read/list/write its isolated report/key prefix and delete retained reports. For S3 handoff, GetObject/PutObject/DeleteObject only under the configured validation-state prefix. Customer-managed KMS permissions where encryption requires them. EFS client mount/write permissions on the Upgrade Center access point where applicable. |
| Upgrade Center task role, temporary credentials | Secrets Manager CreateSecret/DescribeSecret/GetSecretValue/DeleteSecret/TagResource restricted to `bold-upgrade-center/playwright/` and owned resources. New runner task-definition revisions are tagged with the operation ID. Do not grant deletion of product Secrets. |
| Upgrade Center execution role | Pull the UC image, publish CloudWatch logs, and resolve its existing admin Secret. |
| Playwright execution role | Pull the selected runner image, publish logs, and read the temporary runtime Secrets injected by ECS. Add KMS decrypt only if needed. |
| Playwright task role | EFS access for EFS/hybrid handoff. S3-only handoff uses short-lived, exact-object presigned URLs generated by Upgrade Center, not static AWS credentials. |

Some AWS actions do not support resource-level restrictions. Apply the supported resource/tag/cluster conditions and tightly restrict PassRole. Test the complete policy in a non-production installation.

Both tasks need database connectivity and HTTPS access to the application ALB, release API, and container registry. Private subnets need appropriate NAT/VPC endpoints for AWS APIs and registry access. The one-off runner cannot use ECS Service Connect aliases; its base URL and upload URL must be reachable through the ALB. Product services keep their existing Service Connect configuration.

## Register And Route

Run these commands from the selected storage folder after replacing placeholders:

```sh
aws ecs register-task-definition --cli-input-json file://task-definition-playwright.json --region <aws-region>
aws ecs register-task-definition --cli-input-json file://task-definition-upgrade-center.json --region <aws-region>
```

Set `RunnerTaskDefinition` to the registered runner family/revision. Create a separate ECS service for Upgrade Center with desired count 1; do not put Upgrade Center in a task that it will upgrade. This implementation uses the existing single-worker operation lock, not an HA/distributed controller. Configure replacement so two Upgrade Center controller tasks do not overlap (desired count 1 alone does not prevent rolling overlap); for example minimum healthy percent 0 and maximum percent 100. Persistent keys/history allow recovery after replacement, with a brief Upgrade Center UI interruption.

Create an ALB IP target group on port 8080, health path `/upgrade-center/health-check`, success code 200. Add host/path rules for both `/upgrade-center` and `/upgrade-center/*` before the existing IDP catch-all, without stripping the prefix. Use HTTPS. Upgrade Center does not modify listener rules automatically.

`MaintenanceWindowConfirmed=false` permits viewing Upgrade Center but blocks an upgrade before backup. Before starting an upgrade, an operator must block customer writes while allowing the validation task to access BI and upload endpoints, arrange a supported maintenance window, and redeploy Upgrade Center with that setting true. This is an operator confirmation, not an automatic ALB maintenance feature. Restore the flag after the maintenance window.

## Upgrade And Recovery

Open `https://<application-domain>/upgrade-center`, sign in locally, and select a BI release or custom patch. The existing UI and database eligibility restrictions apply. The ECS adapter validates mappings and runner metadata before backup. Main releases use the existing affected-table backup/restore planning; custom patches retain the existing schema-backup skip behavior.

Pre-validation uses the installed version's runner, post-validation uses the target version's runner. Exact mappings take priority, followed only by the same major/minor family. Custom patches retain the installed-release runner policy. Runner credentials are temporary Secrets Manager values, never plaintext RunTask overrides.

For S3-only handoff, the Upgrade Center-supplied bootstrap uploads only `ResponseLogs` and `storage` as an integrity-checked JSON envelope to one job-specific object. Post/cleanup restore this object locally before the original test entrypoint runs. Missing/corrupt state blocks execution. Product files are never included. The transport supports up to 5000 files and a bounded 64 MiB envelope; oversized state fails explicitly. Presigned URLs are private runtime credentials with lifetimes bounded to the validation timeout; do not log/export them. S3 bucket policies requiring extra upload headers must be verified with this transfer path.

Before service changes, all previous/target task-definition ARNs, desired counts, and scaling suspension state are persisted. Autoscaling is suspended and all selected old services are stopped before new services start. Upgrade order is UMS, ID API, ID Web, BI Designer, BI Web, BI API, BI Jobs, then applicable ETL/AI/MCP. Task-definition settings other than approved image changes are retained. Expect product downtime; schema-changing upgrades are not zero-downtime rolling releases.

Upgrade Center waits for the intended service revisions and completed ECS rollouts before post-validation. It reuses persisted runner task ARNs, runtime Secrets, upload tokens, and idempotency tokens after a restart. External service revisions, missing persisted tasks, and unknown ownership block automatic overwrites/reruns.

On rollback, selected services are stopped before database restore. If database restore fails, they remain stopped and backups are retained for manual recovery. Successful restore starts recorded previous revisions, not the latest revision of a family, and verifies product health. A restart during database restore still requires manual intervention, matching existing safety policy. Do not switch execution environments during an operation or try to use Kubernetes rollback records on ECS.

Cleanup stops owned validation tasks first, runs the resource-deletion tests, verifies their result, then deletes owned runtime Secrets, deregisters owned runner revisions, and removes only that job's temporary EFS/S3 state. Reports/session keys/history remain. Failed cleanup retains state for investigation and must not be treated as success. Failed setup may leave tagged artifacts if AWS permissions prevent deletion; review and remove only confirmed-owned artifacts manually.

## Validation Checklist

- Test EFS, S3-only, and hybrid configuration loading and local login; no IDP configuration/SSO API calls should be needed.
- Confirm installed/target runner selection, pre/post state continuity, report uploads, detailed logs, and cleanup.
- Test missing permissions, blocked network access, bad images, failed rollout, failed post-validation, cancellation, restart during validation/rollout, database restore failure, and cleanup deletion failure.
- Verify original task environments, Secrets, volumes, CPU/memory, target groups, Service Connect settings, and scaling state remain intact.
- Verify no product files/Secrets are deleted, failed restore retains backups, and S3-only requires no EFS.
- Existing product autoscaling policies and simultaneous customer pipeline deployments must be coordinated during the maintenance window.

Code-level tests use fake AWS clients and do not prove an AWS installation works. Complete this checklist on the working ECS deployment before production rollout. Kubernetes manifests/Helm charts are not modified by this ECS implementation.
