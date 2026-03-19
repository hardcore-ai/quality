# Tech Stack Decisions — Unit 1: MVP Core

## Decision Log

All decisions confirmed from Requirements Analysis and NFR clarification. This document records the rationale and constraints for each technology choice.

---

## Application Layer

### TS-01: Next.js 16 (App Router)

| Attribute | Value |
|---|---|
| **Technology** | Next.js 16 with App Router |
| **Language** | TypeScript 5.x |
| **Strictness** | `strict: true` |
| **Rendering** | React Server Components for dashboard pages; Client Components for interactive forms |
| **API Routes** | App Router route handlers (`app/api/*/route.ts`) |
| **Rationale** | Single deployable unit for frontend + API. RSC reduces client bundle size. App Router enables streaming and server-side data fetching. |
| **Constraints** | Deployed as Docker container — no Vercel-specific features (no Edge Runtime, no ISR) |

### TS-02: Tailwind CSS + shadcn/ui

| Attribute | Value |
|---|---|
| **CSS Framework** | Tailwind CSS v3 |
| **Component Library** | shadcn/ui (Radix UI primitives + Tailwind) |
| **Icons** | lucide-react |
| **Rationale** | Utility-first CSS for rapid development. shadcn/ui provides accessible, unstyled components that are fully customizable. Components are copied into the project (not a dependency) — full control. |
| **Components used** | Button, Card, Table, Dialog, Input, Textarea, Select, Badge, Checkbox, Label, ScrollArea, Skeleton, Separator |

### TS-03: grammY (Telegram Bot Framework)

| Attribute | Value |
|---|---|
| **Technology** | grammY v1.x |
| **Integration** | Webhook via Next.js API route (`POST /api/telegram`) |
| **Session** | No grammY session middleware — session state managed in DynamoDB directly |
| **Rationale** | TypeScript-native, webhook-first design, well-maintained. Webhook integration fits the Next.js API route model. |
| **Constraints** | Webhook must respond within 5s — async pattern required (NFR-AVAIL-04) |

---

## AI / LLM Layer

### TS-04: OpenAI GPT-4o

| Attribute | Value |
|---|---|
| **Model** | `gpt-4o` (primary), `gpt-4.1` (fallback if available) |
| **SDK** | `openai` npm package (official) |
| **Usage** | Chat completions: screening conversation, follow-up generation, scoring, summary |
| **Response format** | Structured JSON output (`response_format: { type: 'json_object' }`) for scoring calls |
| **Context window** | 128K tokens — full conversation history injected (no trimming for MVP) |
| **Latency budget** | ≤ 7s per call (NFR-PERF-01) |
| **Retry policy** | 3× exponential backoff: 1s, 2s, 4s (NFR-AVAIL-02) |
| **Timeout** | 7s hard timeout per call |
| **Rationale** | Best reasoning capability for dynamic follow-up generation and nuanced evaluation. JSON mode ensures structured scoring output. |

---

## Data Layer

### TS-05: AWS DynamoDB

| Attribute | Value |
|---|---|
| **Technology** | AWS DynamoDB |
| **SDK** | `@aws-sdk/client-dynamodb` + `@aws-sdk/lib-dynamodb` (DocumentClient) |
| **Capacity mode** | On-demand (pay-per-request) |
| **Tables** | 6 tables (conversations, campaigns, candidates, evaluations, audit_events, consent_records) |
| **Key pattern** | `tenantId#entityId` composite partition key |
| **GSIs** | 3 GSIs: candidates by telegramUserId, candidates by campaignId+state, conversations by campaignId+state |
| **Rationale** | Managed NoSQL, serverless scaling, no connection pooling issues with Lambda/ECS. Document model fits conversation transcripts. On-demand mode eliminates capacity planning for MVP. |
| **Constraints** | 400KB item size limit — messages embedded in conversation item (acceptable for 15-25 min sessions) |

---

## Infrastructure Layer

### TS-06: AWS ECS Fargate

| Attribute | Value |
|---|---|
| **Technology** | AWS ECS with Fargate launch type |
| **Task count** | 1 task (MVP) |
| **CPU / Memory** | 512 vCPU / 1024 MB (adjustable via Terraform variables) |
| **Container** | Multi-stage Docker build (Node.js 20 Alpine) |
| **Port** | 3000 (Next.js default) |
| **Health check** | `GET /api/health` |
| **Rationale** | Serverless containers — no EC2 management. Fargate handles patching and scaling. Single task acceptable for MVP volume. |

### TS-07: AWS Application Load Balancer

