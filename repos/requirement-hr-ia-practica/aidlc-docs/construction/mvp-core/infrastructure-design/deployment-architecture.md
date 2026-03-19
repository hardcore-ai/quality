# Deployment Architecture — Unit 1: MVP Core

## Architecture Overview

```
                         INTERNET
                            |
                    +-------+-------+
                    |  Telegram API  |  (outbound: bot sends messages)
                    |  OpenAI API    |  (outbound: LLM calls)
                    +-------+-------+
                            |
              +-------------+-------------+
              |                           |
    Candidates (Telegram)        Recruiters (Browser)
              |                           |
              v                           v
    +--------------------+     +--------------------+
    |  Telegram Servers  |     |  Browser / HTTPS   |
    |  POST /api/telegram|     |  GET/POST /api/*   |
    +--------------------+     +--------------------+
              |                           |
              +-------------+-------------+
                            |
                            v
              +---------------------------+
              |  AWS ALB                  |
              |  entrievista-alb-{env}    |
              |  HTTP :80 → HTTPS :443    |
              |  Health: /api/health      |
              +---------------------------+
                            |
              +-------------+-------------+
              |  public-1 (us-east-1a)   |  public-2 (us-east-1b)
              +---------------------------+
                            |
                            v
              +---------------------------+
              |  ECS Fargate Task         |
              |  1 vCPU / 2 GB            |
              |  Next.js :3000            |
              |  public subnet (MVP)      |
              +---------------------------+
                    |           |
          +---------+           +---------+
          |                               |
          v                               v
+------------------+           +------------------+
|  DynamoDB        |           |  AWS Secrets     |
|  (via VPC        |           |  Manager         |
|  Gateway         |           |  (via VPC        |
|  Endpoint)       |           |  Interface EP)   |
+------------------+           +------------------+
          |
          v
+------------------+
|  CloudWatch Logs |
|  (via VPC        |
|  Interface EP)   |
+------------------+
```

---

## Terraform Module Structure

```
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf          # VPC, subnets, IGW, route tables
│   │   ├── endpoints.tf     # VPC Gateway + Interface endpoints
│   │   ├── security_groups.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── ecr/
│   │   ├── main.tf          # ECR repository + lifecycle policy
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── ecs/
│   │   ├── cluster.tf       # ECS cluster
│   │   ├── task_definition.tf
│   │   ├── service.tf       # ECS service + rolling update config
│   │   ├── iam.tf           # Task execution role + task role
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── alb/
│   │   ├── main.tf          # ALB, listeners, target group
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── dynamodb/
│   │   ├── main.tf          # All 7 tables with GSIs + TTL
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── cognito/
│   │   ├── main.tf          # User Pool + App Client + Hosted UI domain
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── secrets/
│   │   ├── main.tf          # Secrets Manager secrets (empty values — filled manually)
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── cloudwatch/
│       ├── main.tf          # Log groups + retention policies
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/
│   ├── dev/
│   │   ├── main.tf          # Module composition for dev
│   │   ├── variables.tf
│   │   ├── terraform.tfvars # Dev-specific values (non-secret)
│   │   └── backend.tf       # S3 state backend config for dev account
│   │
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   │
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
│
├── main.tf                  # Root module (optional — environments are self-contained)
├── variables.tf             # Shared variable definitions
└── outputs.tf
```

---

## CI/CD Pipeline

### GitHub Actions Workflows

```
.github/
└── workflows/
    ├── ci.yml               # PR checks: lint + type-check + unit tests
    ├── deploy-staging.yml   # Push to develop → deploy to staging
    └── deploy-prod.yml      # Push to main → deploy to prod
```

### deploy-prod.yml (representative)

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: entrievista/prod/app
  ECS_CLUSTER: entrievista-prod
  ECS_SERVICE: entrievista-app-prod
  CONTAINER_NAME: app

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                       -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Run unit tests
        run: npm ci && npm run test:unit

      - name: Update ECS service
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment
```

### GitHub Secrets Required (per environment)

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID_DEV` | IAM user access key for dev account |
| `AWS_SECRET_ACCESS_KEY_DEV` | IAM user secret key for dev account |
| `AWS_ACCESS_KEY_ID_STAGING` | IAM user access key for staging account |
| `AWS_SECRET_ACCESS_KEY_STAGING` | IAM user secret key for staging account |
| `AWS_ACCESS_KEY_ID_PROD` | IAM user access key for prod account |
| `AWS_SECRET_ACCESS_KEY_PROD` | IAM user secret key for prod account |

