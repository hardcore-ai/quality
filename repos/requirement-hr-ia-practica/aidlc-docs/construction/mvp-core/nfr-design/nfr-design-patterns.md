# NFR Design Patterns — Unit 1: MVP Core

## Overview

This document maps each NFR requirement to its concrete design pattern and implementation approach. Patterns are technology-agnostic where possible; infrastructure specifics are in `logical-components.md`.

---

## 0. Hexagonal Architecture + Clean Architecture + DDD (Architectural Constraint)

**Scope**: Entire application — this is the foundational architectural constraint that governs all other design decisions.

**Problem**: Without explicit architectural boundaries, dependencies tend to flow in the wrong direction — infrastructure details leak into business logic, making the domain untestable and tightly coupled to frameworks.

**Pattern**: Hexagonal Architecture (Ports & Adapters) aligned with Clean Architecture dependency rule and DDD tactical patterns.

### Dependency Rule (Clean Architecture)

Dependencies flow **inward only**. Outer layers know about inner layers; inner layers know nothing about outer layers.

```
+----------------------------------------------------------+
|  FRAMEWORKS & DRIVERS (outermost)                        |
|  Next.js App Router, grammY, AWS SDK, NextAuth.js        |
|  src/app/  +  src/infrastructure/                        |
|                                                          |
|  +----------------------------------------------------+  |
|  |  INTERFACE ADAPTERS                                |  |
|  |  Controllers (API routes), Presenters,             |  |
|  |  Repository implementations, External adapters     |  |
|  |  src/infrastructure/  (adapters)                   |  |
|  |                                                    |  |
|  |  +----------------------------------------------+ |  |
|  |  |  APPLICATION (Use Cases)                     | |  |
|  |  |  Orchestration, port interfaces defined here | |  |
|  |  |  src/application/                            | |  |
|  |  |                                              | |  |
|  |  |  +----------------------------------------+ | |  |
|  |  |  |  DOMAIN (innermost)                    | | |  |
|  |  |  |  Entities, Value Objects, Domain Rules | | |  |
|  |  |  |  Aggregates, Domain Events             | | |  |
|  |  |  |  src/domain/                           | | |  |
|  |  |  +----------------------------------------+ | |  |
|  |  +----------------------------------------------+ |  |
|  +----------------------------------------------------+  |
+----------------------------------------------------------+
```

**Hard rule**: `src/domain/` has **zero** imports from `src/application/`, `src/infrastructure/`, or any framework. Violations are caught by ESLint import rules.

### Hexagonal Architecture — Ports & Adapters

The application has two types of ports:

**Driving ports** (left side — how the outside world calls the app):
- `ITelegramMessageHandler` — called by the Telegram webhook adapter
- `IDashboardController` — called by Next.js API route adapters

**Driven ports** (right side — how the app calls the outside world):
- `IConversationRepository` — implemented by DynamoDB adapter
- `ICampaignRepository` — implemented by DynamoDB adapter
- `ICandidateRepository` — implemented by DynamoDB adapter
- `IEvaluationRepository` — implemented by DynamoDB adapter
- `IAuditEventRepository` — implemented by DynamoDB adapter (append-only)
- `IConsentRepository` — implemented by DynamoDB adapter
- `ILLMClient` — implemented by OpenAI adapter
- `IMessageSender` — implemented by Telegram Bot API adapter

```
                    +---------------------------+
  Telegram          |                           |          DynamoDB
  Webhook  -------> | [Driving Port]            |          Adapter
  Adapter           |   ITelegramMessageHandler |  ------> IConversationRepository
                    |                           |
  Next.js           |      APPLICATION          |          OpenAI
  API Route ------> | [Driving Port]            |  ------> ILLMClient
  Adapter           |   IDashboardController    |
                    |                           |          Telegram
                    |      DOMAIN               |  ------> IMessageSender
                    |   (pure, no deps)         |
                    +---------------------------+
                         [Driven Ports defined here,
                          implemented in infrastructure/]
```

### Port Interfaces Location

All port interfaces are defined in the **application layer** (or domain layer for repository interfaces). Infrastructure adapters implement them — the application never imports from infrastructure.

