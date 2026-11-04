# Bold BI Upgrade Center on Amazon ECS

Upgrade Center upgrades your Bold BI services on ECS from a web page. It backs up the database, updates each service to the new image, runs automated checks (Playwright) before and after, and rolls back if something fails.

Expect downtime during an upgrade. All Bold BI services stop, then start on the new version.

## What you need

- A working Bold BI deployment on ECS Fargate (see [deployment.md](../deployment.md)).
- The task definitions in this folder. Pick the folder that matches your storage:

| Folder | Use when Bold BI uses |
| --- | --- |
| `EFS` | EFS storage |
| `S3` | S3 storage |
| `EFS+S3` | S3 for configuration and EFS for Upgrade Center files |

- Two task definition files per folder:
  - `task-definition-upgrade-center.json`: the Upgrade Center application.
  - `task-definition-playwright.json`: the test runner that Upgrade Center starts as a one-time task.

## Step 1: Prepare AWS resources

1. Create a CloudWatch log group, for example `/ecs/bold-upgrade-center`.
2. EFS and EFS+S3 only: in your existing EFS file system, create an access point with root directory `/upgrade-center` and owner UID/GID `1001`. Allow TCP 2049 from the task security group.
3. Create a task role and an execution role for Upgrade Center, and the same pair for Playwright. Grant only what is needed:

| Role | Needs |
| --- | --- |
| Upgrade Center task role | ECS describe, register, update, run and stop tasks; `iam:PassRole` for the product and Playwright roles; Secrets Manager access for `bold-upgrade-center/playwright/*`; CloudWatch Logs read. S3 and EFS: access to your Upgrade Center storage. |
| Upgrade Center execution role | Pull the image, write logs, read the admin secret. |
| Playwright roles | Pull the image, write logs. EFS access for EFS layouts. |

4. Store the admin user name and password in Secrets Manager with the keys `BOLD_ADMIN_USERNAME` and `BOLD_ADMIN_PASSWORD`. Do not put them in the task definition as plain text.

## Step 2: Fill in the task definitions

Replace every `<placeholder>` in both JSON files in your folder.

**Playwright file** (`task-definition-playwright.json`): set the roles, EFS ID and access point, log group, and the Playwright image for your Bold BI release.

**Upgrade Center file** (`task-definition-upgrade-center.json`): set these values.

| Setting | Example | Meaning |
| --- | --- | --- |
| `image` | `syncfusion/bold-upgrade-center:<version>` | Upgrade Center image. |
| `UpgradeCenter__Ecs__Region` | `us-east-1` | AWS Region. |
| `UpgradeCenter__Ecs__Cluster` | `my-cluster` | ECS cluster that runs Bold BI. |
| `UpgradeCenter__Ecs__RunnerTaskDefinition` | `bold-upgrade-center-playwright:1` | Playwright task definition, from Step 3. |
| `UpgradeCenter__Ecs__Subnets__0` | `subnet-...` | Subnet for the Playwright task. |
| `UpgradeCenter__Ecs__SecurityGroups__0` | `sg-...` | Security group for the Playwright task. |
| `UpgradeCenter__Ecs__AssignPublicIp` | `false` | Use `true` only for a public subnet with no NAT gateway. |
| `UpgradeCenter__Ecs__UploadBaseUrl` | `https://<your-domain>/upgrade-center` | Address the Playwright task uses to send results back. It must be reachable through your load balancer. |
| `UpgradeCenter__Playwright__PassRateThreshold` | `50` | Minimum percentage of tests that must pass. |
| `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` | `false` | Keep `false` until you are ready (Step 5). |

Then map each Bold BI component to your real ECS service and container names. Keep the settings named `...Services__<component>__Service` and `...__Container`.

| Component key | Your service | Your container |
| --- | --- | --- |
| `id-ums-deployment` | required | required |
| `id-api-deployment` | required | required |
| `id-web-deployment` | required | required |
| `bi-dataservice-deployment` | required | required |
| `bi-web-deployment` | required | required |
| `bi-api-deployment` | required | required |
| `bi-jobs-deployment` | required | required |
| `bold-etl-deployment` | optional | optional |
| `bold-ai-deployment` | optional | optional |
| `bold-mcp-deployment` | optional | optional |

Delete the settings for optional components you did not install.

## Step 3: Register the task definitions

```sh
aws ecs register-task-definition --cli-input-json file://task-definition-playwright.json --region <aws-region>
aws ecs register-task-definition --cli-input-json file://task-definition-upgrade-center.json --region <aws-region>
```

Copy the Playwright revision number from the first command's output into `UpgradeCenter__Ecs__RunnerTaskDefinition`. Register the Upgrade Center file again if you changed it after the first run.

## Step 4: Create the service and route traffic

1. Create an ECS service for the Upgrade Center task with desired count `1`. Set minimum healthy percent to `0` and maximum percent to `100` so two copies never run together.
2. Create an ALB target group of type IP on port `8080`. Set the health check path to `/upgrade-center/health-check` with success code `200`.
3. Add listener rules for `/upgrade-center` and `/upgrade-center/*` that point to this target group. Place them before the identity catch-all rule. Do not strip the path prefix.

## Step 5: Run an upgrade

1. Open `https://<your-domain>/upgrade-center` and sign in with the admin account.
2. Schedule a maintenance window. Block user traffic, but keep the Playwright task able to reach Bold BI.
3. Set `UpgradeCenter__Ecs__MaintenanceWindowConfirmed` to `true`, register a new revision and update the service.
4. In Upgrade Center, select the target release and start the upgrade.
5. Upgrade Center then runs these steps on its own:
   1. Checks the service mappings and test images.
   2. Backs up the database.
   3. Runs pre-upgrade tests.
   4. Stops all Bold BI services.
   5. Starts them on the new version, in this order: UMS, ID API, ID Web, BI Designer, BI Web, BI API, BI Jobs, then ETL, AI and MCP.
   6. Runs post-upgrade tests and cleans up.
6. After the upgrade, set `MaintenanceWindowConfirmed` back to `false`.

## If something fails

- Upgrade Center stops the services, restores the database and starts the previous task definition revisions.
- If the database restore fails, the services stay stopped and the backup is kept. Restore it manually.
- Do not restart Upgrade Center while a database restore is running.

## Test-only settings

Do not use these in production.

| Setting | Effect |
| --- | --- |
| `UpgradeCenter__Ecs__AllowInsecureUploadUrl=true` | Allows an `http://` upload URL. |
| `UpgradeCenter__Authentication__Standalone__AllowInsecureCookie=true` | Allows sign-in over `http`. |

## Quick checklist

- [ ] Log group, access point and roles created.
- [ ] Admin secret stored in Secrets Manager.
- [ ] Placeholders replaced in both JSON files.
- [ ] Service and container names match your ECS services.
- [ ] Playwright revision set in the Upgrade Center file.
- [ ] ALB rules and health check work at `/upgrade-center/health-check`.
- [ ] Maintenance window confirmed before the upgrade.
