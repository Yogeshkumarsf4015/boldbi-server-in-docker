# Bold BI Upgrade Center on AWS ECS (Fargate) Multi-Container

This guide walks you through deploying Upgrade Center to upgrade your Bold BI multi-container deployment on Amazon ECS Fargate.

For the design-level explanation of how ECS multi-container upgrade support works, see [ECS Upgrade Center Multi-Container Support Specification](ecs-upgrade-center-multi-container-spec.md).

Upgrade Center runs as a separate standalone Fargate service. During an upgrade it does the following in order:

1. Backs up the database tables the release changes.
2. Runs pre-upgrade tests with a one-off Playwright Fargate task.
3. Deploys new container images to your Bold BI ECS services in groups.
4. Runs post-upgrade tests and produces an HTML report.
5. Cleans up temporary test artifacts.

If a step fails, Upgrade Center restores the database backup and rolls each service back to the exact task-definition revision it had before.

Expect downtime during an upgrade. All Bold BI services stop, then start on the new version.

---

## Architecture

```
Browser / Admin
       |
       v
Application Load Balancer
  |-- /upgrade-center, /upgrade-center/*  -> Upgrade Center (port 8080)
  |-- /* (default)                         -> Bold BI services (IDP, BI Web, API, ...)
       |
       v
ECS RunTask (Fargate) -> on-demand Playwright task (pre and post tests)
       |
       v
Validation state and reports: S3 bucket (recommended) or EFS
```

Upgrade Center calls ECS APIs to update services, uses Secrets Manager for short-lived runner credentials, and reads configuration from your existing Bold BI storage (S3 or EFS).

---

## Prerequisites

1. Working Bold BI deployment on ECS Fargate (see [deployment.md](../deploy/ECS-deployment-manifest/Multi%20container/deployment.md)).
2. AWS CLI v2 with permission to create IAM roles and policies, ECS task definitions and services, and ALB target groups.
3. Administrator secret in Secrets Manager. The JSON keys must be `BOLD_ADMIN_USERNAME` and `BOLD_ADMIN_PASSWORD`.
4. Outbound internet access from the Upgrade Center and runner subnets (NAT gateway or VPC endpoints) to AWS APIs, the Bold BI release API, the image registries, and the ALB.

---

## Choose storage for validation state

The Playwright runner passes test state between pre-upgrade, post-upgrade, and cleanup. Pick one.

### Option A: S3 (recommended)

Create a private bucket and block public access:

```sh
aws s3 mb s3://<state-bucket> --region <aws-region>
aws s3api put-public-access-block --bucket <state-bucket> \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Option B: EFS

Create an access point on your existing EFS file system:

- Path: `/upgrade-center`
- POSIX user: UID `1001`, GID `1001`
- Owner: UID `1001`, GID `1001`, permissions `0755`
- Allow TCP `2049` from the task subnets.

Use the `EFS` task-definition folder for EFS, `S3` for S3, and `EFS+S3` when you use S3 for Bold BI configuration and a separate EFS access point for Upgrade Center.

---

## Step 1: Create CloudWatch log groups

```sh
aws logs create-log-group --log-group-name /ecs/bold-upgrade-center --region <aws-region>
aws logs create-log-group --log-group-name /ecs/bold-upgrade-center-playwright --region <aws-region>
aws logs put-retention-policy --log-group-name /ecs/bold-upgrade-center         --retention-in-days 30 --region <aws-region>
aws logs put-retention-policy --log-group-name /ecs/bold-upgrade-center-playwright --retention-in-days 30 --region <aws-region>
```

---

## Step 2: Create IAM roles

You need four roles.

| Role | Used by | Purpose |
| --- | --- | --- |
| Upgrade Center execution role | ECS | Pull the Upgrade Center image, write logs, inject the admin secret. |
| Upgrade Center task role | Upgrade Center | Manage ECS services, secrets, and storage. |
| Playwright execution role | ECS | Pull the runner image, write logs, read injected secrets. |
| Playwright task role | Playwright | Access the validation state (EFS only). |

### Execution role (Upgrade Center and Playwright)

Attach the AWS managed policy `AmazonECSTaskExecutionRolePolicy`. For the Upgrade Center execution role, also allow reading the admin secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:<admin-secret-name>*"
  }]
}
```

### Upgrade Center task role

Attach the following inline policies.

