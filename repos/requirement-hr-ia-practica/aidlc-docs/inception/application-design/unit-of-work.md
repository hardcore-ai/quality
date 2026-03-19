# Units of Work — EntreVista AI

## Decomposition Strategy
**Approach**: MVP-first as single unit
**Rationale**: The entire MVP (foundation + conversation + evaluation + basic dashboard) is implemented as one unit to deliver a complete end-to-end flow. Post-MVP features are added as separate units.

---

## Unit 1: MVP Core

| Attribute | Detail |
|---|---|
| **Name** | `mvp-core` |
| **Priority** | 1 (implement first) |
| **Scope** | Complete end-to-end screening flow: project scaffold + DynamoDB + auth + Telegram bot + conversational screening + evaluation + recruiter dashboard + HITL review |
| **MVP** | Yes — this IS the MVP |

### What's Included

**Foundation (Shared Infrastructure)**:
- Next.js 16 project scaffold with DDD directory structure
- Dockerfile (multi-stage build for Next.js)
- Terraform modules: VPC, ECS (Fargate), ALB, DynamoDB (6 tables + 3 GSIs), Cognito (User Pool + App Client), ECR, CloudWatch
- DynamoDB repository implementations
- NextAuth.js with AWS Cognito provider
- Structured console logging (CloudWatch compatible)
- Shared types, utils, constants
- shadcn/ui component setup (Button, Card, Table, Dialog, Input, Select, Badge, etc.)
- Environment configuration (dev/staging/prod)
- GitHub Actions CI/CD pipeline (build → Docker → ECR → ECS deploy)

**Conversation Module (Domain + Application + Infrastructure)**:
- Conversation domain entities and rules (Conversation, Message, SessionState)
- StartScreening, ProcessMessage, CompleteScreening use cases
- grammY Telegram webhook handler (Next.js API route)
- OpenAI chat client (prompt building with session state + recent messages)
- Conversation state machine (onboarding → consent → verification → screening → closing)
- Basic requirements verification (hardcoded per campaign config)
- Anti-hallucination guardrails (system prompt confinement)
- Basic escalation handling (log and inform candidate, Level 1-2)

**Evaluation Module (Domain + Application)**:
- Evaluation domain entities (Rubric, Competency, Score, Evidence, ExecutiveSummary)
- EvaluateResponse use case (real-time partial scoring during screening)
- GenerateSummary use case (executive summary with cited evidence)
- Hardcoded rubric templates (BPO + Tech/SaaS)
- Recommendation engine (Highly Recommended / Recommended / Not Recommended)

**Campaign Module (Domain + Application — basic)**:
- Campaign domain entities and rules
- CreateCampaign, UpdateCampaign use cases
- Telegram link generation
- Campaign CRUD API routes
- Basic requirements configuration per campaign

**Candidate Module (Domain + Application — basic)**:
- Candidate domain entities and lifecycle state machine
- ReviewCandidate use case (approve/reject with disagreement capture)
- ListCandidatesForReview use case (with filters)
- Candidate state transitions

**Compliance Module (basic)**:
- ConsentRecord entity and recording
- AuditEvent entity and basic append-only logging
- Consent flow in Telegram onboarding

**Dashboard UI**:
- Login page (NextAuth.js)
- Campaign list and creation form
- HITL review queue (filterable, sortable)
- Candidate detail view (summary + scores + evidence + transcript)
- Approve/Reject with optional reason and disagreement capture
- Responsive layout (desktop-first, Tailwind CSS)

### What's NOT Included (deferred to later units)
- Re-engagement (24h/48h/72h auto-reminders)
- Campaign metrics dashboard
- NPS survey
- Knowledge base / RAG (documents use in-context loading instead)
- Configurable rubric editor (hardcoded templates only)
- Duplicate candidate detection
- Data retention / auto-purge
- Prometheus / Grafana / Loki
- Escalation Level 3 (dashboard alerts)

### Infrastructure (Terraform)

