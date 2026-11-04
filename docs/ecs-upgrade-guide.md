# Bold BI Upgrade Center on AWS ECS (Fargate)

This guide walks you through deploying and using Bold Upgrade Center to upgrade your existing Bold BI multi-container deployment on Amazon ECS (Fargate).

Upgrade Center runs as a separate standalone ECS Fargate service. During an upgrade, it automatically:
1. Backs up database tables affected by the release.
2. Runs pre-upgrade validation tests using an on-demand Playwright Fargate task.
3. Deploys updated container image versions across your Bold BI ECS services in ordered groups.
4. Runs post-upgrade validation tests and generates HTML reports.
5. Cleans up temporary test artifacts.

If an issue occurs, Upgrade Center restores your database backup and rolls services back to their exact previous task-definition revisions.

---

## Architecture Overview

```
Browser / Admin
       │
       ▼
Application Load Balancer (ALB)
  ├── Host / Path: /upgrade-center, /upgrade-center/* ──► Bold Upgrade Center (Port 8080)
  │                                                                │
  └── Host / Path: /* (Default catch-all) ─────────────► Bold BI Services (IDP, BI Web, API, etc.)
                                                                   │
       ┌───────────────────────────────────────────────────────────┴────────────────────────────────────────┐
       ▼                                                           ▼                                        ▼
AWS ECS API                                            AWS Secrets Manager                         ECS RunTask (Fargate)
  • Describe / update services                           • Temporary runner secrets                  • On-demand Playwright task
  • Register / deregister task definitions                 (created and deleted per run)             • Pre-upgrade validation
  • Roll out or roll back revisions                                                                 • Post-upgrade validation
                                                                                                            │
                                                                                                            ▼
                                                                                                  Validation State & Reports
                                                                                                    • S3 Bucket (recommended) or EFS
                                                                                                    • Uploads reports to Upgrade Center
```

---

## Prerequisites

Before setting up Upgrade Center, verify you have:

1. **Existing Bold BI ECS Deployment**: All core Bold BI services must be active with desired count >= 1 (`id-ums-deployment`, `id-api-deployment`, `id-web-deployment`, `bi-dataservice-deployment`, `bi-web-deployment`, `bi-api-deployment`, `bi-jobs-deployment`). Optional services (`bold-etl-deployment`, `bold-ai-deployment`, `bold-mcp-deployment`) can be included if installed.
2. **AWS CLI v2**: Configured with administrative access to create IAM policies, ECS tasks, and ALB target groups.
3. **Admin Secret in AWS Secrets Manager**: The existing administrator secret containing `BOLD_ADMIN_USERNAME` and `BOLD_ADMIN_PASSWORD`.
4. **Network Access**: The subnets used by Upgrade Center and the Playwright runner must have outbound internet access (via NAT Gateway or VPC endpoints) to reach:
   - AWS APIs (ECS, Secrets Manager, CloudWatch Logs, Application Auto Scaling, S3)
   - Bold BI Release API: `https://releases.boldbi.com/version/json`
   - Image registries (`docker.io`, `us-docker.pkg.dev`)
   - The application ALB endpoint

---

## Step 1: Choose Storage for Validation State

The Playwright runner transfers test session state between pre-upgrade, post-upgrade, and cleanup stages. Choose one of two options:

- **Option A: S3 (Recommended)**: Fastest and simplest. No EFS mounts needed for the runner or Upgrade Center state.
- **Option B: EFS**: Uses an access point on your existing product EFS filesystem.

### If using Option A (S3 - Recommended)
Use your existing Bold BI S3 bucket or create a private bucket:

