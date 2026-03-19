# Logical Components — Unit 1: MVP Core

## Overview

This document describes the logical infrastructure components and cross-cutting modules introduced by NFR design. These complement the domain/application components from Functional Design.

All components respect the Hexagonal Architecture + Clean Architecture dependency rule defined in `nfr-design-patterns.md` (Pattern 0).

---

## Canonical Directory Structure

The full project structure reflecting Hexagonal + Clean Architecture + DDD:

```
src/
├── domain/                          # INNERMOST — zero external dependencies
│   ├── conversation/
│   │   ├── entities/
│   │   │   ├── Conversation.ts
│   │   │   ├── Message.ts
│   │   │   └── SessionState.ts
│   │   ├── rules/
│   │   │   └── ConversationRules.ts
│   │   └── ports/                   # Repository interfaces (driven ports)
│   │       └── IConversationRepository.ts
│   ├── evaluation/
│   │   ├── entities/
│   │   │   ├── Rubric.ts
│   │   │   ├── ExecutiveSummary.ts
│   │   │   └── Evidence.ts
│   │   ├── rules/
│   │   │   └── EvaluationRules.ts
│   │   └── ports/
│   │       ├── IRubricRepository.ts
│   │       └── IEvaluationRepository.ts
│   ├── campaign/
│   │   ├── entities/
│   │   │   └── Campaign.ts
│   │   ├── rules/
│   │   │   └── CampaignRules.ts
│   │   └── ports/
│   │       └── ICampaignRepository.ts
│   ├── candidate/
│   │   ├── entities/
│   │   │   └── Candidate.ts
│   │   ├── rules/
│   │   │   └── CandidateRules.ts
│   │   └── ports/
│   │       └── ICandidateRepository.ts
│   └── compliance/
│       ├── entities/
│       │   ├── ConsentRecord.ts
│       │   └── AuditEvent.ts
│       ├── rules/
│       │   └── ComplianceRules.ts
│       └── ports/
│           ├── IConsentRepository.ts
│           └── IAuditEventRepository.ts  # append-only interface
│
├── application/                     # USE CASES — depends only on domain
│   ├── conversation/
│   │   ├── use-cases/
│   │   │   ├── StartScreeningUseCase.ts
│   │   │   ├── ProcessMessageUseCase.ts
│   │   │   ├── HandleEscalationUseCase.ts
│   │   │   └── CompleteScreeningUseCase.ts
│   │   └── ports/                   # External service interfaces (driven ports)
│   │       ├── ILLMClient.ts        # OpenAI abstraction
│   │       └── IMessageSender.ts    # Telegram send abstraction
│   ├── evaluation/
│   │   └── use-cases/
│   │       ├── GenerateSummaryUseCase.ts
│   │       └── GetEvaluationDetailUseCase.ts
│   ├── campaign/
│   │   └── use-cases/
│   │       ├── CreateCampaignUseCase.ts
│   │       └── UpdateCampaignUseCase.ts
│   ├── candidate/
│   │   └── use-cases/
│   │       ├── ListCandidatesForReviewUseCase.ts
│   │       └── ReviewCandidateUseCase.ts
│   └── compliance/
│       └── use-cases/
│           ├── RecordConsentUseCase.ts
│           └── LogAuditEventUseCase.ts
│
├── infrastructure/                  # ADAPTERS — implements domain/application ports
│   ├── dynamodb/
│   │   ├── TenantScopedRepository.ts        # [NC-03] base class
│   │   ├── IdempotencyStore.ts              # [NC-04]
│   │   └── adapters/
│   │       ├── DynamoDBConversationRepository.ts  # implements IConversationRepository
│   │       ├── DynamoDBCampaignRepository.ts
│   │       ├── DynamoDBCandidateRepository.ts
│   │       ├── DynamoDBEvaluationRepository.ts
│   │       ├── DynamoDBAuditEventRepository.ts    # append-only
│   │       └── DynamoDBConsentRepository.ts
│   ├── openai/
│   │   └── adapters/
│   │       └── OpenAILLMClient.ts           # implements ILLMClient
│   ├── telegram/
│   │   ├── WebhookAuthMiddleware.ts         # [NC-07]
│   │   └── adapters/
│   │       └── TelegramMessageSender.ts     # implements IMessageSender
│   ├── auth/
│   │   └── authOptions.ts                  # NextAuth.js Cognito config
│   └── config/
│       └── secrets.ts                      # [NC-05] SecretsLoader
│
├── app/                             # DRIVING ADAPTERS — Next.js framework layer
│   ├── api/
│   │   ├── telegram/
│   │   │   └── route.ts            # Driving adapter → calls use cases
│   │   ├── campaigns/
│   │   │   └── route.ts
│   │   ├── candidates/
│   │   │   ├── route.ts
│   │   │   └── [id]/
│   │   │       ├── route.ts
│   │   │       └── review/route.ts
│   │   ├── evaluations/
│   │   │   └── route.ts
│   │   ├── health/
│   │   │   └── route.ts            # [NC-06] HealthCheckHandler
│   │   └── auth/
│   │       └── [...nextauth]/route.ts
│   └── (dashboard)/                # UI — depends on API routes only
│       ├── layout.tsx
│       ├── campaigns/
│       ├── review/
│       └── candidates/
│
└── shared/                          # Cross-cutting — no layer dependencies
    ├── utils/
    │   └── retry.ts                # [NC-01] RetryUtility
    ├── logging/
    │   └── logger.ts               # [NC-02] Logger
    └── types/
        └── common.ts               # Shared primitive types only
```