```
src/
├── domain/
│   └── {module}/
│       ├── entities/          # Aggregates, entities, value objects
│       ├── rules/             # Domain rules (pure functions)
│       └── ports/             # Repository interfaces (driven ports)
│           └── I{Entity}Repository.ts
│
├── application/
│   └── {module}/
│       ├── use-cases/         # One file per use case
│       └── ports/             # Service interfaces (driven ports for external services)
│           ├── ILLMClient.ts
│           └── IMessageSender.ts
│
├── infrastructure/
│   └── {module}/
│       └── adapters/          # Concrete implementations of ports
│           ├── DynamoDB{Entity}Repository.ts  (implements domain port)
│           ├── OpenAILLMClient.ts             (implements application port)
│           └── TelegramMessageSender.ts       (implements application port)
│
└── app/
    └── api/
        └── {route}/
            └── route.ts       # Driving adapter — calls application use cases
```

### DDD Tactical Patterns Applied

| DDD Pattern | Where Used |
|---|---|
| **Aggregate** | `Conversation`, `Campaign`, `Candidate`, `Rubric` — each owns its consistency boundary |
| **Entity** | `Message`, `CompetencyScore`, `Evidence`, `BasicRequirement` |
| **Value Object** | `ConversationState`, `CandidateState`, `Recommendation`, `EscalationLevel` |
| **Domain Service** | `ConversationRules`, `EvaluationRules`, `CampaignRules`, `CandidateRules` |
| **Repository** | Interface in domain, implementation in infrastructure |
| **Use Case / Application Service** | One class per use case in `application/` |
| **Anti-Corruption Layer** | Infrastructure adapters translate between external models (DynamoDB items, OpenAI responses) and domain entities |

### ESLint Enforcement

Dependency direction enforced via `eslint-plugin-import` boundaries:

```json
// .eslintrc rules (simplified)
"import/no-restricted-paths": [
  { "from": "src/domain", "target": "src/application" },   // domain cannot import application
  { "from": "src/domain", "target": "src/infrastructure" }, // domain cannot import infrastructure
  { "from": "src/application", "target": "src/infrastructure" }, // application cannot import infrastructure
  { "from": "src/application", "target": "src/app" }        // application cannot import Next.js app layer
]
```

---

## 1. Async Webhook Pattern (NFR-AVAIL-04)

**Problem**: Telegram requires a webhook HTTP response within 5 seconds. LLM processing takes up to 9 seconds.

**Pattern**: Fire-and-forget with out-of-band response.

```
┌─────────────┐   POST /api/telegram   ┌──────────────────────┐
│  Telegram   │ ─────────────────────> │  WebhookHandler      │
│  Servers    │ <─── 200 OK (< 100ms) ─│  (Next.js API route) │
└─────────────┘                        └──────────┬───────────┘
                                                   │ async (no await)
                                                   v
                                        ┌──────────────────────┐
                                        │  ScreeningOrchestrator│
                                        │  - Load session       │
                                        │  - Build prompt       │
                                        │  - Call OpenAI        │
                                        │  - Save state         │
                                        └──────────┬───────────┘
                                                   │
                                                   v
                                        ┌──────────────────────┐
                                        │  TelegramBotService  │
                                        │  sendMessage()       │
                                        └──────────────────────┘
```

**Implementation**:
```typescript
// app/api/telegram/route.ts
export async function POST(req: Request) {
  const update = await req.json();
  
  // Respond to Telegram immediately
  const response = new Response('OK', { status: 200 });
  
  // Process asynchronously — do NOT await
  processUpdate(update).catch(err => logger.error('Async processing failed', { err }));
  
  return response;
}
```

**Duplicate prevention**: Store processed `update_id` values in DynamoDB with TTL (24h). Check before processing — skip if already seen.

---

## 2. Retry with Exponential Backoff (NFR-AVAIL-02, NFR-AVAIL-03)

**Problem**: OpenAI API and DynamoDB can fail transiently. Naive retries can overwhelm a recovering service.

**Pattern**: Exponential backoff with jitter.

```
Attempt 1: immediate
Attempt 2: wait base × 2^0 + jitter = 1s ± 200ms
Attempt 3: wait base × 2^1 + jitter = 2s ± 200ms
Attempt 4: wait base × 2^2 + jitter = 4s ± 200ms
→ Fail after 4 attempts
```