ECS management:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["ecs:DescribeServices","ecs:DescribeTaskDefinition","ecs:DescribeTasks"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["ecs:RegisterTaskDefinition","ecs:DeregisterTaskDefinition","ecs:TagResource"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["ecs:UpdateService","ecs:RunTask","ecs:StopTask"],
      "Resource": [
        "arn:aws:ecs:<aws-region>:<account-id>:service/<cluster>/*",
        "arn:aws:ecs:<aws-region>:<account-id>:task/<cluster>/*",
        "arn:aws:ecs:<aws-region>:<account-id>:task-definition/*"
      ]
    },
    { "Effect": "Allow", "Action": ["application-autoscaling:DescribeScalableTargets","application-autoscaling:RegisterScalableTarget"], "Resource": "*" },
    { "Effect": "Allow", "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::<account-id>:role/<upgrade-center-execution-role>",
        "arn:aws:iam::<account-id>:role/<product-task-role>",
        "arn:aws:iam::<account-id>:role/<playwright-task-role>"
      ],
      "Condition": { "StringEquals": { "iam:PassedToService": "ecs-tasks.amazonaws.com" } }
    }
  ]
}
```

Runtime secrets and logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["secretsmanager:CreateSecret","secretsmanager:DescribeSecret","secretsmanager:GetSecretValue","secretsmanager:DeleteSecret","secretsmanager:TagResource"],
      "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:bold-upgrade-center/playwright/*"
    },
    { "Effect": "Allow", "Action": "logs:GetLogEvents",
      "Resource": "arn:aws:logs:<aws-region>:<account-id>:log-group:/ecs/bold-upgrade-center-playwright:*"
    }
  ]
}
```

Storage:

- S3:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject","s3:ListBucket"],
      "Resource": ["arn:aws:s3:::<state-bucket>","arn:aws:s3:::<state-bucket>/*"]
    }]
  }
  ```
- EFS:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["elasticfilesystem:ClientMount","elasticfilesystem:ClientWrite"],
      "Resource": "arn:aws:elasticfilesystem:<aws-region>:<account-id>:file-system/<efs-id>",
      "Condition": { "StringEquals": {
        "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:<aws-region>:<account-id>:access-point/<upgrade-center-access-point-id>"
      } }
    }]
  }
  ```

### Playwright task role

For EFS, attach the same EFS policy as above. For S3-only, no storage policy is needed: Upgrade Center issues short-lived presigned URLs. The task definition still needs a role ARN, so create an empty role.

---

## Step 3: Register the Playwright runner task

Open `deploy/ECS-deployment-manifest/Multi container/UpgradeCenter/<S3|EFS|EFS+S3>/task-definition-playwright.json` and replace the placeholders:

- `<playwright-execution-role-arn>`
- `<playwright-task-role-arn>`
- `<upgrade-center-log-group>` -> `/ecs/bold-upgrade-center-playwright`
- `<aws-region>`
- `<existing-efs-file-system-id>` and `<upgrade-center-efs-access-point-id>` (EFS only)

Keep `requiresCompatibilities: ["FARGATE"]`, `networkMode: "awsvpc"`, `cpu: "2048"`, `memory: "4096"`.

Register it:

```sh
aws ecs register-task-definition \
  --cli-input-json file://task-definition-playwright.json \
  --region <aws-region>
```

Note the family and revision, for example `bold-upgrade-center-playwright:1`. You will enter it into the Upgrade Center task as `RunnerTaskDefinition`.

---

## Step 4: Configure and register the Upgrade Center task

Open `task-definition-upgrade-center.json` in the same folder.

### Replace the placeholders

- `<upgrade-center-task-role-arn>`, `<upgrade-center-execution-role-arn>`
- `<upgrade-center-log-group>` -> `/ecs/bold-upgrade-center`
- `<aws-region>`, `<ecs-cluster-name>`
- `<private-subnet-id>`, `<validation-security-group-id>`
- `<application-domain>` -> your ALB host name
- For EFS: `<existing-efs-file-system-id>`, `<upgrade-center-efs-access-point-id>`
- For S3: `<existing-product-bucket>`, `<existing-product-root>`

### Key environment variables

| Variable | Value |
| --- | --- |
| `UpgradeCenter__ExecutionEnvironment` | `ECS` |
| `UpgradeCenter__IntegrationMode` | `Standalone` |
| `UpgradeCenter__DeploymentMode` | `BoldBI` |
| `UpgradeCenter__BiReleaseVersionsApiUrl` | `https://releases.boldbi.com/version/json` |
| `UpgradeCenter__Ecs__Region` | your AWS region |
| `UpgradeCenter__Ecs__Cluster` | your ECS cluster name |
| `UpgradeCenter__Ecs__RunnerTaskDefinition` | the runner family and revision from Step 3 |
| `UpgradeCenter__Ecs__RunnerContainer` | `playwright` |
| `UpgradeCenter__Ecs__Subnets__0` | private subnet id |
| `UpgradeCenter__Ecs__SecurityGroups__0` | security group id |
| `UpgradeCenter__Ecs__UploadBaseUrl` | `https://<application-domain>/upgrade-center` |
| `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` | `false` during setup |
| `UpgradeCenter__Ecs__AssignPublicIp` | `false` |
| `UpgradeCenter__Playwright__PassRateThreshold` | `50` |

