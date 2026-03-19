# Infrastructure Design Plan — Unit 1: MVP Core

## Unit Context
- **Cloud**: AWS
- **IaC**: Terraform (S3 backend + DynamoDB state locking)
- **Compute**: ECS Fargate, 1 task MVP
- **Database**: DynamoDB on-demand, 6 tables + processed_updates
- **Auth**: Cognito User Pool + App Client
- **Networking**: VPC + ALB + ACM
- **Registry**: ECR
- **Secrets**: AWS Secrets Manager
- **Observability**: CloudWatch Logs

---

## Execution Checklist

- [x] Step 1: Analyze design artifacts
- [x] Step 2: Create infrastructure design plan (this file)
- [x] Step 3: Generate clarifying questions (below)
- [x] Step 4: Store plan
- [x] Step 5: Collect and analyze answers
- [x] Step 6: Generate infrastructure design artifacts
  - [x] infrastructure-design.md
  - [x] deployment-architecture.md
- [ ] Step 7: Present completion message
- [ ] Step 8: Wait for explicit approval
- [ ] Step 9: Record approval and update progress

---

## Clarifying Questions

### Section A: Networking

**A1. VPC subnet layout**

What VPC subnet layout should be used?

A) 2 AZs: 2 public subnets (ALB) + 2 private subnets (ECS + DynamoDB VPC endpoint) — standard HA layout
B) 1 AZ only: 1 public + 1 private — minimum cost for MVP
C) 2 AZs: public subnets only (ALB + ECS in public) — simpler, less secure
D) 3 AZs: full HA layout

[Answer]: A

---

**A2. DynamoDB access from ECS**

How should ECS tasks reach DynamoDB?

A) VPC Endpoint for DynamoDB (Gateway type — free, traffic stays in AWS network)
B) Public internet via NAT Gateway (standard but adds cost + latency)
C) No VPC endpoint — use NAT Gateway for all AWS API calls
D) VPC Endpoint for DynamoDB + VPC Endpoint for Secrets Manager

[Answer]: A

---

**A3. Custom domain**

Should the ALB be fronted by a custom domain (e.g., `app.entrievista.ai`)?

A) Yes — Route 53 hosted zone + ACM certificate + ALB alias record
B) No — use ALB DNS name directly for MVP (add domain post-MVP)
C) Yes — but using an existing Route 53 hosted zone (domain already registered)
D) Yes — CloudFront in front of ALB for CDN + custom domain

[Answer]: B

---

### Section B: ECS & Container

**B1. Container resource sizing**

What CPU/memory should the Fargate task have for MVP?

A) 0.5 vCPU / 1 GB — minimum, suitable for low traffic MVP
B) 1 vCPU / 2 GB — comfortable headroom for LLM processing
C) 0.25 vCPU / 512 MB — absolute minimum cost
D) 1 vCPU / 4 GB — generous for concurrent LLM calls

[Answer]: B

---

**B2. Environment separation**

How should dev/staging/prod environments be separated in AWS?

A) Separate AWS accounts per environment (best practice, most isolation)
B) Same AWS account, separate Terraform workspaces with resource name prefixes (e.g., `entrievista-dev-*`, `entrievista-prod-*`)
C) Same AWS account, same resources — environment controlled by env vars only
D) Separate AWS accounts for prod only; dev + staging share an account

[Answer]: A

---

### Section C: CI/CD Pipeline

**C1. GitHub Actions → AWS authentication**

How should GitHub Actions authenticate to AWS for ECR push and ECS deploy?

A) OIDC (OpenID Connect) — no long-lived credentials, GitHub federated identity
B) IAM user with access keys stored as GitHub secrets
C) IAM role assumed via AWS CLI with stored credentials
D) AWS CodePipeline instead of GitHub Actions

[Answer]:  B

---

**C2. Deployment strategy**

When a new Docker image is pushed, how should ECS update the running task?

A) Rolling update — ECS replaces the task with the new image (default, zero-downtime with 1 task is not guaranteed)
B) Blue/green deployment via CodeDeploy — full zero-downtime (more complex)
C) Force new deployment — stop current task, start new one (brief downtime acceptable for MVP)
D) Rolling update with minimum healthy percent = 0 (allows brief downtime, simpler)

[Answer]: A

---

### Section D: Terraform Structure

**D1. Terraform state backend**

Where should Terraform state be stored?

A) S3 bucket + DynamoDB table for state locking (standard, recommended)
B) Terraform Cloud (managed state, free tier available)
C) Local state only (not suitable for team use)
D) S3 bucket only (no locking — acceptable for solo developer)

[Answer]: D

---