**Generic retry utility**:
```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxAttempts: number; baseDelayMs: number; label: string }
): Promise<T> {
  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === options.maxAttempts) throw err;
      const delay = options.baseDelayMs * Math.pow(2, attempt - 1) + Math.random() * 200;
      logger.warn(`${options.label} attempt ${attempt} failed, retrying in ${delay}ms`);
      await sleep(delay);
    }
  }
  throw new Error('unreachable');
}
```

**Applied to**:
- OpenAI calls: `maxAttempts: 4`, `baseDelayMs: 1000` — retries on 429, 5xx, timeout
- DynamoDB writes: `maxAttempts: 3`, `baseDelayMs: 500` — retries on `ProvisionedThroughputExceededException`, `ServiceUnavailable`

---

## 3. Latency Budget Enforcement (NFR-PERF-01)

**Problem**: End-to-end response must be ≤ 10s. OpenAI is the dominant cost at ≤ 7s.

**Pattern**: Per-operation timeout with AbortController.

```typescript
// OpenAI call with hard 7s timeout
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 7000);

try {
  const response = await openai.chat.completions.create(
    { model: 'gpt-4o', messages, ... },
    { signal: controller.signal }
  );
  return response;
} finally {
  clearTimeout(timeout);
}
```

**Slow response warning**: If total processing exceeds 8s, log a `warn` entry with `durationMs` for monitoring.

---

## 4. Structured Logging Pattern (NFR-OBS-01)

**Problem**: CloudWatch needs structured, queryable logs. Console.log is insufficient.

**Pattern**: Centralized logger with mandatory context fields.

```typescript
// shared/logging/logger.ts
interface LogContext {
  requestId: string;
  tenantId: string;
  service: string;
  conversationId?: string;
  candidateId?: string;
  campaignId?: string;
  durationMs?: number;
  errorCode?: string;
}

class Logger {
  private context: Partial<LogContext>;

  withContext(ctx: Partial<LogContext>): Logger { ... }

  info(message: string, extra?: Record<string, unknown>): void {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'info',
      message,
      ...this.context,
      ...extra,
    }));
  }

  warn(message: string, extra?: Record<string, unknown>): void { ... }
  error(message: string, extra?: Record<string, unknown>): void { ... }
  debug(message: string, extra?: Record<string, unknown>): void {
    if (process.env.LOG_LEVEL === 'debug') { ... }
  }
}
```

**Request ID propagation**: Generated at webhook entry point, threaded through all downstream calls via logger context.

---

## 5. Repository Pattern with Tenant Isolation (NFR-SEC-05)

**Problem**: Every DynamoDB query must be scoped to a tenant. Ad-hoc tenantId injection is error-prone.

**Pattern**: Base repository enforces tenant scoping at the infrastructure boundary.

```typescript
abstract class TenantScopedRepository<T> {
  constructor(
    protected readonly tableName: string,
    protected readonly tenantId: string  // Injected at construction time
  ) {}

  protected buildPK(entityId: string): string {
    return `${this.tenantId}#${entityId}`;
  }

  // All query methods automatically scope to this.tenantId
  // No method accepts tenantId as a parameter — it's fixed at construction
}
```

**Usage**: Repositories are instantiated per-request with the tenantId from the authenticated session. Cross-tenant access is structurally impossible.

---

## 6. Secrets Injection Pattern (NFR-SEC-02)

**Problem**: Secrets must not be in code or environment files committed to git.

**Pattern**: ECS secrets injection from AWS Secrets Manager at task startup.

```
AWS Secrets Manager
  └── entrievista/openai-api-key
  └── entrievista/telegram-bot-token
  └── entrievista/telegram-webhook-secret
  └── entrievista/cognito-client-secret
         │
         │ (at ECS task startup)
         v
ECS Task Definition (secrets field)
  └── OPENAI_API_KEY → from Secrets Manager ARN
  └── TELEGRAM_BOT_TOKEN → from Secrets Manager ARN
  └── TELEGRAM_WEBHOOK_SECRET → from Secrets Manager ARN
  └── COGNITO_CLIENT_SECRET → from Secrets Manager ARN
         │
         v
