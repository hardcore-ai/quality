# Code Generation Plan — Unit 1: MVP Core

## Unit Context

- **Unit**: `mvp-core`
- **Project type**: Greenfield — Next.js 16 modular monolith
- **Workspace root**: `/home/andrescaicedom/projects/30x/requirement-hr-ia-practica/webapp/`
- **Architecture**: Hexagonal + Clean Architecture + DDD
- **Stories**: 18 (US-1.1 to US-1.8, US-2.1 to US-2.3, US-3.1 to US-3.5, US-4.1, US-5.1, US-7.1, US-7.2)

## Stories Implemented by This Unit

| Story | Title | Status |
|---|---|---|
| US-1.1 | Bot Onboarding and AI Disclosure | [x] |
| US-1.2 | Consent Collection | [x] |
| US-1.3 | Basic Requirements Verification | [x] |
| US-1.4 | Competency-Based Screening Questions | [x] |
| US-1.5 | Anti-Hallucination and Out-of-Scope Handling | [x] |
| US-1.6 | Escalation to Human (Level 1-2) | [x] |
| US-1.8 | Screening Completion and Closing | [x] |
| US-2.1 | Rubric Creation (hardcoded templates) | [x] |
| US-2.2 | Real-Time Evaluation During Screening | [x] |
| US-2.3 | Executive Summary Generation | [x] |
| US-3.1 | Campaign Creation and Management | [x] |
| US-3.2 | HITL Review Queue | [x] |
| US-3.3 | Candidate Detail View and Decision | [x] |
| US-3.5 | Authentication and Access Control | [x] |
| US-4.1 | Immutable Audit Trail (basic) | [x] |
| US-5.1 | Candidate Profile and Lifecycle | [x] |
| US-7.1 | Multi-Tenant Data Isolation | [x] |
| US-7.2 | Bot Response Performance | [x] |

---

## Generation Steps

### BLOCK 1 — Project Foundation

- [x] **Step 1**: Initialize Next.js 16 project with TypeScript strict mode
  - `npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"`
  - Configure `tsconfig.json`: `strict: true`, `skipLibCheck: true`, path aliases
  - Configure `next.config.js`: `output: 'standalone'`
  - Install core dependencies: `grammy`, `openai`, `@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`, `next-auth`
  - Install dev dependencies: `vitest`, `@playwright/test`, `@cucumber/cucumber`, `eslint-plugin-import`
  - Configure ESLint with import boundary rules (Hexagonal Architecture enforcement)
  - Stories: foundation for all

- [x] **Step 2**: Setup shadcn/ui component library
  - `npx shadcn-ui@latest init`
  - Add components: Button, Card, Table, Input, Textarea, Select, Badge, Checkbox, Label, ScrollArea, Skeleton, Separator, Dialog
  - Stories: US-3.1, US-3.2, US-3.3, US-3.5

- [x] **Step 3**: Create shared cross-cutting modules
  - `src/shared/logging/logger.ts` — structured JSON logger (NC-02)
  - `src/shared/utils/retry.ts` — exponential backoff utility (NC-01)
  - `src/shared/types/common.ts` — shared primitive types (TenantId, EntityId, etc.)
  - `src/infrastructure/config/secrets.ts` — SecretsLoader with fail-fast (NC-05)
  - Stories: US-7.1, US-7.2

---

### BLOCK 2 — Domain Layer (Ports + Entities + Rules)

- [x] **Step 4**: Conversation domain
  - `src/domain/conversation/entities/Conversation.ts` — Conversation, Message, SessionState, EscalationRequest types
  - `src/domain/conversation/rules/ConversationRules.ts` — all BR-CONV-* rules
  - `src/domain/conversation/ports/IConversationRepository.ts` — repository interface
  - Stories: US-1.1, US-1.2, US-1.3, US-1.4, US-1.5, US-1.6, US-1.8

- [x] **Step 5**: Evaluation domain
  - `src/domain/evaluation/entities/Rubric.ts` — Rubric, Competency, CompetencyCriteria
  - `src/domain/evaluation/entities/ExecutiveSummary.ts` — ExecutiveSummary, CompetencyScore, Evidence, Recommendation
  - `src/domain/evaluation/rules/EvaluationRules.ts` — all BR-EVAL-* rules + hardcoded BPO/Tech templates
  - `src/domain/evaluation/ports/IRubricRepository.ts`
  - `src/domain/evaluation/ports/IEvaluationRepository.ts`
  - Stories: US-2.1, US-2.2, US-2.3

