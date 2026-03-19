# Infrastructure Design — Unit 1: MVP Core

## Decision Summary

| Area | Decision |
|---|---|
| Cloud provider | AWS |
| IaC | Terraform, S3 backend (no locking — solo developer) |
| Environments | Separate AWS accounts per environment (dev / staging / prod) |
| VPC layout | 2 AZs: 2 public subnets (ALB) + 2 private subnets (ECS) |
| DynamoDB access | VPC Gateway Endpoint (free, traffic stays in AWS network) |
| Custom domain | No — ALB DNS name directly for MVP |
| ECS task sizing | 1 vCPU / 2 GB memory (Fargate) |
| ECS task count | 1 task (MVP) |
| Deployment strategy | Rolling update (ECS default) |
| CI/CD auth | IAM user + access keys stored as GitHub secrets |
| Terraform state | S3 bucket only (no DynamoDB locking) |

---

## 1. AWS Account Strategy

Three separate AWS accounts — one per environment. This provides full blast-radius isolation: a misconfiguration or data issue in dev cannot affect prod.

```
AWS Organization (optional — can be standalone accounts)
├── entrievista-dev     (development account)
├── entrievista-staging (staging account)
└── entrievista-prod    (production account)
```

Each account has its own:
- Terraform state S3 bucket
- ECR repository
- All infrastructure resources (VPC, ECS, DynamoDB, Cognito, etc.)
- IAM user for GitHub Actions CI/CD

Terraform workspace per environment maps to its corresponding account via AWS provider configuration.

---

## 2. Networking

### 2.1 VPC Design

**CIDR**: `10.0.0.0/16` (65,536 addresses — ample for MVP and future growth)

**Subnets** (2 AZs — `us-east-1a` and `us-east-1b` as default, configurable):

| Subnet | AZ | CIDR | Purpose |
|---|---|---|---|
| `public-1` | us-east-1a | `10.0.1.0/24` | ALB |
| `public-2` | us-east-1b | `10.0.2.0/24` | ALB |
| `private-1` | us-east-1a | `10.0.10.0/24` | ECS tasks |
| `private-2` | us-east-1b | `10.0.11.0/24` | ECS tasks |

**Internet Gateway**: Attached to VPC, routes public subnets to internet.

**NAT Gateway**: NOT provisioned for MVP. ECS tasks in private subnets reach AWS services (DynamoDB, Secrets Manager, ECR, CloudWatch) exclusively via VPC Endpoints — no internet egress needed for core functionality.

**Note**: OpenAI API calls require internet egress. Two options:
- Option A: Move ECS to public subnet (simpler, less secure)
- Option B: Add NAT Gateway for internet egress from private subnet

**Decision**: ECS tasks run in **public subnets** for MVP to avoid NAT Gateway cost (~$32/month). Security Groups restrict inbound traffic to ALB only. This is acceptable for MVP; move to private + NAT post-MVP.

### 2.2 VPC Endpoints

| Endpoint | Type | Purpose |
|---|---|---|
| `com.amazonaws.{region}.dynamodb` | Gateway | DynamoDB access from ECS (free) |
| `com.amazonaws.{region}.ecr.api` | Interface | ECR image pull |
| `com.amazonaws.{region}.ecr.dkr` | Interface | ECR Docker registry |
| `com.amazonaws.{region}.logs` | Interface | CloudWatch Logs |
| `com.amazonaws.{region}.secretsmanager` | Interface | Secrets Manager |

Gateway endpoints (DynamoDB) are free. Interface endpoints have a small hourly cost (~$7/month each) — include ECR and CloudWatch as minimum; Secrets Manager optional (can use NAT for initial secret fetch).

### 2.3 Security Groups

**ALB Security Group** (`sg-alb`):
- Inbound: TCP 443 from `0.0.0.0/0` (HTTPS)
- Inbound: TCP 80 from `0.0.0.0/0` (HTTP → redirect to 443)
- Outbound: TCP 3000 to ECS Security Group

**ECS Security Group** (`sg-ecs`):
- Inbound: TCP 3000 from ALB Security Group only
- Outbound: TCP 443 to `0.0.0.0/0` (OpenAI API, Telegram API)
- Outbound: TCP 443 to VPC Endpoint Security Group (ECR, CloudWatch, Secrets Manager)

---

## 3. Compute — ECS Fargate

### 3.1 ECS Cluster

```
Cluster name: entrievista-{env}
Launch type: FARGATE
Container insights: disabled (MVP — CloudWatch basic metrics sufficient)
```

