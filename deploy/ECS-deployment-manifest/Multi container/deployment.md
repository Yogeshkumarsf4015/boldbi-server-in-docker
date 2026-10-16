## 1. Overview
This guide deploys Bold BI on AWS ECS Fargate using Amazon EFS/S3/EFS+S3 storages, and an Application Load Balancer.

For optional in-app upgrades after installation, see [Upgrade Center on ECS](UpgradeCenter/README.md). It uses a separate Standalone Upgrade Center service and supports the EFS, S3-only, and EFS+S3 layouts described here.

## 2. Prerequisites
•	Amazon ECS cluster - https://docs.aws.amazon.com/AmazonECS/latest/userguide/create_cluster.html
•	Amazon S3 bucket - https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html
•       Amazon EFS - https://docs.aws.amazon.com/efs/latest/ug/gs-step-two-create-efs-resources.html
•	IAM roles: ECS task execution role + a task role with S3 / CloudWatch access
•	Application Load Balancer (Internet-facing) with at least 2 subnets across AZs
•	Relational database (PostgreSQL, MySQL, Oracle or MSSQL) reachable from the ECS subnets
•	All Bold BI resources in the same VPC and Security Group

## 3. ECS Cluster
After prerequisites, the new cluster appears on the ECS Clusters list.
 
 ![Picture1.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzA4Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.rjnsJ8H85TNIpSLth-sTMt28gw5Jxd_3VPXC_FeN4uM)

## 4. Task Definitions

### 4.1 Placeholders used in every task definition

| Placeholder | What to put in |
|----------|-----------|
| `<service-task-definition-name>` | ECS Task Definition family name for the service. |
| `<service-container-name>` | Container name inside the task definition. |
| `<update the latest image>` | Container image repository with latest version tag. |
| `<application-domain>` | Public DNS domain mapped to the Application Load Balancer. |
| `<provide the s3 bucket name>` | Name of the dedicated Amazon S3 bucket. |
| `<provide the aws root folder name>>` | Root prefix/folder path inside the S3 bucket. |
| `<aws-access-key-id>` / `<aws-secret-access-key>` | AWS IAM credentials (can be left blank if IAM Task Role is used). |
| `<provide the DB type>` | Database engine type (`PostgreSQL`/ `MySQL`/ `Oracle`/ `MSSQL`). |
| `<provide the DB host>` | Endpoint or IP address of the database server. |
| `<provide the DB port number>` | Database port number (PostgreSQL: 5432, MySQL: 3306, MSSQL: 1433). |
| `<provide the DB maintenance DB name>` | Default maintenance database name. |
| `<provide the DB user name>` | Database master or application user with DDL rights. |
| `<provide the DB name>` | Target database name for Bold BI application data. |
| `<provide the user email ID>` | Initial Bold BI administrator email address. |
| `<aws-region>` | AWS region where the ECS cluster and services reside. |
| `<ecs-task-role-arn>` | ARN of the IAM Task Role (grants containers S3/CloudWatch access). |
| `<ecs-task-execution-role-arn>` | ARN of the ECS Task Execution Role (grants ECS access to ECR/Secrets). |
| `<account-id>` | 12-digit AWS Account ID. |
| `<secret-name>` | Secret name defined in AWS Secrets Manager. |
| `<efs-volume-name>` | Name given to the EFS volume in the task definition storage settings. | 
| `<efs-file-system-id>` | File system ID of the Amazon EFS instance. |

### 4.2 Task Definition Files - Service Reference Table

⚠️ **Task definitions for all Bold BI services are maintained in the GitHub repository:**

1. From the sidebar click Task definitions › Create new task definition with JSON.
 
 ![Picture2.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzA5Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.yJZ0QCxBg_QUsrThXJCIXOaKzFqArpMUyk4SM1SqxKU) 