```bash
aws s3 mb s3://<your-state-bucket> --region <aws-region>

aws s3api put-public-access-block \
  --bucket <your-state-bucket> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### If using Option B (EFS)
Create an access point on your existing EFS filesystem:
- **Path**: `/upgrade-center`
- **POSIX User**: UID `1001`, GID `1001`
- **Owner**: UID `1001`, GID `1001`, Permissions `0755`
- Ensure port `2049` (NFS) is allowed in your EFS security group from the task subnets.

---

## Step 2: Create CloudWatch Log Groups

Create dedicated CloudWatch log groups for Upgrade Center and the Playwright runner:

```bash
aws logs create-log-group --log-group-name /ecs/bold-upgrade-center --region <aws-region>
aws logs create-log-group --log-group-name /ecs/bold-upgrade-center-playwright --region <aws-region>

aws logs put-retention-policy --log-group-name /ecs/bold-upgrade-center --retention-in-days 30 --region <aws-region>
aws logs put-retention-policy --log-group-name /ecs/bold-upgrade-center-playwright --retention-in-days 30 --region <aws-region>
```

---

## Step 3: Configure IAM Roles and Policies

You need two IAM roles:
1. **ECS Task Execution Role** (`UpgradeCenterExecutionRole`): Used by ECS to pull container images, write logs, and inject secrets.
2. **Upgrade Center Task Role** (`UpgradeCenterTaskRole`): Used by the Upgrade Center application to manage ECS services, Secrets Manager, and storage.

### 3.1 Task Execution Role
Attach the AWS-managed policy `AmazonECSTaskExecutionRolePolicy`.

If you use AWS Secrets Manager for administrator credentials, also attach an inline policy allowing `secretsmanager:GetSecretValue` on the admin secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:<existing-admin-secret-name>*"
    }
  ]
}
```

### 3.2 Upgrade Center Task Role
Attach the following inline policies to `UpgradeCenterTaskRole`:

#### Policy 1: ECS Management and Service Updates (`UpgradeCenterEcsUpgrade`)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcsRead",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeTasks"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EcsTaskDefinitions",
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:DeregisterTaskDefinition",
        "ecs:TagResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EcsServicesAndTasks",
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:RunTask",
        "ecs:StopTask"
      ],
      "Resource": [
        "arn:aws:ecs:<aws-region>:<account-id>:service/<ecs-cluster-name>/*",
        "arn:aws:ecs:<aws-region>:<account-id>:task/<ecs-cluster-name>/*",
        "arn:aws:ecs:<aws-region>:<account-id>:task-definition/*"
      ]
    },
    {
      "Sid": "Autoscaling",
      "Effect": "Allow",
      "Action": [
        "application-autoscaling:DescribeScalableTargets",
        "application-autoscaling:RegisterScalableTarget"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassRoles",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::<account-id>:role/<execution-role-name>",
        "arn:aws:iam::<account-id>:role/<product-service-task-role-name>",
        "arn:aws:iam::<account-id>:role/<runner-task-role-name>"
      ],
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```

#### Policy 2: Playwright Runtime Credentials and Logs (`UpgradeCenterPlaywrightRuntime`)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RuntimeSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:DescribeSecret",
        "secretsmanager:GetSecretValue",
        "secretsmanager:DeleteSecret",
        "secretsmanager:TagResource"
      ],
      "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:bold-upgrade-center/playwright/*"
    },
    {
      "Sid": "RunnerLogs",
      "Effect": "Allow",
      "Action": "logs:GetLogEvents",
      "Resource": "arn:aws:logs:<aws-region>:<account-id>:log-group:/ecs/bold-upgrade-center-playwright:*"
    }
  ]
}
```

#### Policy 3: Storage Access
- **For S3 Storage**:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::<your-state-bucket>",
          "arn:aws:s3:::<your-state-bucket>/*"
        ]
      }
    ]
  }
  ```
- **For EFS Storage**:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite"
        ],
        "Resource": "arn:aws:elasticfilesystem:<aws-region>:<account-id>:file-system/<existing-efs-file-system-id>",
        "Condition": {
          "StringEquals": {
            "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:<aws-region>:<account-id>:access-point/<upgrade-center-efs-access-point-id>"
          }
        }
      }
    ]
  }
  ```