- [x] **Step 6**: Campaign domain
  - `src/domain/campaign/entities/Campaign.ts` — Campaign, CampaignStatus, BasicRequirement
  - `src/domain/campaign/rules/CampaignRules.ts` — all BR-CAMP-* rules
  - `src/domain/campaign/ports/ICampaignRepository.ts`
  - Stories: US-3.1, US-7.1

- [x] **Step 7**: Candidate domain
  - `src/domain/candidate/entities/Candidate.ts` — Candidate, CandidateState, ReviewDecision
  - `src/domain/candidate/rules/CandidateRules.ts` — all BR-CAND-* rules + state transition map
  - `src/domain/candidate/ports/ICandidateRepository.ts`
  - Stories: US-5.1, US-3.3, US-7.1

- [x] **Step 8**: Compliance domain
  - `src/domain/compliance/entities/ConsentRecord.ts`
  - `src/domain/compliance/entities/AuditEvent.ts` — AuditEvent, AuditEventType
  - `src/domain/compliance/rules/ComplianceRules.ts` — all BR-COMP-* rules
  - `src/domain/compliance/ports/IConsentRepository.ts`
  - `src/domain/compliance/ports/IAuditEventRepository.ts` — append-only interface (no update/delete)
  - Stories: US-4.1, US-1.2, US-7.1

- [x] **Step 9**: Domain layer unit tests
  - `tests/unit/domain/conversation/ConversationRules.test.ts`
  - `tests/unit/domain/evaluation/EvaluationRules.test.ts`
  - `tests/unit/domain/campaign/CampaignRules.test.ts`
  - `tests/unit/domain/candidate/CandidateRules.test.ts`
  - `tests/unit/domain/compliance/ComplianceRules.test.ts`
  - Coverage: all BR-* rules, state transitions, edge cases

---

### BLOCK 3 — Application Layer (Ports + Use Cases)

- [x] **Step 10**: Application ports (external service interfaces)
  - `src/application/conversation/ports/ILLMClient.ts` — LLM abstraction (generateResponse, generateScoringResponse)
  - `src/application/conversation/ports/IMessageSender.ts` — Telegram send abstraction

- [x] **Step 11**: Conversation use cases
  - `src/application/conversation/use-cases/StartScreeningUseCase.ts`
  - `src/application/conversation/use-cases/ProcessMessageUseCase.ts` — core orchestration (phase routing, follow-up logic, escalation)
  - `src/application/conversation/use-cases/HandleEscalationUseCase.ts`
  - `src/application/conversation/use-cases/CompleteScreeningUseCase.ts` — triggers GenerateSummary
  - Stories: US-1.1, US-1.2, US-1.3, US-1.4, US-1.5, US-1.6, US-1.8

- [x] **Step 12**: Evaluation use cases
  - `src/application/evaluation/use-cases/GenerateSummaryUseCase.ts` — sequential LLM calls per competency, evidence extraction, global score, recommendation
  - `src/application/evaluation/use-cases/GetEvaluationDetailUseCase.ts`
  - Stories: US-2.2, US-2.3

- [x] **Step 13**: Campaign use cases
  - `src/application/campaign/use-cases/CreateCampaignUseCase.ts`
  - `src/application/campaign/use-cases/UpdateCampaignUseCase.ts`
  - `src/application/campaign/use-cases/ListCampaignsUseCase.ts`
  - Stories: US-3.1

- [x] **Step 14**: Candidate use cases
  - `src/application/candidate/use-cases/ListCandidatesForReviewUseCase.ts` — FIFO sort, filters
  - `src/application/candidate/use-cases/ReviewCandidateUseCase.ts` — approve/reject + disagreement capture
  - `src/application/candidate/use-cases/GetCandidateDetailUseCase.ts`
  - Stories: US-3.2, US-3.3, US-5.1

- [x] **Step 15**: Compliance use cases
  - `src/application/compliance/use-cases/RecordConsentUseCase.ts`
  - `src/application/compliance/use-cases/LogAuditEventUseCase.ts`
  - Stories: US-4.1, US-1.2

- [x] **Step 16**: Application layer unit tests (mocked ports)
  - `tests/unit/application/conversation/ProcessMessageUseCase.test.ts` — all phase transitions, escalation paths
  - `tests/unit/application/conversation/CompleteScreeningUseCase.test.ts`
  - `tests/unit/application/evaluation/GenerateSummaryUseCase.test.ts` — scoring, evidence validation, recommendation thresholds
  - `tests/unit/application/candidate/ReviewCandidateUseCase.test.ts` — disagreement detection
  - All infrastructure dependencies mocked via port interfaces

---

### BLOCK 4 — Infrastructure Layer (Adapters)