2. Based on the storage type use the Bold BI task definitions from below GitHub
**GitHub Repository Location:** [GitHub - Bold BI ECS Task Definitions](https://github.com/KingsleyAnbinnesan/ECS-multi-container-templates/tree/main/ECS-multi-container-task-definition/Bold%20BI)

3. Use the latest images from https://github.com/boldbi/boldbi-docker/blob/main/deploy/multiple-container/docker-compose.yml.

### Secrets

⚠️ **Secrets:** Replace the AWS access, AWS secret key, DB password and user password into AWS Secrets Manager and reference it via the `secrets` block (valueFrom).

### 4.3 AWS Secrets Manager Creation

The following steps describe how to create the secret in AWS Secrets Manager for the AWS access keys, etc.

#### Steps

1. Go to **Secret manager** Page

2. Click **Store a new secret**

 ![Picture23.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzE1Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.ObCHiWvKjtqXcPl4jtigbv3Ev-rX0a56lgTm5Deoa_s)

3. Choose Secret type as **Other type of secret**
 
 ![Picture24.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzE2Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.4R6j_43UmpBg7-1_pKTZ_Fiwtnz6bxExtKuZNaTybOY)

4. Next add the **key/value pairs**
 
 ![Picture25.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzE3Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.CSr0aZkBaUQ6uVHMzPu0bYqTk6DNPc-odabMifwOn6c)

5. Next set the **Encryption key** as **aws/secretsmanager**
 
 ![Picture26.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzE4Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.qvXMpYDQTkeA72TZCvVvCNZNf-mfPaBpz4ctDw4pzlI)

6. Next set the **secret name and description**
 
 ![Picture27.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzE5Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.8yWYr-BsqrXPckKrmmt4gTmlhpz_XrcfsT7LYVh6nP0)

7. Next set the **tag** if needed.
 
 ![Picture28.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzIwIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.VI6V8XlzVrrzbwI3koBo6WGCaQpyW0c3q5Mdy2TzxeA)

8. Next set the **default options** for Resource permission, Replicate secret, Configure rotation.
 
 ![Picture29.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzIxIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.uheWdWb_a24t97wpcg3-fr2N_QsQ28a_9_HOhA8s5ik)

9. Next check the **review**.
 
 ![Picture30.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzIyIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.XPWMv1vbZ-lkVMaURimUu9KjOFJCkzDzrivJnyP3VGo)

10. Next click **store** to create the secret manager.

#### Policies

**Policy need to attach with the IAM user account:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSpecificSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:<secret-name>*"
    },
    {
      "Sid": "ListSecretsForConsole",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:ListSecrets"
      ],
      "Resource": "*"
    }
  ]
}
```

**Policy needs to attach with the task definition role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:<aws-region>:<account-id>:secret:<secret-name>*"
    }
  ]
}
```


### 4.4 Suggested CPU and Memory to support concurrent users

| Service | CPU (milli core) | Memory (GB) | Desired Tasks |
|---------|-----------|-----------|---------------|
| ID WEB | 512 | 1 | 1 |
| ID UMS | 512 | 1 | 1 |
| ID API | 512 | 1 | 1 |
| BI WEB | 512 | 1 | 1 |
| BI API | 512 | 1 | 1 |
| BI JOBS | 512 | 1 | 1 |
| BI DESIGNER | 512 | 1 | 1 |
| BOLD ETL | 512 | 1 | 1 |
| BI AI | 512 | 1 | 1 |
| BI MCP | 512 | 1 |1 |


## 5. Create Target Groups

### Create Target Groups

Each Bold BI service needs its own target group.

### 5.1 Create Target Group for each services

1. Go to **EC2 > Target Groups > Create target group**
 
 ![Picture3.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzIzIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.srJwRm7EsKassOcqMjVNM1hy4BPhkuAqanatN3TRePw)