---

## Step 4: Register the Playwright Runner Task Definition

Register the runner task definition from your chosen storage template directory (`S3/task-definition-playwright.json` or `EFS/task-definition-playwright.json`):

1. Open `task-definition-playwright.json`.
2. Replace all placeholder values:
   - `<playwright-execution-role-arn>`: ARN of your task execution role.
   - `<playwright-task-role-arn>`: ARN of your task role.
   - `<upgrade-center-log-group>`: `/ecs/bold-upgrade-center-playwright`.
   - `<aws-region>`: Your AWS region.
3. Keep `requiresCompatibilities: ["FARGATE"]`, `networkMode: "awsvpc"`, `cpu: "2048"`, and `memory: "4096"`.
4. Register the definition:

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition-playwright.json \
  --region <aws-region>
```

Note the registered task definition family and revision (e.g., `bold-upgrade-center-playwright:1`).

---

## Step 5: Configure and Register the Upgrade Center Task Definition

Navigate to your chosen storage directory (`S3/`, `EFS/`, or `EFS+S3/`) and edit `task-definition-upgrade-center.json`:

### 5.1 Key Environment Variables

| Variable | Description / Value |
|---|---|
| `UpgradeCenter__ExecutionEnvironment` | `ECS` |
| `UpgradeCenter__IntegrationMode` | `Standalone` |
| `UpgradeCenter__DeploymentMode` | `BoldBI` |
| `UpgradeCenter__BiReleaseVersionsApiUrl` | `https://releases.boldbi.com/version/json` |
| `UpgradeCenter__Ecs__Region` | Your AWS region (e.g., `us-east-1`) |
| `UpgradeCenter__Ecs__Cluster` | Name of your existing Bold BI ECS cluster |
| `UpgradeCenter__Ecs__RunnerTaskDefinition` | Registered runner definition from Step 4 (e.g., `bold-upgrade-center-playwright:1`) |
| `UpgradeCenter__Ecs__RunnerContainer` | `playwright` |
| `UpgradeCenter__Ecs__Subnets__0` | Private subnet ID where tasks will run |
| `UpgradeCenter__Ecs__SecurityGroups__0` | Security group ID allowing access to ALB, database, and AWS endpoints |
| `UpgradeCenter__Ecs__UploadBaseUrl` | Public URL the runner uses to reach Upgrade Center (e.g., `https://<application-domain>/upgrade-center`) |
| `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` | Set to `false` during setup. Must be set to `true` during scheduled maintenance before running an upgrade. |

### 5.2 Service and Container Mappings
Map each Bold BI component key to your real ECS service name and container name:

```text
UpgradeCenter__Ecs__Services__<component-key>__Service   = <real-ecs-service-name>
UpgradeCenter__Ecs__Services__<component-key>__Container = <container-name-in-task-definition>
```

Required components:
- `id-ums-deployment`
- `id-api-deployment`
- `id-web-deployment`
- `bi-dataservice-deployment`
- `bi-web-deployment`
- `bi-api-deployment`
- `bi-jobs-deployment`

Optional components (remove these environment variables if the service is not installed in your cluster):
- `bold-etl-deployment`
- `bold-ai-deployment`
- `bold-mcp-deployment`

### 5.3 Register the Task Definition

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition-upgrade-center.json \
  --region <aws-region>