- [x] **Step 17**: DynamoDB base + idempotency
  - `src/infrastructure/dynamodb/TenantScopedRepository.ts` — abstract base class (NC-03)
  - `src/infrastructure/dynamodb/IdempotencyStore.ts` — update_id dedup (NC-04)

- [x] **Step 18**: DynamoDB repository adapters
  - `src/infrastructure/dynamodb/adapters/DynamoDBConversationRepository.ts`
  - `src/infrastructure/dynamodb/adapters/DynamoDBCampaignRepository.ts`
  - `src/infrastructure/dynamodb/adapters/DynamoDBCandidateRepository.ts`
  - `src/infrastructure/dynamodb/adapters/DynamoDBEvaluationRepository.ts`
  - `src/infrastructure/dynamodb/adapters/DynamoDBAuditEventRepository.ts` — PutItem only, no update/delete
  - `src/infrastructure/dynamodb/adapters/DynamoDBConsentRepository.ts`
  - Stories: US-7.1 (tenant isolation via TenantScopedRepository)

- [x] **Step 19**: OpenAI adapter
  - `src/infrastructure/openai/adapters/OpenAILLMClient.ts` — implements ILLMClient
    - `generateResponse()`: screening conversation (system prompt + history)
    - `generateScoringResponse()`: structured JSON output for evaluation (score + evidence)
    - AbortController timeout (7s), retry via withRetry utility
  - Stories: US-1.4, US-2.2, US-2.3, US-7.2

- [x] **Step 20**: Telegram adapter
  - `src/infrastructure/telegram/adapters/TelegramMessageSender.ts` — implements IMessageSender
  - `src/infrastructure/telegram/WebhookAuthMiddleware.ts` — secret token validation (NC-07)
  - Stories: US-1.1, US-7.2

- [x] **Step 21**: Auth infrastructure
  - `src/infrastructure/auth/authOptions.ts` — NextAuth.js Cognito provider config
  - `src/infrastructure/auth/middleware.ts` — session validation + tenantId extraction
  - Stories: US-3.5, US-7.1

---

### BLOCK 5 — App Layer (Driving Adapters — API Routes)

- [x] **Step 22**: Telegram webhook route
  - `src/app/api/telegram/route.ts` — async fire-and-forget pattern, idempotency check, auth middleware
  - Stories: US-1.1 through US-1.8, US-7.2

- [x] **Step 23**: Campaign API routes
  - `src/app/api/campaigns/route.ts` — GET (list) + POST (create)
  - `src/app/api/campaigns/[id]/route.ts` — PATCH (update/activate/deactivate)
  - Stories: US-3.1

- [x] **Step 24**: Candidate + evaluation API routes
  - `src/app/api/candidates/route.ts` — GET (list with filters)
  - `src/app/api/candidates/[id]/route.ts` — GET (detail with evaluation + conversation)
  - `src/app/api/candidates/[id]/review/route.ts` — POST (approve/reject decision)
  - `src/app/api/evaluations/[id]/route.ts` — GET (evaluation detail)
  - Stories: US-3.2, US-3.3, US-5.1

- [x] **Step 25**: Auth + health routes
  - `src/app/api/auth/[...nextauth]/route.ts` — NextAuth.js handler
  - `src/app/api/health/route.ts` — shallow health check (NC-06)
  - Stories: US-3.5

---

### BLOCK 6 — Frontend (Dashboard UI)

- [x] **Step 26**: App shell + auth pages
  - `src/app/layout.tsx` — root layout with SessionProvider
  - `src/app/page.tsx` — redirect to /review or /login
  - `src/app/login/page.tsx` — LoginPage (dev login with test/test)
  - `src/app/(dashboard)/layout.tsx` — DashboardLayout (sidebar + auth guard)
  - Stories: US-3.5

- [x] **Step 27**: Campaign pages
  - `src/app/(dashboard)/campaigns/page.tsx` — CampaignListPage (RSC, table with status badges, copy link)
  - `src/app/(dashboard)/campaigns/new/page.tsx` — CampaignCreatePage (form with validation, basic requirements builder)
  - `data-testid` attributes on all interactive elements
  - Stories: US-3.1

- [x] **Step 28**: Review queue page
  - `src/app/(dashboard)/review/page.tsx` — ReviewQueuePage (filter bar, FIFO table, pending count)
  - `data-testid` attributes on all interactive elements
  - Stories: US-3.2