---

## New Components Introduced by NFR Design

### NC-01: RetryUtility

**Location**: `src/shared/utils/retry.ts`
**Purpose**: Generic exponential backoff retry wrapper used by OpenAI and DynamoDB clients.

**Interface**:
```typescript
interface RetryOptions {
  maxAttempts: number;
  baseDelayMs: number;
  label: string;
  retryableErrors?: (err: unknown) => boolean; // defaults to all errors
}

async function withRetry<T>(fn: () => Promise<T>, options: RetryOptions): Promise<T>
async function sleep(ms: number): Promise<void>
```

**Used by**: `OpenAIChatClient`, `OpenAIEmbeddingClient`, all DynamoDB repository write operations.

---

### NC-02: Logger

**Location**: `src/shared/logging/logger.ts`
**Purpose**: Structured JSON logger outputting to stdout (captured by CloudWatch via ECS awslogs driver).

**Interface**:
```typescript
interface LogContext {
  requestId?: string;
  tenantId?: string;
  service: string;
  conversationId?: string;
  candidateId?: string;
  campaignId?: string;
  durationMs?: number;
}

class Logger {
  withContext(ctx: Partial<LogContext>): Logger
  info(message: string, extra?: object): void
  warn(message: string, extra?: object): void
  error(message: string, extra?: object): void
  debug(message: string, extra?: object): void  // no-op unless LOG_LEVEL=debug
}

const logger: Logger  // singleton, service='entrievista-api'
```

**Request ID**: Generated as UUID at webhook entry point. Passed via `logger.withContext({ requestId })` to all downstream operations.

---

### NC-03: TenantScopedRepository (Base Class)

**Location**: `src/infrastructure/dynamodb/TenantScopedRepository.ts`
**Purpose**: Abstract base class enforcing tenant isolation on all DynamoDB operations.

**Interface**:
```typescript
abstract class TenantScopedRepository<T> {
  constructor(tableName: string, tenantId: string)
  protected buildPK(entityId: string): string  // returns `${tenantId}#${entityId}`
  protected async getItem(pk: string, sk?: string): Promise<T | null>
  protected async putItem(item: T & { pk: string }): Promise<void>  // with retry
  protected async updateItem(pk: string, updates: Partial<T>): Promise<void>  // with retry
  protected async query(pk: string, options?: QueryOptions): Promise<T[]>
}
```

All concrete repositories extend this class. `tenantId` is injected at construction — never passed as a method parameter.

---

### NC-04: IdempotencyStore

**Location**: `src/infrastructure/dynamodb/IdempotencyStore.ts`
**Purpose**: Tracks processed Telegram `update_id` values to prevent duplicate processing.

**Interface**:
```typescript
class IdempotencyStore {
  async markProcessed(updateId: number): Promise<boolean>
  // Returns true if successfully marked (first time seen)
  // Returns false if already exists (duplicate — skip processing)
  // Uses DynamoDB conditional PutItem + TTL (24h)
}
```

**DynamoDB table**: `processed_updates` — PK: `updateId` (number), TTL: `expiresAt` (Unix timestamp, now + 86400s).

---

### NC-05: SecretsLoader

**Location**: `src/infrastructure/config/secrets.ts`
**Purpose**: Loads application configuration from environment variables (injected by ECS from Secrets Manager at startup).

**Interface**:
```typescript
interface AppConfig {
  openaiApiKey: string;
  telegramBotToken: string;
  telegramWebhookSecret: string;
  cognitoClientSecret: string;
  dynamodbTablePrefix: string;  // e.g., 'entrievista-prod'
  botUsername: string;
  appVersion: string;
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  nodeEnv: 'development' | 'staging' | 'production';
}

