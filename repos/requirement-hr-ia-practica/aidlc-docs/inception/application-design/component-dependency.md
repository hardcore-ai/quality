# Component Dependencies — EntreVista AI

## Dependency Rules

1. **Domain layer** has ZERO external dependencies — pure business logic
2. **Application layer** depends on domain entities and repository interfaces (not implementations)
3. **Infrastructure layer** implements repository interfaces and integrates with external systems
4. **App layer (Next.js)** depends on application use cases only — never directly on infrastructure
5. **Dependency injection** via constructor injection — infrastructure implementations injected into application use cases

---

## Dependency Matrix

| Component | Depends On | Depended By |
|---|---|---|
| **C1** Conversation Domain | — | C6, C7 |
| **C2** Evaluation Domain | — | C6, C7, C9 |
| **C3** Campaign Domain | — | C8 |
| **C4** Candidate Domain | — | C9 |
| **C5** Compliance Domain | — | C6, C10 |
| **C6** Conversation App | C1, C2, C5, repos | C11 (Telegram), API routes |
| **C7** Evaluation App | C1, C2, repos | C6, API routes |
| **C8** Campaign App | C3, repos | API routes |
| **C9** Candidate App | C2, C4, repos | API routes |
| **C10** Compliance App | C5, repos | C6, C8, C9 |
| **C11** Telegram Infra | C6 | Next.js API route (webhook) |
| **C12** OpenAI Infra | — | C6, C7, C15 |
| **C13** DynamoDB Infra | Domain entities | All application components (via repos) |
| **C14** Auth Infra | — | Next.js middleware, API routes |
| **C15** Knowledge Base Infra | C12 | C8 |
| **C16** Dashboard UI | API routes | — (user-facing) |

---

## Communication Patterns

### Pattern 1: Telegram → Screening (Synchronous)

```
Telegram Bot API
  → POST /api/telegram/webhook
    → TelegramWebhookHandler.handleUpdate()
      → ScreeningOrchestrator.processMessage()
        → ConversationRepository.findByTelegramUser()
        → OpenAIChatClient.generateResponse()
        → EvaluationService.evaluateResponse()
        → ConversationRepository.updateSessionState()
        → ComplianceService.logAuditEvent()
      → TelegramBotService.sendMessage()
    ← 200 OK to Telegram
```

**Latency budget**: < 10 seconds total
- DynamoDB read: ~50ms
- OpenAI chat completion: ~3-8 seconds
- DynamoDB writes: ~50ms
- Telegram send: ~100ms

### Pattern 2: Dashboard → Review (Request/Response)

```
Browser
  → GET /api/candidates?status=pending_review&campaign=X
    → withAuth(middleware)
      → getTenantId(session)
      → ListCandidatesForReviewUseCase.execute()
        → CandidateRepository.findForReview(tenantId, filters)
        → EvaluationRepository.findByConversation() (for each)
      ← JSON response
  ← Rendered in ReviewQueue component
```

### Pattern 3: HITL Decision (Command)

```
Browser
  → POST /api/candidates/:id/review
    → withAuth(middleware)
      → ReviewCandidateUseCase.execute()
        → CandidateRepository.findById()
        → Validate state = 'pending_review'
        → CandidateRepository.updateState()
        → ComplianceService.logAuditEvent()
      ← 200 OK
  ← UI updates candidate status
```

### Pattern 4: Campaign CRUD (Command/Query)

```
Browser
  → POST /api/campaigns
    → withAuth(middleware)
      → CreateCampaignUseCase.execute()
        → CampaignRules.canActivate()
        → CampaignRules.generateTelegramLink()
        → CampaignRepository.save()
        → ComplianceService.logAuditEvent()
      ← Campaign JSON
  ← UI updates campaign list
```

---

## Data Flow Diagram

```
                    ┌──────────────┐
                    │   Candidate  │
                    │  (Telegram)  │
                    └──────┬───────┘
                           │ messages
                           v
              ┌────────────────────────┐
              │   Telegram Bot API     │
              │   (Webhook)            │
              └────────────┬───────────┘
                           │
                           v
┌──────────────────────────────────────────────────────┐
│                   Next.js Application                │
│                                                      │
│  ┌─────────────┐    ┌──────────────┐                 │
│  │ /api/telegram│    │ /api/*       │◄── Recruiter   │
│  │ webhook      │    │ dashboard    │    (Browser)    │
│  └──────┬───────┘    └──────┬───────┘                 │
│         │                   │                         │
│         v                   v                         │
│  ┌──────────────────────────────────────┐             │
│  │        Application Layer             │             │
│  │  Screening    Campaign    HITL       │             │
│  │  Orchestrator Management  Review     │             │
│  └──────────┬───────────────┬───────────┘             │
│             │               │                         │
│             v               v                         │
│  ┌──────────────────────────────────────┐             │
│  │        Domain Layer                  │             │
│  │  Conversation  Evaluation  Campaign  │             │
│  │  Candidate     Compliance            │             │
│  └──────────┬───────────────────────────┘             │
│             │                                         │
│             v                                         │
│  ┌──────────────────────────────────────┐             │
│  │        Infrastructure Layer          │             │
│  │  DynamoDB   OpenAI   Auth   Logging  │             │
│  └──────┬────────┬───────────────────────┘            │
└─────────┼────────┼───────────────────────────────────┘
          │        │
          v        v
   ┌──────────┐  ┌──────────┐
   │ DynamoDB │  │ OpenAI   │
   │ (AWS)    │  │ API      │
   └──────────┘  └──────────┘
```

---

## DynamoDB Table Design (High-Level)

| Table | Partition Key | Sort Key | Purpose |
|---|---|---|---|
| `Conversations` | `tenantId` | `conversationId` | Screening sessions + messages + session state |
| `Campaigns` | `tenantId` | `campaignId` | Campaign configuration |
| `Candidates` | `tenantId` | `candidateId` | Candidate profiles + lifecycle state |
| `Evaluations` | `tenantId` | `conversationId` | Executive summaries + scores + evidence |
| `AuditEvents` | `tenantId` | `timestamp#eventId` | Immutable audit trail (append-only) |
| `Consent` | `tenantId` | `candidateId` | Consent records (immutable) |

**GSI (Global Secondary Indexes)**:
- `Candidates-ByCampaign`: PK=`campaignId`, SK=`candidateState` — for review queue filtering
- `Candidates-ByTelegram`: PK=`telegramUserId`, SK=`tenantId` — for duplicate detection
- `Conversations-ByTelegram`: PK=`telegramUserId#campaignId` — for session lookup