2. Choose the Target type as **IP address** and provide the target group name
3. Choose the protocol as **HTTP** and choose the respective VPC
 
 ![Picture4.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI0Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.nlaHCKmyhCTtEcOZWrG23My8ONJkXypXrmI4NCEHKlY)

4. Provide health check endpoint for each services as mentioned in the table below
 
 ![Picture5.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI1Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.G8TGpK_ZjQxfiOoqvFaF5rGrT4ToXhDzwRTQrYotRfQ)


### Health-check paths per target group:

| Service | Health check path |
|---------|------------------|
| IDP Web | /health-check |
| IDP UMS | /health-check |
| IDP API | /health-check |
| BI Web | /health-check |
| BI Jobs | /health-check |
| BI API | /health-check |
| BI Designer | /health-check |
| BI ETL | /etlservice/health-check |
| BI AI | /aiservice/health-check |
| BI MCP | /health-check |

5. Under the **Advanced health check settings** change the **Success Codes** value from 200 to 200-399
 ![Picture38.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Ijc0NTIwIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.9Sc0OCpcBN1ts7fIZnH3Ufngc06WllXKEcQkNFsMRCQ)

6. Click **Next** and click **Create target group**
7. Follow the above steps and create target group for each service

### Suggested target-group names:

| Service | Suggested TG name | Port |
|---------|------------------|------|
| IDP Web | tg-idp-web | 80 |
| IDP UMS | tg-idp-ums | 80 |
| IDP API | tg-id-api | 80 |
| BI Web | tg-bi-web | 80 |
| BI Jobs | tg-bi-jobs | 80 |
| BI API | tg-bi-api | 80 |
| BI Designer | tg-bi-designer | 80 |
| BI ETL | tg-bold-etl | 80 |
| BI AI | tg-bi-ai | 80 |
| BI MCP | tg-bi-mcp | 80 |

⚠️ **All target groups must live in the same VPC as the ECS cluster.**


## 6. Application Load Balancer

### Application Load Balancer

1. Go to **EC2 > Load balancers > Create load balancer**
2. Create **Application Load balancer**
 
 ![Picture6.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI2Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.QC8niX25vT7kERWJAAIWJS9yjdzpzdLaShT3UMwO58w)

3. To connect all the services, create a common load balancer and choose a listener based on your preference (80 or 443)

⚠️ **Screenshots show 443 Listener**

4. Create the load balancer with:
   - Load balancer name
   - Internet-facing scheme
   - IPv4 IP address type
   - Same VPC and subnets/security group as the cluster

 ![Picture7.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI3Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.0bdvWPaiOxyU9WsYXHrGdiBZA9pJKqvJV5lJP3KGI1w)
 
 ![Picture8.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI4Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.C7dRkwGYcLlUYOzq8gwccwFZMd7r4ObVlHnTeSEx864)

5. For the **Listeners and routing:**
   - ALB listens to **HTTPS:443** (and HTTP:80 redirected to HTTPS)
   - Choose the **ID-WEB**'s target group name
   - For HTTPS, import the respective SSL certificate
 
 ![Picture9.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzI5Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.hNcaI-P3x2HHZOzUY_pdV9BMf9YA_0628yhugX8NB1M)
 
 ![Picture10.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzMwIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.0oXJD7Bm2HhiuTk57o3geu59YWQIt-FyvO-8J70m4xI)

6. Click **Create load balancer**

## 7. Listener Rules for Path-Based Routing

### Listener Rules for Path-Based Routing

Once the Load balancer is created, click the HTTPS listener on the ALB **Listeners** tab and add rules to the **HTTPS:443** listener.
 
 ![Picture11.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzMxIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.JIKKCYpXowaEqjEdHFsu96cuiDOF0LMRsg_PGuSDCY8)
 
 ![Picture12.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzMyIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.XL-dKETRQ2bLDuLVI7Fq2HUCJoRiyp6WpoqqiDIPtuA)