function loadConfig(): AppConfig  // validates all required env vars at startup, throws if missing
```

**Fail-fast**: Called once at application startup. If any required secret is missing, the process exits with a clear error message before accepting traffic.

---

### NC-06: HealthCheckHandler

**Location**: `src/app/api/health/route.ts`
**Purpose**: Shallow health check endpoint for ALB target group health checks.

**Response**:
```json
{
  "status": "ok",
  "timestamp": "2026-03-18T00:00:00.000Z",
  "version": "1.0.0"
}
```

**HTTP 200** always (process alive = healthy). No external dependency checks.

---

### NC-07: WebhookAuthMiddleware

**Location**: `src/infrastructure/telegram/WebhookAuthMiddleware.ts`
**Purpose**: Validates `X-Telegram-Bot-Api-Secret-Token` header on every webhook request.

**Interface**:
```typescript
function validateWebhookSecret(req: Request): boolean
// Returns true if header matches configured secret
// Returns false (→ 403) if missing or invalid
```

---

## Updated Component Interactions

The NFR components integrate with the existing application layer as follows:

```
POST /api/telegram
  │
  ├─ [NC-07] WebhookAuthMiddleware — validate secret header
  │
  ├─ Return 200 OK immediately (async pattern)
  │
  └─ async processUpdate(update)
       │
       ├─ [NC-04] IdempotencyStore.markProcessed(update_id)
       │    └─ if duplicate → skip
       │
       ├─ [NC-02] Logger.withContext({ requestId, tenantId })
       │
       ├─ ScreeningOrchestrator
       │    ├─ [NC-03] TenantScopedRepository (load conversation, campaign)
       │    ├─ [NC-01] withRetry → OpenAI call (7s timeout)
       │    └─ [NC-01] withRetry → DynamoDB write (session state)
       │
       └─ TelegramBotService.sendMessage()


Dashboard API routes
  │
  ├─ NextAuth.js session validation (tenantId extraction)
  │
  └─ [NC-03] TenantScopedRepository (all queries scoped to tenantId)


Application startup
  └─ [NC-05] SecretsLoader.loadConfig() — fail-fast on missing secrets


ALB health check
  └─ GET /api/health → [NC-06] HealthCheckHandler
```

---

## Infrastructure Components (AWS)

These are provisioned via Terraform (detailed in Infrastructure Design stage).

| Component | Type | Purpose |
|---|---|---|
| ECS Cluster | AWS ECS | Container orchestration |
| Fargate Task | AWS ECS Fargate | 1 task, 512 vCPU / 1024 MB |
| ALB | AWS ALB | TLS termination, health checks, routing |
| DynamoDB Tables | AWS DynamoDB | 6 tables + `processed_updates` (idempotency) |
| Secrets Manager | AWS Secrets Manager | 4 secrets (OpenAI, Telegram, Cognito) |
| ECR | AWS ECR | Docker image registry |
| CloudWatch Logs | AWS CloudWatch | Log group `/ecs/entrievista-api` |
| Cognito User Pool | AWS Cognito | Recruiter authentication |
| VPC | AWS VPC | Network isolation (public + private subnets) |

---

## Cross-Cutting Concerns Summary

| Concern | Implementation | Scope |
|---|---|---|
| Tenant isolation | `TenantScopedRepository` base class | All DynamoDB access |
| Retry resilience | `withRetry` utility | OpenAI + DynamoDB writes |
| Structured logging | `Logger` singleton | All application code |
| Secret management | ECS secrets injection | Startup config |
| Webhook idempotency | `IdempotencyStore` | Telegram webhook handler |
| Webhook security | `WebhookAuthMiddleware` | `/api/telegram` route |
| Health monitoring | `/api/health` endpoint | ALB health checks |
| Async processing | Fire-and-forget pattern | Telegram webhook handler |