- [x] **Step 29**: Candidate detail page
  - `src/app/(dashboard)/candidates/[candidateId]/page.tsx` — CandidateDetailPage
    - Executive summary card (score, recommendation, key signals)
    - Competency scores with evidence quotes
    - Full scrollable transcript
    - Review decision form (approve/reject + auto-shown disagreement field)
  - `data-testid` attributes on all interactive elements
  - Stories: US-3.3

---

### BLOCK 7 — Tests

- [x] **Step 30**: API integration tests (Playwright)
  - `tests/integration/api/campaigns.spec.ts`
  - `tests/integration/api/candidates.spec.ts`
  - `tests/integration/api/telegram-webhook.spec.ts` — webhook auth, idempotency, async processing
  - Uses DynamoDB Local for test isolation

- [x] **Step 31**: E2E Gherkin feature files
  - `tests/e2e/features/screening-flow.feature` — US-1.1, US-1.2, US-1.3, US-1.4, US-1.5, US-1.6, US-1.8
  - `tests/e2e/features/evaluation.feature` — US-2.2, US-2.3
  - `tests/e2e/features/dashboard-campaigns.feature` — US-3.1
  - `tests/e2e/features/dashboard-review.feature` — US-3.2, US-3.3
  - `tests/e2e/features/authentication.feature` — US-3.5

- [x] **Step 32**: E2E Playwright step definitions
  - `tests/e2e/steps/screening-steps.ts`
  - `tests/e2e/steps/dashboard-steps.ts`
  - `tests/e2e/steps/auth-steps.ts`
  - Playwright page object models for dashboard pages

---

### BLOCK 8 — Infrastructure & Deployment

- [x] **Step 33**: Terraform modules
  - `terraform/modules/vpc/` — VPC, 2 AZs, public subnets, IGW, route tables, security groups, VPC endpoints (DynamoDB Gateway + ECR/CloudWatch/SecretsManager Interface)
  - `terraform/modules/ecr/` — ECR repository + lifecycle policy
  - `terraform/modules/dynamodb/` — all 7 tables with GSIs, TTL, deletion protection
  - `terraform/modules/cognito/` — User Pool + App Client + Hosted UI
  - `terraform/modules/secrets/` — Secrets Manager secrets (empty values)
  - `terraform/modules/cloudwatch/` — log groups + retention
  - `terraform/modules/alb/` — ALB, listeners, target group
  - `terraform/modules/ecs/` — cluster, task definition, service, IAM roles

- [x] **Step 34**: Terraform environments
  - `terraform/environments/dev/` — main.tf, variables.tf, terraform.tfvars, backend.tf
  - `terraform/environments/staging/` — same structure
  - `terraform/environments/prod/` — same structure + deletion protection enabled
  - `terraform/environments/localstack/` — local DynamoDB provisioning via LocalStack

- [x] **Step 35**: CI/CD + Docker
  - `Dockerfile` — multi-stage build (deps → builder → runner, Node 20 Alpine, non-root user)
  - `.dockerignore`
  - `.github/workflows/ci.yml` — PR: lint + type-check + unit tests
  - `.github/workflows/deploy-staging.yml` — push to develop: build + push ECR + deploy ECS
  - `.github/workflows/deploy-prod.yml` — push to main: build + push ECR + deploy ECS

---

### BLOCK 9 — Documentation

- [x] **Step 36**: Project documentation
  - `README.md` — project overview, local dev setup, environment variables, deployment guide
  - `aidlc-docs/construction/mvp-core/code/code-summary.md` — generated files inventory

---

## Step Sequence Summary

| Block | Steps | Scope |
|---|---|---|
| 1 — Foundation | 1-3 | Next.js init, shadcn/ui, shared modules |
| 2 — Domain | 4-9 | Entities, rules, ports, domain tests |
| 3 — Application | 10-16 | Use cases, app ports, use case tests |
| 4 — Infrastructure | 17-21 | DynamoDB adapters, OpenAI, Telegram, Auth |
| 5 — API Routes | 22-25 | Driving adapters (Next.js routes) |
| 6 — Frontend | 26-29 | Dashboard pages with data-testid |
| 7 — Tests | 30-32 | Integration tests, E2E Gherkin + Playwright |
| 8 — Infrastructure | 33-35 | Terraform modules + environments + CI/CD + Docker |
| 9 — Docs | 36 | README + code summary |
| **Total** | **36 steps** | |

## Dependency Order Rationale

Domain → Application → Infrastructure → App Layer → Frontend → Tests → Deployment

This order ensures:
1. Domain entities and rules exist before use cases reference them
2. Port interfaces exist before adapters implement them
3. Use cases exist before API routes call them
4. API routes exist before frontend pages call them
5. All code exists before tests reference it
6. All code exists before Docker/Terraform packages it
