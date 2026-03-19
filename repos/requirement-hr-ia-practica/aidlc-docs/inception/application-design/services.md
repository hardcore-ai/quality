# Services — EntreVista AI

## Service Architecture

The application follows a **use case-driven service layer** where each use case is a self-contained operation. Services in the application layer orchestrate domain logic and infrastructure calls.

---

## S1: Screening Orchestrator

| Attribute | Detail |
|---|---|
| **Name** | `ScreeningOrchestrator` |
| **Location** | `application/conversation/` |
| **Purpose** | Central orchestrator for the end-to-end screening flow from Telegram message to agent response |
| **Responsibilities** | Route incoming messages to the correct use case based on conversation state, coordinate between conversation, evaluation, and compliance |

**Orchestration Flow**:
```
Telegram Update
  → TelegramWebhookHandler (infrastructure)
    → ScreeningOrchestrator (application)
      → Determine conversation state
      → IF new: StartScreeningUseCase
      → IF active: ProcessMessageUseCase
        → Build prompt with session state + knowledge context
        → Call OpenAI for response
        → Evaluate response against rubric (partial scoring)
        → Update session state
        → Log audit event
      → IF complete: CompleteScreeningUseCase
        → GenerateSummaryUseCase
        → Update candidate state to 'pending_review'
      → Send response via TelegramBotService
```

**Dependencies**: C6, C7, C10, C11, C12, C13

---

## S2: Evaluation Service

| Attribute | Detail |
|---|---|
| **Name** | `EvaluationService` |
| **Location** | `application/evaluation/` |
| **Purpose** | Manage rubric-based evaluation logic and executive summary generation |
| **Responsibilities** | Score individual responses, accumulate partial scores in session state, generate final executive summary with evidence |

**Orchestration Flow**:
```
Candidate Response (during screening)
  → EvaluateResponseUseCase
    → Load rubric for campaign
    → Call OpenAI to score response against competency criteria
    → Extract verbatim quote as evidence
    → Update partial scores in session state

Screening Complete
  → GenerateSummaryUseCase
    → Load full conversation + all partial scores
    → Call OpenAI to generate executive summary
    → Validate every score has ≥1 cited quote
    → Determine recommendation level
    → Save ExecutiveSummary to DynamoDB
```

**Dependencies**: C2, C12, C13

---

## S3: Campaign Management Service

| Attribute | Detail |
|---|---|
| **Name** | `CampaignManagementService` |
| **Location** | `application/campaign/` |
| **Purpose** | Handle campaign CRUD operations and knowledge base management |
| **Responsibilities** | Create/update campaigns, generate Telegram links, manage knowledge base documents |

**Orchestration Flow**:
```
Create Campaign
  → CreateCampaignUseCase
    → Validate campaign data
    → Assign rubric (from templates or custom)
    → Generate unique Telegram link
    → Save to DynamoDB
    → Log audit event

Upload Knowledge Base Document (post-MVP)
  → ManageKnowledgeBaseUseCase
    → Parse document (PDF/DOCX/TXT)
    → Chunk text
    → Generate embeddings via OpenAI
    → Store in vector DB
```

**Dependencies**: C3, C12, C13, C15

---

## S4: HITL Review Service

| Attribute | Detail |
|---|---|
| **Name** | `HITLReviewService` |
| **Location** | `application/candidate/` |
| **Purpose** | Manage the human-in-the-loop review queue and decision workflow |
| **Responsibilities** | List candidates for review with filters, process approve/reject decisions, capture disagreements |

**Orchestration Flow**:
```
List for Review
  → ListCandidatesForReviewUseCase
    → Query DynamoDB with tenant scope + filters
    → Join with evaluation summaries
    → Return paginated results

Review Decision
  → ReviewCandidateUseCase
    → Validate candidate is in 'pending_review' state
    → Record decision (approve/reject)
    → IF decision differs from AI recommendation:
      → Capture disagreement reason
    → Update candidate state
    → Log audit event
```

**Dependencies**: C4, C9, C10, C13

---

## S5: Compliance Service

| Attribute | Detail |
|---|---|
| **Name** | `ComplianceService` |
| **Location** | `application/compliance/` |
| **Purpose** | Manage consent recording, audit logging, and data retention |
| **Responsibilities** | Record consent with immutable timestamp, append audit events, manage escalation logs |

**Orchestration Flow**:
```
All Operations (cross-cutting)
  → LogAuditEventUseCase
    → Append immutable event to DynamoDB audit table
    → Include: tenantId, eventType, entityId, timestamp, actorId

Consent
  → RecordConsentUseCase
    → Create immutable consent record
    → Log audit event
```

**Dependencies**: C5, C13

---

## Service Interaction Map

```
┌─────────────────────────────────────────────────────────┐
│                    ENTRY POINTS                         │
│  Telegram Webhook          Dashboard API Routes         │
│  (POST /api/telegram)      (GET/POST /api/*)            │
└──────────┬─────────────────────────┬────────────────────┘
           │                         │
           v                         v
┌──────────────────┐    ┌──────────────────────────────┐
│ S1: Screening    │    │ S3: Campaign Management      │
│ Orchestrator     │    │ S4: HITL Review Service      │
│                  │    │                              │
│ Uses:            │    │ Uses:                        │
│ - OpenAI Client  │    │ - DynamoDB Repos             │
│ - DynamoDB Repos │    │ - Knowledge Base (post-MVP)  │
│ - S2 (Evaluation)│    │                              │
│ - S5 (Compliance)│    │                              │
└──────────────────┘    └──────────────────────────────┘
           │                         │
           v                         v
┌──────────────────┐    ┌──────────────────────────────┐
│ S2: Evaluation   │    │ S5: Compliance Service       │
│ Service          │    │ (cross-cutting)              │
│                  │    │                              │
│ Uses:            │    │ Uses:                        │
│ - OpenAI Client  │    │ - DynamoDB Audit Table       │
│ - DynamoDB Repos │    │                              │
└──────────────────┘    └──────────────────────────────┘
```