Next.js process.env.*  (available at runtime, never in code)
```

**Local development**: `.env.local` file (gitignored) with development secrets.

---

## 7. Append-Only Audit Log Pattern (NFR-SEC-03, BR-COMP-02)

**Problem**: Audit events must be immutable — no updates or deletes allowed.

**Pattern**: Write-only repository interface + IAM enforcement.

```typescript
// infrastructure/dynamodb/repositories/AuditEventRepository.ts
interface AuditEventRepository {
  append(event: Omit<AuditEvent, 'id' | 'timestamp'>): Promise<void>;
  findByEntity(entityId: string): Promise<AuditEvent[]>;
  // NO update() method
  // NO delete() method
}
```

**IAM enforcement**: ECS task role policy grants only `dynamodb:PutItem` and `dynamodb:Query` on the `audit_events` table. `UpdateItem` and `DeleteItem` are explicitly denied.

```json
{
  "Effect": "Deny",
  "Action": ["dynamodb:UpdateItem", "dynamodb:DeleteItem"],
  "Resource": "arn:aws:dynamodb:*:*:table/audit_events"
}
```

---

## 8. Health Check Pattern (NFR-AVAIL-01)

**Problem**: ALB needs a reliable health check endpoint that reflects actual application readiness.

**Pattern**: Shallow health check (fast, no external dependencies).

```typescript
// app/api/health/route.ts
export async function GET() {
  return Response.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION ?? 'unknown',
  });
}
```

**Rationale**: Deep health checks (checking DynamoDB, OpenAI) risk false negatives causing unnecessary task restarts. Shallow check confirms the process is alive and serving requests.

---

## 9. Idempotency Pattern — Telegram Updates (NFR-AVAIL-04)

**Problem**: Telegram retries webhook delivery if it doesn't receive a 200 OK within 5s. With the async pattern, Telegram may retry before the first processing completes, causing duplicate messages.

**Pattern**: Idempotency key check using `update_id`.

```
On webhook receipt:
  1. Extract update.update_id
  2. Attempt conditional write to DynamoDB processed_updates table:
     PutItem with ConditionExpression: attribute_not_exists(updateId)
  3. IF condition fails (already exists) → skip processing, return 200 OK
  4. IF condition succeeds → proceed with async processing
  5. TTL on processed_updates items: 24 hours
```

This ensures each Telegram update is processed exactly once.

---

## 10. Gherkin-to-Playwright Test Pattern (NFR-TEST-02)

**Problem**: User story acceptance criteria (Given/When/Then) must map directly to automated tests.

**Pattern**: Cucumber + Playwright integration.

```
User Story (stories.md)
  └── Acceptance Criteria (Given/When/Then)
         │
         v
.feature file (tests/e2e/features/)
  └── Scenario: Bot Onboarding and AI Disclosure
      Given the candidate clicks the campaign Telegram link
      When the bot initiates the conversation
      Then the first message includes AI identity disclosure
         │
         v
Step definitions (tests/e2e/steps/)
  └── Playwright actions + assertions
```

**Traceability**: Each `.feature` file references the source user story ID in a comment (`# US-1.1`). CI output links test failures back to specific acceptance criteria.

---

## Pattern Summary

| Pattern | NFR Addressed | Location |
|---|---|---|
| **Hexagonal Architecture (Ports & Adapters)** | **Maintainability, Testability** | **All layers** |
| Async Webhook | AVAIL-04, PERF-01 | `infrastructure/telegram/` |
| Exponential Backoff | AVAIL-02, AVAIL-03 | `shared/utils/retry.ts` |
| Latency Budget (AbortController) | PERF-01 | `infrastructure/openai/` |
| Structured Logger | OBS-01 | `shared/logging/logger.ts` |
| Tenant-Scoped Repository | SEC-05 | `infrastructure/dynamodb/` |
| Secrets Injection | SEC-02 | ECS task definition (Terraform) |
| Append-Only Audit | SEC-03, COMP-02 | `infrastructure/dynamodb/repositories/` |
| Shallow Health Check | AVAIL-01 | `app/api/health/route.ts` |
| Idempotency (update_id) | AVAIL-04 | `infrastructure/telegram/` |
| Gherkin-Playwright | TEST-02 | `tests/e2e/` |
