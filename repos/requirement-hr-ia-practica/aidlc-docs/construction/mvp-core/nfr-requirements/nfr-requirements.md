# NFR Requirements — Unit 1: MVP Core

## Summary of Decisions

| Area | Decision |
|---|---|
| Latency budget | OpenAI ≤ 7s / DynamoDB ≤ 1s / Overhead ≤ 2s |
| Summary generation | Sequential LLM calls (acceptable within 30s) |
| DynamoDB capacity | On-demand (pay-per-request) |
| ECS task count | 1 task (MVP — cost-optimized) |
| OpenAI failure | Retry 3× exponential backoff, then error message |
| DynamoDB write failure | Retry 3×, then fail gracefully |
| Telegram webhook | Async: 200 OK immediately, process + respond via Bot API |
| Webhook security | X-Telegram-Bot-Api-Secret-Token header validation |
| Secrets | AWS Secrets Manager |
| DynamoDB IAM | Task role with least-privilege table-level permissions |
| JWT validation | NextAuth.js Cognito provider |
| Logging | Structured JSON: timestamp, level, message, requestId, tenantId, service |
| Alerting | Manual log review (no alarms for MVP) |
| Concurrency | Soft target 50 — no enforcement, ECS handles it |
| Token budget | Full conversation history (no trimming for MVP) |
| Testing | Unit + Integration + E2E with Playwright (API + browser) |
| TypeScript | `strict: true` |

---

## 1. Performance Requirements

### NFR-PERF-01: Bot Response Latency

**Requirement**: End-to-end bot response ≤ 10 seconds from message receipt to Telegram delivery.

**Budget breakdown** (A1):
| Component | Budget |
|---|---|
| OpenAI API call | ≤ 7s |
| DynamoDB read + write | ≤ 1s |
| Overhead (webhook parse, prompt build, send) | ≤ 2s |
| **Total** | **≤ 10s** |

**Implementation note**: The webhook handler responds to Telegram with `200 OK` immediately (async pattern — B4). The actual processing and bot response happen asynchronously via the Telegram Bot API `sendMessage` call. This decouples Telegram's 5s webhook timeout from the LLM processing time.

### NFR-PERF-02: Executive Summary Generation

**Requirement**: Executive summary generated ≤ 30 seconds after screening completion.

**Strategy** (A2): Sequential LLM calls — one per competency (typically 5 competencies).
- Estimated: 5 × ~4s = ~20s sequential → within 30s budget
- No parallelization needed for MVP
- Hard timeout: if total exceeds 28s, log warning and continue (do not fail)

### NFR-PERF-03: Dashboard Page Load

**Requirement**: Dashboard pages load ≤ 3 seconds.

**Strategy**: Next.js React Server Components for data fetching. DynamoDB queries scoped by tenantId + GSI. No client-side data fetching on initial load.

### NFR-PERF-04: DynamoDB Capacity

**Mode** (A3): On-demand (pay-per-request).
- No capacity planning required for MVP volume (100-500 candidates / 90 days)
- Auto-scales to handle burst traffic
- Cost acceptable at MVP scale
- Revisit at post-MVP scale

---

## 2. Availability & Reliability Requirements

### NFR-AVAIL-01: ECS Task Configuration

**Task count** (B1): 1 Fargate task for all environments (dev, staging, prod) in MVP.
- Single point of failure accepted for MVP cost optimization
- ECS service restart policy: restart on failure (ECS handles this automatically)
- Target: 99.5% uptime achieved via ECS service health checks + ALB health checks

### NFR-AVAIL-02: OpenAI API Failure Handling

**Strategy** (B2): Retry with exponential backoff.

```
Attempt 1: immediate
Attempt 2: wait 1s
Attempt 3: wait 2s
Attempt 4: wait 4s
After 4 attempts failed → send error message to candidate:
  "Tuve un problema técnico. Por favor intenta de nuevo en un momento."
  Session state preserved — candidate can retry by sending any message.
```

**Applicable errors**: timeout (> 7s), HTTP 429 (rate limit), HTTP 5xx.
**Non-retryable**: HTTP 400 (bad request — log and fail immediately).

### NFR-AVAIL-03: DynamoDB Write Failure Handling

