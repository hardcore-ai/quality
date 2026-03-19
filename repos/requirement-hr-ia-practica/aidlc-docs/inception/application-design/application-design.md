# Application Design — EntreVista AI (Consolidated)

## Architecture Overview

| Dimension | Decision |
|---|---|
| **Architecture Style** | Domain-Driven Design (DDD) layered architecture |
| **Deployment Model** | Modular monolith — single Next.js 16 application |
| **Layer Structure** | `src/domain/` → `src/application/` → `src/infrastructure/` → `src/app/` |
| **Database** | AWS DynamoDB (managed NoSQL, partition key-based tenant isolation) |
| **LLM** | OpenAI GPT-4o (chat + embeddings) |
| **Bot Framework** | grammY (TypeScript, webhook via Next.js API route) |
| **Auth** | AWS Cognito (OAuth 2.0 / OIDC) |
| **Frontend** | React Server Components + Tailwind CSS + shadcn/ui |
| **Deploy** | AWS ECS (Docker) + ALB |
| **IaC** | Terraform (ECS, DynamoDB, ALB, Cognito, VPC, ECR) |

---

## Layer Responsibilities

| Layer | What it contains | Dependency rule |
|---|---|---|
| **Domain** | Entities, value objects, business rules | ZERO external dependencies |
| **Application** | Use cases, orchestration | Depends on domain + repository interfaces |
| **Infrastructure** | DB repos, API clients, auth, logging | Implements interfaces, integrates externals |
| **App (Next.js)** | API routes, UI pages, middleware | Calls application use cases only |
| **Shared** | Types, utils, constants | Used by all layers |

---

## Module Decomposition

### 5 Domain Modules

| Module | Domain Entities | Application Use Cases | Infrastructure |
|---|---|---|---|
| **Conversation** | Conversation, Message, SessionState | StartScreening, ProcessMessage, HandleEscalation, ResumeSession, CompleteScreening | Telegram (grammY), OpenAI Chat |
| **Evaluation** | Rubric, Competency, Score, Evidence, ExecutiveSummary | EvaluateResponse, GenerateSummary, GetEvaluationDetail | OpenAI Chat |
| **Campaign** | Campaign, CampaignConfig, BasicRequirement | CreateCampaign, UpdateCampaign, GetCampaignMetrics, ManageKnowledgeBase | Knowledge Base (post-MVP) |
| **Candidate** | Candidate, CandidateState, ReviewDecision | ListCandidatesForReview, ReviewCandidate | — |
| **Compliance** | ConsentRecord, AuditEvent | RecordConsent, LogAuditEvent, GetAuditTrail | — |

### Cross-Cutting Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| DynamoDB Repos | AWS DynamoDB | Persistence for all entities |
| OpenAI Client | OpenAI GPT-4o | Chat completions + embeddings |
| Telegram Handler | grammY | Bot webhook + message sending |
| Auth | AWS Cognito + NextAuth.js (Cognito adapter) | Session management, tenant extraction, JWT validation |
| Logging | Structured console logs | Observability (CloudWatch for MVP) |

---

## 5 Services (Application Layer)

| Service | Entry Point | Core Flow |
|---|---|---|
| **S1: Screening Orchestrator** | Telegram webhook | Message → session lookup → OpenAI → evaluation → response |
| **S2: Evaluation Service** | Called by S1 | Score response → extract evidence → generate summary |
| **S3: Campaign Management** | Dashboard API | CRUD campaigns, assign rubrics, generate Telegram links |
| **S4: HITL Review** | Dashboard API | List pending → review detail → approve/reject |
| **S5: Compliance** | Cross-cutting | Consent recording, audit event logging |

---

## Key Design Decisions

### 1. Session State Object (Conversation Context)

Instead of sending full message history to OpenAI on every turn, the system maintains a structured `SessionState` object in DynamoDB:

```typescript
interface SessionState {
  currentPhase: 'onboarding' | 'consent' | 'verification' | 'screening' | 'closing';
  competenciesCovered: string[];
  partialScores: Record<string, number>;
  questionsAsked: number;
  followUpsAsked: number;
  escalationLevel: 0 | 1 | 2 | 3;
  lastActivityAt: Date;
}
```

On each message: `sessionState + last N messages + system prompt + rubric → OpenAI → response`

### 2. Telegram Webhook (In-Process)