Add a rule for each of the Bold BI services. Each rule has:
- **Condition 1:** Host header = `<application-domain>`
- **Condition 2:** Path pattern = `/<service-path>/*`
- **Action:** Forward to target group

### Listener-rule reference table:


| Service | Target Group | Host header | Path pattern | Priority |
|---------|--------------|-------------|--------------|----------|
| id-web | tg-id-web | <application-domain> | /* | 11 |
| id-ums | tg-id-ums | <application-domain> | /ums/* | 10 |
| id-api | tg-id-api | <application-domain> | /api/* | 9 |
| bi-web | tg-bi-web | <application-domain> | /bi/* | 8 |
| bi-jobs | tg-bi-jobs | <application-domain> | /bi/jobs/* | 7 |
| bi-api | tg-bi-api | <application-domain> | /bi/api/* | 6 |
| bold-etl | tg-bold-etl | <application-domain> | /etlservice/* | 5 |
| bi-designer | tg-bi-designer | <application-domain> | /bi/designer/* | 4 |
| bi-ai | tg-bi-ai | <application-domain> | /aiservice/* | 3 |
| bi-mcp | tg-bi-mcp | <application-domain> | /mcp/* | 2 |

⚠️ **Lower priority values are evaluated first.** The listener's default action forwards to the IDP Web target group; it does not need a non-default priority.

###### Example for IDP Web:
 
 ![Picture13.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzMzIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.dddqz5UfCfUZhtp02V5hOODZoEl5H0kW4XHnHPjaMwo)
 
 ![Picture14.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzM0Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.632xslS9byL6FC8p-PGWeKys5F96NoW3H2XvZbzGoNQ)
 
 ![Picture15.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzM1Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.DZZJloEAa-Fi_6r3boupClSEKseByMj593uwx6zQTsg)

## 8. ECS Service Creation

### ECS Service Creation

⚠️ **RECOMMENDED DEPLOYMENT ORDER:** 
id-ums (first deployed, must be RUNNING) → id-api → id-web → bi-dataservice → bi-web → bi-api → bi-jobs → bold-etl → bi-ai → bi-mcp

**The id-ums service owns the schema + the initial administrator account.** Any service that boots before id-ums is RUNNING will fail health checks at startup.

From the cluster → **Services** tab → click **Create**. Pick the right task definition family for this service.
 
 ![Picture16.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzM2Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.BFlIzNd6ehCasD_DLYKITi0cOF_DtiBLuloXRIV_98U)

### 8.1 Environment

1. **Compute options** = Launch type
2. **Launch type** = Fargate
3. Open **Troubleshooting configuration** - recommended → tick **Turn on ECS Exec**

 ![Picture17.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzM3Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.ntWLFZB83orT7lgfmOfJiXuvrsg7JLPErACub4tnRBI)

### 8.2 Tasks and health

⚠️ **Desired tasks** = 1, **Health check grace period** = 120 s
 
 ![Picture39.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Ijc0NTIzIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.To3sThElejm7CB1DM7dthg1rKXqb4bNQy9_BLCgXZ_s)

### 8.3 Networking

- **VPC** — same as the cluster
- **Subnets** — private subnets only
- **Security group** — same SG as the cluster
- **Public IP** = Turned off
 
 ![Picture19.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzM5Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.TC6VO0yg2WwSFGdGqLVgrnFedUzhgqM7tEKoMICxSmc)

### 8.4 Service Connect (shared namespace)

1. Tick **Use Service Connect** → **Configuration** = Client and server → use the same namespace for every service
2. If namespace is not available, **Create namespace:**
   - Provide the load balancer name
   - **Instance discovery** – API calls and DNS queries in VPCs
   - Select the same VPC as the cluster uses
   - Click **Create Namespace**
3. Once namespace is created, choose it in the service connect
 
 ![Picture20.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQwIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.uWmo0cxkZ-UYv758Yh835mvUvSL1WWaedHk1a2zP8iM)


### 8.5 Add Port Mappings and Applications

Use these Service Connect values verbatim:

| Port alias | Discovery | DNS | Port |
|-----------|-----------|-----|------|
| idp-web | id-web-service | id-web-service | 6000 |
| idp-api | id-api-service | id-api-service | 6001 |
| idp-ums | id-ums-service | id-ums-service | 6002 |
| bi-web | bi-web-service | bi-web-service | 6004 |
| bi-api | bi-api-service | bi-api-service | 6005 |
| bi-jobs | bi-jobs-service | bi-jobs-service | 6006 |
| bi-dataservice | bi-dataservice-service | bi-dataservice-service | 6007 |
| bold-etl | bold-etl-service | bold-etl-service | 6009 |
| bi-ai | bold-ai-service | bold-ai-service | 6010 |
| bi-mcp | bold-mcp-service | bold-mcp-service | 6011 |

⚠️ **Same Service Connect namespace for every service**
⚠️ **Set per-request timeout as 240 s for every services**

 
 ![Picture21.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQxIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.keFQTGMuf-iJs94UMPzdSgEQsJUgF4_L618kkAmmM6s)

### 8.6 Load balancer integration

In the **Load balancer** section:
- Use **existing ALB**
- Use **existing HTTPS listener**
- Use **matching listener row**
- Click **Create**

 
 ![Picture22.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQyIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.vlAjccj1l9WbZDzN4EizLom7OPVYQN7sQXvPQNG9rnc)

Once all the details are given, click **Create** option to create the service. Repeat for all services.

## 9. Auto-Deployment and Verification

⚠️ **After all services are RUNNING**, browse to `https://<application-domain>/`. The Bold BI auto-deployment wizard loads automatically on a fresh tenant.

Log in with the admin credentials (user-email + user-password).

### 9.1 Post-deployment checklist

- ☐ All ECS services in RUNNING state
- ☐ All target groups report at least one healthy target
- ☐ Service Connect DNS resolves between every pair of services
- ☐ S3 bucket shows files in /app_data/
- ☐ No errors in CloudWatch Log Group /aws/ecs/boldbi
- ☐ Map the ALB DNS to `<application-domain>` in Route 53

## 10. Auto Scaling Configuration

### Auto Scaling Configuration - AWS ECS Fargate with Application Load Balancer

**Overview**

This section provides step-by-step instructions to enable auto-scaling for Bold BI services on AWS ECS. Once configured, services will automatically scale based on CPU and Memory utilization metrics.

⚠️ **Important Note:** Once CPU or Memory reaches above 80%, it takes approximately 3 minutes for a new task to be completely launched and ready to serve traffic.

### Step 1: Update Service with Monitoring Configuration

Navigate to the respective service and configure monitoring metrics:

1. Go to **AWS ECS > Clusters > Select your cluster > Services > Select the Bold BI service**
2. Click on **'Update service'** button
3. Navigate to **'Monitoring configuration'** section
4. Add metric configuration for **CPU Utilization**
5. Add metric configuration for **Memory Utilization**
6. Set **Resolution** as **20 seconds** (for faster scaling detection)
7. Click **'Update service'** to save changes
 
 ![Picture31.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQzIiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.aHDczIh37n3CbVCcTM76FUH2MqGkP0MvvGfXhGbKSNY)

### Step 2: Configure Service Auto Scaling Task Counts

Set the minimum and maximum number of tasks for auto-scaling:

1. Navigate to: **ECS > Services > Select service > Service Auto Scaling**
2. Click **'Set the number of tasks'** button
3. Enable: **'Use service auto scaling to adjust your service's desired task count'**
4. Set **Minimum number of tasks: 1**
5. Set **Maximum number of tasks: 20** (based on expected load)
6. Click **'Save'** button to apply changes
 
 ![Picture32.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ0Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.WwGDcmCDoPtnMM6vpbkgpfh2A3fhuVNXATxFYGb83gE)
 
 ![Picture33.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ1Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.xSnacIv_OeVHphvG4AvaZZ1s0oThWfPirKolMxdUDkk)

### Step 3: Create New Scaling Policies

Navigate to the scaling policies section:

1. Go to the service's **Auto scaling tab > Scaling policies** section
2. Click **'Create scaling policy'** button
 
 ![Picture34.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ2Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.l213iafy8aVX1TtS4y6xtEfDLvUj54Z3Lm423OxKZ5k)

### Step 4: Create CPU Utilization Scaling Policy

Configure Target Tracking policy for CPU metrics:

1. Choose **'Target Tracking'** as the scaling policy type
2. **Policy name:** Enter a descriptive name (e.g., 'service-name-cpu-scaling')
3. **Metric type:** Select **'ECSServiceAverageCPUUtilizationHighResolution'**
4. **Target utilization:** Enter **80** (percent)
5. Expand **'Additional settings'** section
6. **Scale-in cooldown period:** Set to **60 seconds**
7. **Scale-out cooldown period:** Set to **300 seconds** (5 minutes)
8. Click **'Create scaling policy'** button to save
 
 ![Picture35.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ3Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.foorr5_7q_KPsBh9xDIoy43RF8dr-f4BCv94CbxLQps)

### Step 5: Create Memory Utilization Scaling Policy

Configure Target Tracking policy for Memory metrics:

1. Click **'Create scaling policy'** button again
2. Choose **'Target Tracking'** as the scaling policy type
3. **Policy name:** Enter a descriptive name (e.g., 'service-name-memory-scaling')
4. **Metric type:** Select **'ECSServiceAverageMemoryUtilizationHighResolution'**
5. **Target utilization:** Enter **80** (percent)
6. Expand **'Additional settings'** section
7. **Scale-in cooldown period:** Set to **60 seconds**
8. **Scale-out cooldown period:** Set to **300 seconds** (5 minutes)
9. Click **'Create scaling policy'** button to save
 
 ![Picture36.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ4Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.s6UDrQXOlCJkQdjRE22QqRy-e7c9C6JASKENisf2HtU)

### Step 6: Verification

Verify that both policies are now active:

1. Navigate to the service's **Auto scaling tab > Scaling policies** section
2. You should see both policies listed with **'Active'** status
3. Verify CPU utilization policy shows **80%** target
4. Verify Memory utilization policy shows **80%** target
5. Both policies are now active and will trigger scaling events
 
 ![Picture37.jpg](https://support.boldbi.com/kb/agent/attachment/article/25490/inline?token=eyJhbGciOiJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNobWFjLXNoYTI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjczNzQ5Iiwib3JnaWQiOiIzIiwiaXNzIjoic3VwcG9ydC5ib2xkYmkuY29tIn0.H8mHiiy50FVV_J5fgpgczHstPjodRP4kswFr6wdGn7g)

### Scaling Timeline & Important Notes

**When CPU or Memory exceeds 80%**, AWS automatically launches new tasks. Timeline breakdown:

| Phase | Duration | Activity |
|-------|----------|----------|
| 0-1 minute | 1 min | Alarm threshold exceeded, scaling initiated |
| 1-3 minutes | 2 min | AWS ECS service initiates task launch request; Fargate provisions container, pulls image from ECR |
| During launch | - | Application (Bold BI) initializes and starts |
| Health checks | - | Health checks pass, task added to load balancer |
| Traffic serving | - | Task receives traffic from load balancer |
| **Total time** | **3 minutes** | From alarm trigger to production traffic |

### High Availability Configuration

**Setting Minimum=2 ensures redundancy:**

- Service always runs minimum 2 tasks
- If one task fails, other task continues serving traffic
- Replacement task automatically launched by AWS
- Result: Zero downtime during task failure

⚠️ **Single task (Min=1) = Single point of failure!**