**Strategy** (B3): Retry up to 3 times, then fail gracefully.

```
Attempt 1: immediate
Attempt 2: wait 500ms
Attempt 3: wait 1s
After 3 attempts failed:
  - Log error with full context (tenantId, conversationId, operation)
  - Send error message to candidate: "Tuve un problema técnico. Tu progreso puede no haberse guardado. Por favor intenta de nuevo."
  - Session remains recoverable from last successful persist
```

### NFR-AVAIL-04: Telegram Webhook Async Pattern

**Strategy** (B4): Decouple webhook receipt from processing.

```
1. Webhook handler receives Telegram update
2. Immediately return HTTP 200 OK to Telegram (< 100ms)
3. Process message asynchronously (LLM call, state update)
4. Send response to candidate via Telegram Bot API sendMessage
```

This prevents Telegram from retrying the webhook due to timeout, which would cause duplicate message processing.

**Duplicate prevention**: Check `update_id` — if already processed, skip silently.

### NFR-AVAIL-05: Session Durability

All session state persisted to DynamoDB after every message (A4 from Functional Design). On ECS task restart, the next message from the candidate loads the last persisted state — no progress lost.

---

## 3. Security Requirements

### NFR-SEC-01: Telegram Webhook Authentication

**Method** (C1): Validate `X-Telegram-Bot-Api-Secret-Token` header on every request to `/api/telegram`.

```
- Secret token set when registering webhook with Telegram Bot API
- Token stored in AWS Secrets Manager
- If header missing or invalid → return HTTP 403, log attempt
```

### NFR-SEC-02: Secrets Management

**Storage** (C2): AWS Secrets Manager for all sensitive credentials.

Secrets to store:
| Secret | Key |
|---|---|
| OpenAI API key | `entrievista/openai-api-key` |
| Telegram bot token | `entrievista/telegram-bot-token` |
| Telegram webhook secret | `entrievista/telegram-webhook-secret` |
| Cognito client secret | `entrievista/cognito-client-secret` |

**Access pattern**: ECS task IAM role has `secretsmanager:GetSecretValue` permission for specific secret ARNs. Secrets injected as environment variables at task startup via ECS secrets configuration.

### NFR-SEC-03: DynamoDB Access Control

**Method** (C3): IAM role attached to ECS task definition — least privilege.

IAM policy grants:
- `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem`, `dynamodb:Query` on specific table ARNs
- `dynamodb:PutItem` only (no Get/Update/Delete) on `audit_events` table (append-only enforcement)
- No `dynamodb:DeleteItem` on any table
- No `dynamodb:*` wildcard

### NFR-SEC-04: Authentication — Dashboard

**Method** (C4): NextAuth.js with AWS Cognito provider.

- All dashboard routes protected by NextAuth.js middleware
- Session validated on every request via `getServerSession()`
- `tenantId` extracted from Cognito custom attribute `custom:tenantId`
- API routes return `401` if no valid session, `403` if tenantId missing

### NFR-SEC-05: Data Isolation

All DynamoDB queries include `tenantId` in the partition key (`tenantId#entityId` pattern). No query can return data from another tenant without knowing their tenantId, which is never exposed cross-tenant.

### NFR-SEC-06: Transport Security

All traffic over HTTPS:
- ALB terminates TLS (ACM certificate)
- ECS tasks communicate with ALB over HTTP internally (within VPC)
- All external API calls (OpenAI, Telegram, Cognito) over HTTPS

---

## 4. Observability Requirements

### NFR-OBS-01: Structured Logging

**Format** (D1): JSON to stdout → CloudWatch Logs via ECS log driver (`awslogs`).

**Mandatory fields per log entry**:
```json
{
  "timestamp": "2026-03-18T00:00:00.000Z",
  "level": "info|warn|error|debug",
  "message": "Human-readable description",
  "requestId": "uuid",
  "tenantId": "tenant-id or 'system'",
  "service": "entrievista-api"
}
```

**Optional contextual fields** (added when available):
- `conversationId`, `candidateId`, `campaignId`, `durationMs`, `errorCode`, `stack`

**Log levels**:
- `error`: unhandled exceptions, failed retries, data integrity issues
- `warn`: retried operations, degraded behavior, slow responses (> 8s)
- `info`: phase transitions, audit events, successful operations
- `debug`: prompt content, DynamoDB queries (disabled in prod)