grammY handles Telegram updates directly within the Next.js API route. No separate process or job queue. Acceptable for MVP volume (100-500 candidates, <50 concurrent).

### 3. DynamoDB Table Design

6 tables with partition key = `tenantId` for multi-tenant isolation. 3 GSIs for query patterns (review queue, duplicate detection, session lookup).

### 4. MVP Simplifications

| Full Design | MVP Implementation |
|---|---|
| Configurable rubric editor | Hardcoded BPO + Tech templates |
| Vector DB + RAG | In-context loading (docs in prompt) |
| Re-engagement (24/48/72h) | Candidate can resume, no auto-reminders |
| Campaign metrics dashboard | Query DynamoDB directly |
| NPS survey | Deferred |
| Prometheus + Grafana + Loki | CloudWatch logs + basic metrics |
| Duplicate detection | Deferred |

---

## Directory Structure

```
├── terraform/                 # Infrastructure as Code
│   ├── modules/
│   │   ├── ecs/               # ECS cluster, service, task definition
│   │   ├── dynamodb/          # DynamoDB tables + GSIs
│   │   ├── alb/               # Application Load Balancer
│   │   ├── cognito/           # Cognito User Pool + App Client
│   │   ├── vpc/               # VPC, subnets, security groups
│   │   ├── ecr/               # ECR repository
│   │   └── cloudwatch/        # Log groups, alarms
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
│
├── Dockerfile                 # Multi-stage build for Next.js
├── docker-compose.yml         # Local development
│
src/
├── domain/
│   ├── conversation/
│   │   ├── entities/          # Conversation, Message, SessionState
│   │   ├── rules/             # ConversationRules
│   │   └── ports/             # ConversationRepository (interface)
│   ├── evaluation/
│   │   ├── entities/          # Rubric, Competency, Score, Evidence, ExecutiveSummary
│   │   ├── rules/             # EvaluationRules
│   │   └── ports/             # EvaluationRepository (interface)
│   ├── campaign/
│   │   ├── entities/          # Campaign, CampaignConfig
│   │   ├── rules/             # CampaignRules
│   │   └── ports/             # CampaignRepository (interface)
│   ├── candidate/
│   │   ├── entities/          # Candidate, CandidateState, ReviewDecision
│   │   ├── rules/             # CandidateRules
│   │   └── ports/             # CandidateRepository (interface)
│   └── compliance/
│       ├── entities/          # ConsentRecord, AuditEvent
│       ├── rules/             # ComplianceRules
│       └── ports/             # AuditEventRepository, ConsentRepository (interfaces)
│
├── application/
│   ├── conversation/          # StartScreening, ProcessMessage, HandleEscalation, etc.
│   ├── evaluation/            # EvaluateResponse, GenerateSummary
│   ├── campaign/              # CreateCampaign, UpdateCampaign, GetMetrics
│   ├── candidate/             # ListForReview, ReviewCandidate
│   └── compliance/            # RecordConsent, LogAuditEvent, GetAuditTrail
│
├── infrastructure/
│   ├── telegram/              # grammY webhook handler, bot service
│   ├── openai/                # Chat client, embedding client
│   ├── dynamodb/              # Repository implementations (6 tables)
│   ├── auth/                  # NextAuth.js config, middleware
│   ├── logging/               # Structured logger
│   └── knowledge-base/        # Document processing (post-MVP)
│
├── app/
│   ├── api/
│   │   ├── telegram/webhook/  # POST handler for grammY
│   │   ├── campaigns/         # CRUD endpoints
│   │   ├── candidates/        # Query + review endpoints
│   │   ├── evaluations/       # Query endpoints
│   │   └── auth/[...nextauth]/ # NextAuth routes
│   ├── (dashboard)/
│   │   ├── campaigns/         # Campaign management pages
│   │   ├── review/            # HITL review queue + detail
│   │   ├── candidates/        # Candidate list
│   │   └── layout.tsx         # Dashboard layout with nav
│   ├── layout.tsx             # Root layout
│   └── page.tsx               # Landing / login redirect
│
└── shared/
    ├── types/                 # Shared TypeScript types
    ├── utils/                 # Date, string, ID generation utils
    └── constants/             # App-wide constants
```

---

## Detailed References

- **Components**: See [components.md](components.md)
- **Method Signatures**: See [component-methods.md](component-methods.md)
- **Services**: See [services.md](services.md)
- **Dependencies**: See [component-dependency.md](component-dependency.md)