### 3.2 Task Definition

```
Family: entrievista-app
CPU: 1024 (1 vCPU)
Memory: 2048 MB (2 GB)
Network mode: awsvpc
Execution role: ecsTaskExecutionRole (ECR pull + CloudWatch logs)
Task role: entrievista-task-role (DynamoDB + Secrets Manager access)

Container:
  Name: app
  Image: {account_id}.dkr.ecr.{region}.amazonaws.com/entrievista:{tag}
  Port: 3000
  Essential: true
  
  Environment variables (non-secret):
    NODE_ENV: production
    PORT: 3000
    AWS_REGION: us-east-1
    DYNAMODB_TABLE_PREFIX: entrievista-prod
    BOT_USERNAME: {telegram_bot_username}
    APP_VERSION: {git_sha}
    LOG_LEVEL: info

  Secrets (from Secrets Manager):
    OPENAI_API_KEY: arn:aws:secretsmanager:{region}:{account}:secret:entrievista/openai-api-key
    TELEGRAM_BOT_TOKEN: arn:aws:secretsmanager:{region}:{account}:secret:entrievista/telegram-bot-token
    TELEGRAM_WEBHOOK_SECRET: arn:aws:secretsmanager:{region}:{account}:secret:entrievista/telegram-webhook-secret
    NEXTAUTH_SECRET: arn:aws:secretsmanager:{region}:{account}:secret:entrievista/nextauth-secret
    COGNITO_CLIENT_SECRET: arn:aws:secretsmanager:{region}:{account}:secret:entrievista/cognito-client-secret

  Log configuration:
    Driver: awslogs
    Options:
      awslogs-group: /ecs/entrievista-{env}
      awslogs-region: us-east-1
      awslogs-stream-prefix: app

  Health check:
    Command: ["CMD-SHELL", "curl -f http://localhost:3000/api/health || exit 1"]
    Interval: 30s
    Timeout: 5s
    Retries: 3
    Start period: 60s
```

### 3.3 ECS Service

```
Service name: entrievista-app-{env}
Desired count: 1
Launch type: FARGATE
Deployment: Rolling update
  Minimum healthy percent: 0   (allows brief downtime during deploy — acceptable MVP)
  Maximum percent: 200

Load balancer:
  Target group: entrievista-tg-{env}
  Container: app:3000

Subnets: public-1, public-2 (MVP — avoids NAT Gateway cost)
Security group: sg-ecs
Assign public IP: true (required for public subnet ECS tasks)
```

---

## 4. Load Balancer — ALB

```
Name: entrievista-alb-{env}
Scheme: internet-facing
Subnets: public-1, public-2
Security group: sg-alb

Listeners:
  HTTP :80  → Redirect to HTTPS :443
  HTTPS :443 → Forward to target group entrievista-tg-{env}
  
  Certificate: ACM self-signed or Let's Encrypt (no custom domain for MVP)
  Note: For MVP without custom domain, use HTTP only or self-signed cert.
        ALB DNS: entrievista-alb-{env}.{region}.elb.amazonaws.com

Target Group:
  Name: entrievista-tg-{env}
  Protocol: HTTP
  Port: 3000
  Target type: ip (required for Fargate awsvpc)
  Health check:
    Path: /api/health
    Interval: 30s
    Healthy threshold: 2
    Unhealthy threshold: 3
```

**MVP simplification**: Since no custom domain is used (A3), HTTPS requires either a self-signed certificate or HTTP-only. Recommendation: use HTTP for MVP internal testing, add ACM + domain for staging/prod.

---

## 5. Database — DynamoDB

### 5.1 Tables

All tables use on-demand capacity. Naming convention: `{prefix}-{table}` where prefix = `entrievista-{env}`.

