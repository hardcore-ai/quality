# NFR Requirements Plan — Unit 1: MVP Core

## Unit Context
- **Unit**: `mvp-core`
- **Stack**: Next.js 16, DynamoDB, OpenAI GPT-4o, grammY, AWS Cognito, ECS Fargate, ALB, Terraform
- **Functional Design**: Complete (domain entities, business rules, business logic model, frontend components)

---

## Execution Checklist

- [x] Step 1: Analyze functional design artifacts
- [x] Step 2: Create NFR requirements plan (this file)
- [x] Step 3: Generate clarifying questions (below)
- [x] Step 4: Store plan
- [x] Step 5: Collect and analyze answers
- [x] Step 6: Generate NFR artifacts
  - [x] nfr-requirements.md
  - [x] tech-stack-decisions.md
- [ ] Step 7: Present completion message
- [ ] Step 8: Wait for explicit approval
- [ ] Step 9: Record approval and update progress

---

## Clarifying Questions

### Section A: Performance

**A1. OpenAI latency budget**

The requirement is < 10 seconds for bot responses (including LLM processing). The flow involves: receive Telegram webhook → load session state → build prompt → call OpenAI → save state → send response. How should the latency budget be distributed?

A) No strict sub-budget — just optimize for the 10s total end-to-end target
B) OpenAI call budget: ≤ 7s | DynamoDB ops: ≤ 1s | Overhead: ≤ 2s
C) OpenAI call budget: ≤ 8s | DynamoDB ops: ≤ 1s | Overhead: ≤ 1s
D) Define a timeout: if OpenAI exceeds X seconds, send a "procesando..." message and retry

[Answer]: B

---

**A2. Executive summary generation timeout**

Summary generation requires N LLM calls (one per competency, typically 5). The requirement is ≤ 30 seconds total. If the calls are sequential, 5 × ~4s = ~20s. Should they run in parallel?

A) Sequential — simpler, acceptable within 30s budget
B) Parallel (Promise.all) — faster, but higher concurrent OpenAI API usage
C) Parallel with concurrency limit (e.g., max 3 simultaneous calls)
D) Sequential with a hard timeout: if > 25s, save partial summary and complete async

[Answer]: A

---

**A3. DynamoDB read/write capacity**

For MVP (100-500 candidates over 90 days), what DynamoDB capacity mode should be used?

A) On-demand (pay-per-request) — no capacity planning needed, auto-scales, higher per-request cost
B) Provisioned with auto-scaling — lower cost at steady load, requires capacity estimation
C) On-demand for MVP, plan to switch to provisioned at scale
D) Provisioned fixed (no auto-scaling) — simplest Terraform config, acceptable for MVP volume

[Answer]: A

---

### Section B: Availability & Reliability

**B1. ECS task count for MVP**

How many ECS Fargate tasks should run for the MVP deployment?

A) 1 task (minimum cost, single point of failure acceptable for MVP)
B) 2 tasks minimum (basic redundancy, ALB distributes traffic)
C) 2 tasks with auto-scaling (scale up to 4 on load)
D) 1 task for dev/staging, 2 tasks for production

[Answer]: A

---

**B2. OpenAI API failure handling**

If the OpenAI API call fails (timeout, rate limit, 5xx), what should the bot do?

A) Retry once after 2 seconds, then send error message to candidate: "Tuve un problema técnico. Por favor intenta de nuevo en un momento."
B) Retry up to 3 times with exponential backoff (1s, 2s, 4s), then send error message
C) Retry once, then save session state and ask candidate to send their message again
D) No retry — immediately send error message and preserve session state for resume

[Answer]:  B

---

**B3. DynamoDB write failure handling**

If a DynamoDB write fails (e.g., session state persistence after a message), what should happen?

A) Log the error, continue processing — accept potential state loss for MVP
B) Retry once, then fail the entire message processing (send error to candidate)
C) Retry up to 3 times, then fail gracefully (send error to candidate, session recoverable)
D) Queue the write for async retry — respond to candidate immediately

[Answer]: C

---

**B4. Telegram webhook timeout**

Telegram requires a webhook response within 5 seconds, but LLM processing can take up to 8-9 seconds. How should this be handled?

A) Respond to Telegram immediately with 200 OK, process asynchronously, send response via Bot API
B) Use Telegram's sendChatAction ("typing...") to buy time, then respond synchronously
C) Accept the timeout risk — Telegram retries, and the second attempt finds the response ready
D) Implement a queue (e.g., SQS) between webhook receipt and processing