The MVP includes full Terraform IaC for all AWS resources:

```
terraform/
├── modules/
│   ├── vpc/          # VPC, public/private subnets, NAT gateway, security groups
│   ├── ecs/          # ECS cluster (Fargate), service, task definition, auto-scaling
│   ├── alb/          # Application Load Balancer, target group, listener rules
│   ├── dynamodb/     # 6 tables + 3 GSIs with tenant isolation
│   ├── cognito/      # User Pool, App Client, domain
│   ├── ecr/          # Container registry for Docker images
│   └── cloudwatch/   # Log groups, basic alarms
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

## Unit 2: Compliance & Audit

| Attribute | Detail |
|---|---|
| **Name** | `compliance-audit` |
| **Priority** | 2 |
| **Scope** | Full compliance implementation: immutable audit trail, data retention with auto-purge, escalation Level 3 with dashboard alerts, exportable audit reports |
| **MVP** | No — post-MVP |

### What's Included
- Immutable audit trail (DynamoDB with delete prevention via IAM)
- Data retention policy configuration (default 90 days)
- Automatic data purge (scheduled job / cron)
- Escalation Level 3: candidate requests human → dashboard alert with context
- Escalation log viewer in dashboard
- Audit trail viewer per candidate
- Exportable audit reports (PDF — S5 from PRD)

### Dependencies
- Requires Unit 1 (MVP Core) — builds on existing audit event logging

---

## Unit 3: Knowledge Base & RAG

| Attribute | Detail |
|---|---|
| **Name** | `knowledge-base-rag` |
| **Priority** | 3 |
| **Scope** | Full RAG implementation: document upload, chunking, embedding, vector DB, semantic retrieval during conversations |
| **MVP** | No — post-MVP |

### What's Included
- Document upload UI (PDF, DOCX, TXT) per campaign
- Document parsing and text extraction
- Text chunking strategy
- OpenAI embeddings generation
- Vector database integration (Pinecone or pgvector)
- Semantic similarity search during conversations
- Knowledge base isolation per campaign
- Replace in-context loading with RAG retrieval

### Dependencies
- Requires Unit 1 (MVP Core) — extends existing knowledge base infrastructure stub
- Requires Unit 2 (Compliance) — document operations need audit logging

---

## Unit 4: Advanced Features

| Attribute | Detail |
|---|---|
| **Name** | `advanced-features` |
| **Priority** | 4 |
| **Scope** | Re-engagement automation, campaign metrics, NPS, configurable rubric editor, duplicate detection, observability stack |
| **MVP** | No — post-MVP |

### What's Included
- **Re-engagement**: Automated 24h/48h/72h reminder messages via scheduled jobs
- **Campaign Metrics Dashboard**: Completion rate, approval rate, average score, NPS, escalation count, abandonment rate, abandonment by question
- **NPS Survey**: Post-screening satisfaction rating (1-5) + open text
- **Configurable Rubric Editor**: Full CRUD UI for rubrics with competencies, weights, criteria per level
- **Duplicate Detection**: Detect same candidate applying multiple times to same tenant
- **Observability**: Prometheus metrics, Grafana dashboards, Loki log aggregation

### Dependencies
- Requires Unit 1 (MVP Core) — extends all modules
- Requires Unit 2 (Compliance) — re-engagement and metrics need audit trails

---

## Code Organization (Greenfield)

Since all units contribute to the same Next.js modular monolith, the directory structure is shared across all units. Each unit adds to the existing DDD structure:

```
src/
├── domain/           # Unit 1 creates all domain modules
├── application/      # Unit 1 creates core use cases; Units 2-4 add more
├── infrastructure/   # Unit 1 creates core infra; Unit 3 adds vector DB
├── app/              # Unit 1 creates routes + dashboard; Units 2-4 extend
└── shared/           # Unit 1 creates; all units may extend
```

**Implementation rule**: Each unit's code changes are additive — they extend existing modules without breaking previous units.