### NFR-OBS-02: Alerting

**Strategy** (D2): Manual log review for MVP — no CloudWatch Alarms configured.

CloudWatch Log Groups:
- `/ecs/entrievista-api` — application logs
- `/ecs/entrievista-api/errors` — error-level logs only (filtered)

Revisit alerting strategy post-MVP when production traffic warrants it.

### NFR-OBS-03: Health Check

ALB health check endpoint: `GET /api/health` → returns `{ status: 'ok', timestamp }` with HTTP 200.
ECS service monitors task health via ALB target group health checks (interval: 30s, threshold: 2).

---

## 5. Scalability Requirements

### NFR-SCALE-01: Concurrent Conversations

**Target** (E1): 50 concurrent Telegram conversations — soft target, no hard enforcement.

ECS single task handles concurrent requests via Node.js event loop (non-blocking I/O). At 50 concurrent conversations, the bottleneck is OpenAI API rate limits, not ECS compute. No concurrency counter implemented for MVP.

### NFR-SCALE-02: Token Budget

**Strategy** (E2): Full conversation history injected into every prompt — no trimming.

Rationale: A 15-25 minute screening conversation with 5 competencies (main + follow-up each) generates approximately 20-30 messages. At ~100 tokens/message average, total conversation history ≈ 2,000-3,000 tokens — well within GPT-4o's 128K context window. No sliding window or summarization needed for MVP.

Monitor token usage per conversation in logs (`durationMs` + token count from OpenAI response).

---

## 6. Testing Requirements

### NFR-TEST-01: Test Strategy

**Coverage** (F1): Unit + Integration + E2E with Playwright.

| Layer | Framework | Scope |
|---|---|---|
| Unit tests | Vitest | Domain rules, use cases, pure functions |
| Integration tests | Playwright API testing | API routes, DynamoDB interactions (test DB) |
| E2E tests | Playwright browser | Full dashboard flows + Telegram bot simulation |

### NFR-TEST-02: E2E Test Scenarios

E2E tests implemented using Gherkin scenarios from user stories. Each user story's Given/When/Then acceptance criteria maps directly to a Playwright test.

**Priority E2E scenarios** (from Unit 1 stories):
- US-1.1: Bot onboarding and AI disclosure
- US-1.2: Consent collection (granted + denied paths)
- US-1.3: Basic requirements verification (pass + fail paths)
- US-1.4: Competency-based screening questions
- US-1.5: Anti-hallucination handling
- US-2.2: Real-time evaluation during screening
- US-2.3: Executive summary generation
- US-3.1: Campaign creation
- US-3.2: HITL review queue
- US-3.3: Candidate detail view and decision
- US-3.5: Authentication flow

### NFR-TEST-03: Test File Structure

```
tests/
├── unit/
│   ├── domain/
│   │   ├── conversation/    # ConversationRules tests
│   │   ├── evaluation/      # EvaluationRules tests
│   │   ├── campaign/        # CampaignRules tests
│   │   └── candidate/       # CandidateRules tests
│   └── application/
│       ├── conversation/    # Use case unit tests (mocked infra)
│       └── evaluation/      # Use case unit tests (mocked infra)
├── integration/
│   ├── api/
│   │   ├── campaigns.spec.ts
│   │   ├── candidates.spec.ts
│   │   └── telegram-webhook.spec.ts
│   └── repositories/
│       └── dynamodb.spec.ts  # Against local DynamoDB (DynamoDB Local)
└── e2e/
    ├── features/             # Gherkin .feature files (from user stories)
    │   ├── screening-flow.feature
    │   ├── evaluation.feature
    │   ├── dashboard-campaigns.feature
    │   ├── dashboard-review.feature
    │   └── authentication.feature
    └── steps/                # Playwright step implementations
        ├── screening-steps.ts
        ├── dashboard-steps.ts
        └── auth-steps.ts
```

### NFR-TEST-04: TypeScript Configuration

**Strictness** (F2): `strict: true`.

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "skipLibCheck": true,
    "noEmit": true
  }
}
```

`skipLibCheck: true` included for build performance (does not affect application code strictness).