[Answer]: A

---

### Section C: Security

**C1. Telegram webhook security**

The `/api/telegram` endpoint receives all bot updates. How should it be secured?

A) Validate  Telegram's secret token header (`X-Telegram-Bot-Api-Secret-Token`)
B) Validate source IP against Telegram's known IP ranges
C) Both A and B
D) No validation needed — the endpoint only processes Telegram-formatted payloads

[Answer]: A

---

**C2. OpenAI API key storage**

The OpenAI API key is a sensitive secret. Where should it be stored?

A) AWS Secrets Manager — retrieved at runtime, rotatable, auditable
B) ECS task environment variables (set via Terraform from a `.tfvars` file not committed to git)
C) AWS Systems Manager Parameter Store (SecureString)
D) Environment variable in `.env.local` for dev, ECS task definition for prod

[Answer]: A

---

**C3. DynamoDB access control**

How should the ECS task access DynamoDB?

A) IAM role attached to ECS task definition — least privilege, no credentials in code
B) IAM user with access keys stored in environment variables
C) IAM role with full DynamoDB access (simpler for MVP)
D) IAM role with table-level permissions (specific tables only, no admin access)

[Answer]: A

---

**C4. Cognito JWT validation**

When the dashboard API routes receive requests, how is the Cognito JWT validated?

A) NextAuth.js handles validation automatically via the Cognito provider
B) Manual JWT verification using `jsonwebtoken` + Cognito JWKS endpoint
C) API Gateway handles JWT validation (not applicable — direct ECS deployment)
D) Both A (NextAuth session) + B (manual verification for API routes without NextAuth middleware)

[Answer]: A

---

### Section D: Observability

**D1. Structured logging fields**

CloudWatch will receive structured JSON logs. What fields should every log entry include?

A) timestamp, level, message, requestId, tenantId, service
B) timestamp, level, message, requestId, tenantId, service, conversationId (when applicable), durationMs
C) timestamp, level, message only (minimal logging for MVP)
D) timestamp, level, message, requestId, tenantId, service, conversationId, candidateId, campaignId, durationMs

[Answer]: A

---

**D2. Error alerting**

For MVP, what error alerting is needed beyond CloudWatch logs?

A) CloudWatch Alarms on ECS task health + error log metric filter → SNS email notification
B) No alerting for MVP — manual log review is sufficient
C) CloudWatch Alarms only on ECS task health (no log-based alarms)
D) CloudWatch Alarms on: ECS health + 5xx error rate + OpenAI timeout rate → SNS email

[Answer]: B

---

### Section E: Scalability & Capacity

**E1. Concurrent conversation limit**

The requirement states support for up to 50 concurrent Telegram conversations for MVP. How should this be enforced?

A) No enforcement — ECS auto-scaling handles it; 50 is a soft target not a hard limit
B) Implement a concurrency counter in DynamoDB; reject new conversations above the limit
C) ECS task count × estimated conversations per task = capacity; scale tasks to meet demand
D) No limit needed for MVP — the actual load will be well below 50

[Answer]: A

---

**E2. Token budget for OpenAI prompts**

GPT-4o has a 128K context window. For a 15-25 minute screening conversation, what token budget strategy should be used?

A) No budget management — inject full conversation history (safe for MVP session lengths)
B) Sliding window: keep last 20 messages + system prompt (trim older messages)
C) Summarize older messages when context exceeds 50K tokens
D) Fixed budget: system prompt ≤ 4K tokens, conversation history ≤ 8K tokens, total ≤ 12K

[Answer]: A

---

### Section F: Testing & Quality

**F1. Test coverage targets**

What test coverage is expected for MVP?

A) Unit tests for domain rules and use cases only (no integration tests for MVP)
B) Unit tests for domain + integration tests for critical paths (screening flow, evaluation)
C) Unit tests + integration tests + E2E tests for the full screening flow
D) No automated tests for MVP — manual testing only

[Answer]: C. Adicional, debes realizar la implementación de pruebas con el framework Playwright tanto para API como para End to End. Asegurate de implementar las pruebas E2E usando los escenarios gherkin de las historias de usuario. 

---

**F2. TypeScript strictness**

What TypeScript compiler strictness level should be used?

A) `strict: true` (all strict checks enabled — recommended)
B) `strict: true` + `noUncheckedIndexedAccess: true` (maximum strictness)
C) Default TypeScript config (no strict mode)
D) `strict: true` but with `skipLibCheck: true` for faster builds

[Answer]: A

---