| Table | Partition Key | Sort Key | GSIs |
|---|---|---|---|
| `conversations` | `pk` (tenantId#convId) | `createdAt` | GSI1: campaignId + state |
| `campaigns` | `pk` (tenantId#campaignId) | `createdAt` | — |
| `candidates` | `pk` (tenantId#candidateId) | `createdAt` | GSI1: telegramUserId+tenantId, GSI2: campaignId+state |
| `evaluations` | `pk` (tenantId#evalId) | `conversationId` | — |
| `audit_events` | `pk` (tenantId#entityId) | `timestamp` | — |
| `consent_records` | `pk` (tenantId#candidateId) | `timestamp` | — |
| `processed_updates` | `updateId` | — | TTL: `expiresAt` |

### 5.2 DynamoDB Configuration

```
Billing mode: PAY_PER_REQUEST (on-demand)
Encryption: AWS managed key (SSE enabled by default)
Point-in-time recovery: enabled for prod, disabled for dev/staging
Deletion protection: enabled for prod, disabled for dev/staging
TTL attribute: expiresAt (on processed_updates table only)
```

---

## 6. Authentication — Cognito

```
User Pool:
  Name: entrievista-users-{env}
  Username: email
  Password policy: min 8 chars, uppercase, lowercase, number
  MFA: optional (off for MVP)
  Custom attributes: custom:tenantId (required, immutable)

App Client:
  Name: entrievista-dashboard-{env}
  Auth flows: ALLOW_USER_PASSWORD_AUTH, ALLOW_REFRESH_TOKEN_AUTH
  Callback URLs: http://localhost:3000/api/auth/callback/cognito (dev)
                 https://{alb-dns}/api/auth/callback/cognito (prod)
  Logout URLs: http://localhost:3000 (dev)
               https://{alb-dns} (prod)
  OAuth scopes: openid, email, profile

Hosted UI: enabled (Cognito managed login page)
Domain: entrievista-{env}.auth.{region}.amazoncognito.com
```

---

## 7. Container Registry — ECR

```
Repository name: entrievista/{env}/app
Image tag mutability: MUTABLE (allows :latest tag)
Scan on push: enabled (basic vulnerability scanning)

Lifecycle policy:
  Rule 1: Keep last 10 tagged images
  Rule 2: Delete untagged images older than 1 day
```

---

## 8. Secrets Manager

```
Secrets (per environment, prefixed with env):
  entrievista/{env}/openai-api-key
  entrievista/{env}/telegram-bot-token
  entrievista/{env}/telegram-webhook-secret
  entrievista/{env}/nextauth-secret
  entrievista/{env}/cognito-client-secret

Rotation: manual for MVP (no automatic rotation configured)
KMS: AWS managed key
```

---

## 9. Observability — CloudWatch

```
Log Groups:
  /ecs/entrievista-{env}        (application logs — retention: 30 days prod, 7 days dev)

Metrics:
  ECS service metrics (CPU, memory, task count) — automatic via ECS
  ALB metrics (request count, latency, 5xx rate) — automatic via ALB

Alarms: none for MVP (manual log review — NFR-OBS-02)
```

---

## 10. IAM Roles & Policies

### 10.1 ECS Task Execution Role (`ecsTaskExecutionRole`)

Standard AWS managed role. Grants ECS agent permission to:
- Pull images from ECR
- Write logs to CloudWatch
- Fetch secrets from Secrets Manager (for task definition secrets)

### 10.2 ECS Task Role (`entrievista-task-role`)

Custom role attached to the running container. Least-privilege:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem", "dynamodb:PutItem",
        "dynamodb:UpdateItem", "dynamodb:Query", "dynamodb:BatchWriteItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-conversations",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-conversations/index/*",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-campaigns",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-candidates",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-candidates/index/*",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-evaluations",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-consent_records",
        "arn:aws:dynamodb:*:*:table/entrievista-{env}-processed_updates"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:*:*:table/entrievista-{env}-audit_events"
    },
    {
      "Effect": "Deny",
      "Action": ["dynamodb:UpdateItem", "dynamodb:DeleteItem"],
      "Resource": "arn:aws:dynamodb:*:*:table/entrievista-{env}-audit_events"
    }
  ]
}
```

### 10.3 GitHub Actions IAM User (`github-actions-{env}`)

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": [
        "arn:aws:iam::*:role/ecsTaskExecutionRole",
        "arn:aws:iam::*:role/entrievista-task-role"
      ]
    }
  ]
}
```

Access keys stored as GitHub repository secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.

---

## 11. Terraform State Backend

```
Backend: S3 (no DynamoDB locking — solo developer)

S3 bucket: entrievista-terraform-state-{account_id}
  Versioning: enabled
  Encryption: AES-256
  Public access: blocked

State file paths:
  s3://entrievista-terraform-state-{account_id}/dev/terraform.tfstate
  s3://entrievista-terraform-state-{account_id}/staging/terraform.tfstate
  s3://entrievista-terraform-state-{account_id}/prod/terraform.tfstate
```

Note: Since each environment is a separate AWS account, each account has its own state bucket. No cross-account state sharing needed.