---

## Dockerfile

```dockerfile
# Multi-stage build for Next.js

# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Runner
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built artifacts
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000

CMD ["node", "server.js"]
```

**next.config.js** must include `output: 'standalone'` for the multi-stage build to work.

---

## Environment Variables Reference

| Variable | Source | Required | Description |
|---|---|---|---|
| `NODE_ENV` | ECS task def | Yes | `production` |
| `PORT` | ECS task def | Yes | `3000` |
| `AWS_REGION` | ECS task def | Yes | `us-east-1` |
| `DYNAMODB_TABLE_PREFIX` | ECS task def | Yes | `entrievista-{env}` |
| `BOT_USERNAME` | ECS task def | Yes | Telegram bot username |
| `APP_VERSION` | ECS task def | Yes | Git SHA |
| `LOG_LEVEL` | ECS task def | Yes | `info` (prod), `debug` (dev) |
| `NEXTAUTH_URL` | ECS task def | Yes | ALB DNS URL |
| `COGNITO_ISSUER` | ECS task def | Yes | Cognito User Pool URL |
| `COGNITO_CLIENT_ID` | ECS task def | Yes | Cognito App Client ID |
| `OPENAI_API_KEY` | Secrets Manager | Yes | OpenAI API key |
| `TELEGRAM_BOT_TOKEN` | Secrets Manager | Yes | Telegram bot token |
| `TELEGRAM_WEBHOOK_SECRET` | Secrets Manager | Yes | Webhook validation secret |
| `NEXTAUTH_SECRET` | Secrets Manager | Yes | NextAuth.js signing secret |
| `COGNITO_CLIENT_SECRET` | Secrets Manager | Yes | Cognito App Client secret |

---

## Infrastructure Provisioning Order

Terraform modules must be applied in dependency order:

```
1. ecr/          → ECR repository (needed before first Docker push)
2. vpc/          → VPC, subnets, security groups, VPC endpoints
3. dynamodb/     → DynamoDB tables
4. cognito/      → Cognito User Pool + App Client
5. secrets/      → Secrets Manager secrets (empty — fill values manually after)
6. cloudwatch/   → Log groups
7. alb/          → ALB, target group, listeners (needs VPC)
8. ecs/          → ECS cluster, task definition, service (needs ALB, ECR, IAM)
```

**First-time setup steps**:
1. `terraform apply` modules 1-8 in order
2. Manually set secret values in Secrets Manager console
3. Register Telegram webhook: `POST https://api.telegram.org/bot{TOKEN}/setWebhook?url=https://{ALB_DNS}/api/telegram&secret_token={WEBHOOK_SECRET}`
4. Create first Cognito user in console (or via AWS CLI)
5. Push first Docker image via GitHub Actions

---

## Cost Estimate (MVP — prod account)

| Service | Config | Est. Monthly Cost |
|---|---|---|
| ECS Fargate | 1 task, 1 vCPU / 2 GB, 24/7 | ~$30 |
| ALB | 1 ALB, low traffic | ~$18 |
| DynamoDB | On-demand, MVP volume | ~$1-5 |
| ECR | ~1 GB storage | ~$0.10 |
| CloudWatch Logs | 30-day retention, low volume | ~$1-3 |
| Secrets Manager | 5 secrets | ~$2.50 |
| VPC Endpoints | 3 interface endpoints | ~$21 |
| Cognito | < 50,000 MAU free tier | $0 |
| **Total** | | **~$75-80/month** |

Note: VPC Interface Endpoints ($7/each) are the second-largest cost. If budget is tight, remove Secrets Manager and ECR endpoints — use NAT Gateway or public subnet access for those services only.