| Attribute | Value |
|---|---|
| **Technology** | AWS ALB |
| **TLS** | ACM certificate (HTTPS termination at ALB) |
| **Target group** | ECS service (port 3000) |
| **Health check** | `GET /api/health`, interval 30s, threshold 2 |
| **Rationale** | Required for ECS service exposure. Handles TLS termination, health checks, and future multi-task load distribution. |

### TS-08: AWS Cognito

| Attribute | Value |
|---|---|
| **Technology** | AWS Cognito User Pool + App Client |
| **Integration** | NextAuth.js Cognito provider |
| **Custom attributes** | `custom:tenantId` (required for all users) |
| **Hosted UI** | Enabled (Cognito hosted login page) |
| **Token validation** | NextAuth.js handles JWKS validation automatically |
| **Rationale** | Managed auth service — no custom auth implementation. Cognito handles password policies, MFA (future), and token lifecycle. |

### TS-09: AWS Secrets Manager

| Attribute | Value |
|---|---|
| **Technology** | AWS Secrets Manager |
| **Secrets** | OpenAI API key, Telegram bot token, Telegram webhook secret, Cognito client secret |
| **Access** | ECS task IAM role with `secretsmanager:GetSecretValue` on specific ARNs |
| **Injection** | ECS task definition `secrets` field → environment variables at startup |
| **Rationale** | Centralized secret management, rotation support, audit trail via CloudTrail. |

### TS-10: Terraform

| Attribute | Value |
|---|---|
| **Technology** | Terraform (HashiCorp) |
| **Modules** | vpc, ecs, alb, dynamodb, cognito, ecr, cloudwatch, secrets |
| **Environments** | dev, staging, prod (separate state files) |
| **State backend** | S3 + DynamoDB state locking |
| **Provider** | `hashicorp/aws` ~> 5.0 |
| **Rationale** | IaC for reproducible infrastructure. Module structure enables environment parity. S3 backend enables team collaboration. |

### TS-11: AWS ECR

| Attribute | Value |
|---|---|
| **Technology** | AWS Elastic Container Registry |
| **Image lifecycle** | Keep last 10 images (lifecycle policy via Terraform) |
| **Rationale** | Native AWS container registry. Integrated with ECS task definitions. No external registry dependency. |

---

## CI/CD Layer

### TS-12: GitHub Actions

| Attribute | Value |
|---|---|
| **Technology** | GitHub Actions |
| **Pipeline stages** | lint → type-check → unit tests → build Docker → push ECR → deploy ECS |
| **Triggers** | Push to `main` (deploy to prod), push to `develop` (deploy to staging), PR (lint + test only) |
| **Secrets** | AWS credentials via OIDC (no long-lived access keys) |
| **Rationale** | Native GitHub integration. OIDC eliminates static AWS credentials in CI. |

---

## Testing Layer

### TS-13: Vitest

| Attribute | Value |
|---|---|
| **Technology** | Vitest |
| **Scope** | Unit tests — domain rules, use cases (mocked infrastructure) |
| **Config** | `vitest.config.ts` with TypeScript support |
| **Rationale** | Fast, ESM-native, compatible with TypeScript strict mode. Vite-based — no separate transpilation step. |

### TS-14: Playwright

| Attribute | Value |
|---|---|
| **Technology** | Playwright v1.x |
| **Scope** | API integration tests + E2E browser tests |
| **E2E scenarios** | Gherkin `.feature` files from user story acceptance criteria |
| **API tests** | Direct HTTP calls to Next.js API routes (test environment) |
| **Browser** | Chromium (primary), Firefox (secondary) |
| **Test DB** | DynamoDB Local for integration tests |
| **Rationale** | Single framework for both API and browser testing. Gherkin integration enables direct traceability from user stories to test cases. |

---

## Dependency Summary

```json
{
  "dependencies": {
    "next": "^16.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "grammy": "^1.x",
    "openai": "^4.x",
    "@aws-sdk/client-dynamodb": "^3.x",
    "@aws-sdk/lib-dynamodb": "^3.x",
    "next-auth": "^4.x",
    "tailwindcss": "^3.x",
    "lucide-react": "^0.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "vitest": "^1.x",
    "@playwright/test": "^1.x",
    "@cucumber/cucumber": "^10.x",
    "eslint": "^8.x",
    "eslint-config-next": "^16.x"
  }
}
```

**shadcn/ui**: Components copied into `src/components/ui/` (not a package dependency — installed via `npx shadcn-ui@latest add`).