### Map each Bold BI component

```text
UpgradeCenter__Ecs__Services__<component-key>__Service   = <ecs-service-name>
UpgradeCenter__Ecs__Services__<component-key>__Container = <container-name>
```

Required component keys: `id-ums-deployment`, `id-api-deployment`, `id-web-deployment`, `bi-dataservice-deployment`, `bi-web-deployment`, `bi-api-deployment`, `bi-jobs-deployment`. Optional: `bold-etl-deployment`, `bold-ai-deployment`, `bold-mcp-deployment`. Remove optional mappings for services that are not installed.

### Admin secret

The task definition uses `secrets` for `BOLD_ADMIN_USERNAME` and `BOLD_ADMIN_PASSWORD`. Replace the `valueFrom` ARNs with your admin secret. Do not put the password in plain text in `environment`.

### Register it

```sh
aws ecs register-task-definition \
  --cli-input-json file://task-definition-upgrade-center.json \
  --region <aws-region>
```

---

## Step 5: Create the Upgrade Center service and ALB rule

Create a Fargate service in your Bold BI cluster with desired count 1. Set minimum healthy percent to `0` and maximum percent to `100` so only one Upgrade Center task runs at a time. Use the subnets and security group from the task definition.

Create a target group:

- Target type: IP
- Protocol: HTTP, Port: 8080
- Health check path: `/upgrade-center/health-check`, success codes: 200

Add an HTTPS listener rule that forwards `/upgrade-center` and `/upgrade-center/*` to the new target group. Set the priority below the Bold BI catch-all rule for `/*`. Do not rewrite the path; Upgrade Center expects the `/upgrade-center` prefix.

---

## Step 6: Run an upgrade

1. Take a full RDS or Aurora snapshot of the Bold BI database.
2. Pause public customer writes on the ALB while keeping `/upgrade-center` reachable.
3. Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed=true` on the Upgrade Center service and apply the update.
4. Open `https://<application-domain>/upgrade-center` and sign in with the administrator credentials. Confirm the banner shows your AWS region and cluster, the current version, and database connectivity.
5. Pick the target release under Available Releases and start the upgrade.
6. Watch the stages: database backup, pre-upgrade validation, image rollout, post-upgrade validation, cleanup.
7. When the UI shows Completed, review the HTML report and logs.
8. Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed=false` and reopen customer traffic.

---

## Rollback

Open the History tab in Upgrade Center and click Rollback on the failed job. Upgrade Center stops the services, restores the database from the backup, reverts each service to the recorded previous task-definition revision, and checks service health.

If database restore fails, the services stay stopped and the backup is kept. Recover manually.

Do not switch execution environments during an operation. Do not try to use Kubernetes rollback records on ECS.

---

## Troubleshooting

| Issue | Cause | Resolution |
| --- | --- | --- |
| `AccessDeniedException: iam:PassRole` | Upgrade Center task role cannot pass a required role | Add the execution role, runner role, and Bold BI service task roles to the `iam:PassRole` statement. |
| Pre-upgrade shows `0 of 0 tests` or `EACCES` | Validation state is not writable | For S3, confirm the bucket policy and `UpgradeCenter__Ecs__StateBucket`. For EFS, confirm the access point POSIX owner is UID `1001`. |
| `ECS image repository is not approved` | Target registry is not on the allow-list | Keep `UpgradeCenter__Kubernetes__AllowedImageRegistryHosts__0=docker.io` and `UpgradeCenter__Kubernetes__AllowedImageRepositories__0=syncfusion`. |
| `An operator must block customer writes and confirm the ECS maintenance window` | Safety check is active | Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed=true` during maintenance. |
| Playwright runner times out or runs out of memory | Runner size too small | Keep `cpu: "2048"` and `memory: "4096"` in the runner task definition. |
| Runner cannot reach Upgrade Center | Security group or subnet issue | Confirm outbound internet from the subnets and that the security group allows the ALB. |
| Upgrade Center stuck after a restart | External task revision no longer exists | Re-run the upgrade from the UI; do not import old persisted task ARNs. |

---

## Validation checklist

- Log in locally; no IDP or SSO call is made.
- Pick a release and confirm pre and post validation both run.
- Confirm report upload, logs, and cleanup.
- Test a failed upgrade and confirm rollback restores the database and the previous task-definition revisions.
- Test bad IAM permissions, blocked network, a missing image, and a cancelled run.
- Confirm product task definitions, secrets, CPU, memory, target group, Service Connect settings, and autoscaling are unchanged after a successful upgrade.
- Confirm no product files or secrets were deleted. A failed restore must keep the backup.
- S3-only setup must not require EFS.