```

---

## Step 6: Create the ECS Service and Configure ALB Routing

### 6.1 Create the Upgrade Center Service
Create a standalone Fargate service in your existing Bold BI cluster with **desired count 1**:

- **Launch type**: Fargate
- **Task definition**: `bold-upgrade-center`
- **Desired tasks**: `1`
- **Deployment configuration**:
  - Minimum healthy percent: `0`
  - Maximum percent: `100` *(prevents two Upgrade Center tasks from running simultaneously)*
- **Networking**: Assign the private subnets and security group configured in Step 5.

### 6.2 Configure Application Load Balancer (ALB) Routing
1. **Create Target Group**:
   - Target type: `IP`
   - Protocol: `HTTP`, Port: `8080`
   - VPC: Select your Bold BI VPC
   - Health check path: `/upgrade-center/health-check`
   - Success codes: `200`
2. **Add ALB Listener Rules**:
   - On your HTTPS listener (Port 443), add a rule:
     - **Path conditions**: `/upgrade-center` and `/upgrade-center/*`
     - **Action**: Forward to the Upgrade Center Target Group
     - **Priority**: Set priority higher than the default Bold BI catch-all rule (`/*`)
     - **Path handling**: Do not strip the prefix

---

## Step 7: Perform an Upgrade

Follow these steps when performing a planned upgrade:

1. **Take a Full Database Snapshot**:
   Take a snapshot of your relational database in Amazon RDS / Aurora.
2. **Set Maintenance Window Confirmation**:
   In your Upgrade Center task definition or service environment, update:
   ```text
   UpgradeCenter__Ecs__MaintenanceWindowConfirmed=true
   ```
   Apply the update to the service.
3. **Block Customer Traffic**:
   Pause incoming public customer write requests on your ALB while allowing access to `/upgrade-center` and `/upgrade-center/*`.
4. **Start the Upgrade in the UI**:
   - Open `https://<application-domain>/upgrade-center` in your browser.
   - Log in with your Bold BI Administrator credentials.
   - Verify the top status banner shows `AWS / ECS FARGATE`, your current version, and database connectivity.
   - Under **Available Releases**, select your target release and click **Start Upgrade**.
5. **Monitor Upgrade Stages**:
   - **Database Backup**: Backs up affected application tables.
   - **Pre-upgrade Validation**: Runs automated Playwright tests against your current version.
   - **ECS Image Rollout**: Deploys updated container versions across services in parallel groups.
   - **Post-upgrade Validation**: Runs automated Playwright tests against the new version.
   - **Cleanup**: Removes temporary runtime secrets and validation state.
6. **Restore Regular Operations**:
   - Once the UI displays **Completed**, review the test reports and logs.
   - Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed=false`.
   - Reopen customer traffic on the ALB.

---

## Rollback Procedure

If an upgrade fails or post-validation detects issues:
1. Open the **History** tab in the Upgrade Center portal.
2. Select the failed job and click **Rollback**.
3. Upgrade Center will:
   - Stop active services.
   - Restore database tables from the pre-upgrade backup snapshot.
   - Revert services to their exact recorded previous task-definition revisions.
   - Verify health across all restored services.

---

## Troubleshooting Guide

| Issue | Cause | Resolution |
|---|---|---|
| `AccessDeniedException: ... iam:PassRole` | Upgrade Center task role lacks `iam:PassRole` permission for a service or runner role | Add the execution role, runner task role, and Bold BI service task roles to Policy 1 (`PassRoles` statement). |
| Pre-upgrade shows `0 of 0 tests` or `EACCES: permission denied` | Validation state directory is not writable | If using S3, ensure `UpgradeCenter__Ecs__StateBucket` is set. If using EFS, verify the access point POSIX owner is UID 1001. |
| `ECS image repository is not approved` | Target image registry is missing from the allow-list | Ensure `UpgradeCenter__Kubernetes__AllowedImageRegistryHosts__0=docker.io` and `UpgradeCenter__Kubernetes__AllowedImageRepositories__0=syncfusion` are set in the task definition. |
| `An operator must block customer writes and confirm the ECS maintenance window` | Safety check is active | Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed=true` in the Upgrade Center environment during maintenance. |
| Playwright runner fails with timeout or out of memory | Runner task size is insufficient | Ensure `task-definition-playwright.json` allocates at least 2 vCPU (`2048`) and 4 GB RAM (`4096`). |
| Runner cannot reach Upgrade Center or Bold BI | Security group or routing restriction | Verify subnets have outbound internet access and security groups allow traffic to the ALB. |
