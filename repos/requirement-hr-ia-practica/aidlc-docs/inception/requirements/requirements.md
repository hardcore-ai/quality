# Requirements Document — EntreVista AI

## Intent Analysis

| Dimension | Value |
|---|---|
| **User Request** | Build EntreVista AI — an agentic interview platform for LATAM that conducts conversational screenings via Telegram for high-volume hiring companies |
| **Request Type** | New Project (Greenfield) |
| **Scope Estimate** | Cross-system — 5 modules (Conversational Engine, Evaluation Engine, Recruiter Dashboard, Compliance, Candidate Management) |
| **Complexity Estimate** | Complex — AI agentic system, multi-tenant, Telegram integration, compliance requirements |
| **Depth Level** | Comprehensive |

---

## 1. Tech Stack Decisions

| Layer | Technology | Notes |
|---|---|---|
| **Fullstack Framework** | Next.js 16 (React + TypeScript) | Single deployable unit for frontend + API routes |
| **Styling** | Tailwind CSS + shadcn/ui | Utility-first CSS + accessible component library |
| **LLM Provider** | OpenAI GPT-4o / GPT-4.1 | Both chat completions and embeddings |
| **Database** | AWS DynamoDB | Managed NoSQL, shared tables with partition key-based tenant isolation |
| **Vector Database** | Pinecone, Weaviate, or compatible | Embedding-based retrieval for knowledge base RAG |
| **Cloud Provider** | AWS | Primary infrastructure |
| **Authentication** | AWS Cognito | OAuth 2.0 / OIDC for recruiter dashboard |
| **Telegram Bot** | grammY (TypeScript framework) | Webhook-based integration with Next.js API routes |
| **Monitoring** | Prometheus + Grafana + Loki | Open-source observability stack |
| **CI/CD** | GitHub Actions | Automated build, test, deploy pipelines |
| **Architecture** | Modular Monolith | Single deployment with internal module boundaries |
| **Deployment** | AWS ECS (Docker containers) | Containerized deployment behind ALB |
| **IaC** | Terraform | ECS, DynamoDB, ALB, Cognito, VPC, and all AWS resources |
| **Language Convention** | English (code + docs), Spanish (product UI) | |
| **Security Extensions** | Disabled | MVP/prototype phase — security rules not enforced as blocking constraints |

**Note**: Previous decisions overridden:
- ~~Vercel~~ → AWS ECS (Docker) + ALB
- ~~NextAuth.js credentials (MVP)~~ → AWS Cognito from day one
- ~~Tailwind only~~ → Tailwind + shadcn/ui
- Infrastructure managed via Terraform

---

## 2. Functional Requirements

### FR-01: Telegram Bot — Onboarding and Consent (Must Have — M1)

| ID | Requirement |
|---|---|
| FR-01.1 | The system shall provide a Telegram bot accessible via a shareable link generated per campaign |
| FR-01.2 | On first interaction, the bot shall display a clear disclaimer identifying itself as AI, explaining the purpose, data usage, and the candidate's right to opt out |
| FR-01.3 | The bot shall require explicit affirmative consent ("Sí" or equivalent) before initiating any evaluative interaction |
| FR-01.4 | Consent shall be recorded with an immutable timestamp |
| FR-01.5 | If consent is denied, the bot shall thank the candidate and end the session without penalization |
| FR-01.6 | If asked "¿Eres IA?" at any point, the bot shall confirm its AI nature truthfully |
| FR-01.7 | All bot messages shall be in Spanish neutro (neutral Spanish) |

### FR-02: Basic Requirements Verification (Must Have — M2)

| ID | Requirement |
|---|---|
| FR-02.1 | After consent, the bot shall verify configurable basic requirements (availability, location/zone, documentation) |
| FR-02.2 | Requirements shall be configurable per campaign by the operator |
| FR-02.3 | If a candidate fails a mandatory requirement, the bot shall inform them and end the session gracefully |
| FR-02.4 | Verification results shall be stored as structured data per candidate |

### FR-03: Conversational Screening Engine (Must Have — M3)

| ID | Requirement |
|---|---|
| FR-03.1 | The agent shall conduct a screening interview of 3-5 competency-based questions per session |
| FR-03.2 | The agent shall generate dynamic follow-up questions (repreguntas) based on the candidate's responses |
| FR-03.3 | The agent shall use the campaign's rubric to guide question selection and depth |
| FR-03.4 | The agent shall maintain conversational context throughout the entire session |
| FR-03.5 | Sessions shall have states: `active`, `paused`, `abandoned`, `completed` |
| FR-03.6 | The agent shall handle candidate responses in Spanish; if a candidate responds in English, it shall continue without breaking |
| FR-03.7 | There shall be no time limit on the interview — the candidate proceeds at their own pace |
| FR-03.8 | The target screening duration is 15-25 minutes |

### FR-04: Configurable Evaluation Rubrics (Must Have — M4)

| ID | Requirement |
|---|---|
| FR-04.1 | Operators shall be able to create and edit rubrics with competencies, weights, and criteria per level (1-5) |
| FR-04.2 | The system shall provide default rubric templates for BPO and Tech/SaaS roles |
| FR-04.3 | Each rubric competency shall have defined evaluation criteria per score level |
| FR-04.4 | Rubrics shall be assignable to campaigns |

### FR-05: Executive Summary with Cited Evidence (Must Have — M5)

| ID | Requirement |
|---|---|
| FR-05.1 | Upon screening completion, the system shall generate an executive summary per candidate |
| FR-05.2 | The summary shall include: global score, per-competency scores, textual evidence (verbatim quotes from the transcript), key signals, and a recommendation level |
| FR-05.3 | Every score shall be linked to at least one verbatim quote from the candidate's transcript |
| FR-05.4 | The summary shall include a recommendation: Highly Recommended, Recommended, Not Recommended |
| FR-05.5 | Summaries shall be generated automatically after screening completion |

### FR-06: Recruiter Dashboard with HITL Review Queue (Must Have — M6)

| ID | Requirement |
|---|---|
| FR-06.1 | The dashboard shall display a review queue of completed screenings pending human review |
| FR-06.2 | The queue shall support filtering by campaign, recommendation level, score range, and date |
| FR-06.3 | The queue shall support sorting by score, date, and recommendation |
| FR-06.4 | Each candidate detail view shall show: executive summary, per-competency scores with cited evidence, full transcript |
| FR-06.5 | The recruiter shall be able to mark each candidate as "Approved" or "Rejected" with an optional reason |
| FR-06.6 | The dashboard shall capture disagreement reasons when the recruiter's decision differs from the AI recommendation |
| FR-06.7 | Dashboard data updates via manual page refresh (no real-time for MVP) |

### FR-07: Anti-Hallucination Guardrails and Escalation (Must Have — M7)

| ID | Requirement |
|---|---|
| FR-07.1 | The agent shall be confined to the campaign's knowledge base for factual information |
| FR-07.2 | When asked about salary, benefits, or company policies not in the knowledge base, the agent shall respond: "No tengo esa información. El equipo de reclutamiento te dará los detalles en la siguiente etapa." |
| FR-07.3 | The agent shall never speculate, invent information, or make contractual promises |
| FR-07.4 | If a candidate requests human contact, the agent shall: acknowledge the request, notify the recruiter via dashboard alert, preserve session context |
| FR-07.5 | Escalation levels: Level 1 (info not available — log and continue), Level 2 (repeated requests — suggest human follow-up), Level 3 (explicit human request — create alert) |
| FR-07.6 | All escalated questions shall be logged for knowledge base improvement |

### FR-08: Campaign Knowledge Base (Must Have — M8)

| ID | Requirement |
|---|---|
| FR-08.1 | Each campaign shall have an associated knowledge base |
| FR-08.2 | Operators shall be able to upload documents (PDF, DOCX, TXT) to the knowledge base |
| FR-08.3 | Documents shall be processed, chunked, and embedded into the vector database for RAG retrieval |
| FR-08.4 | The agent shall query the knowledge base before responding to factual questions |
| FR-08.5 | Knowledge bases shall be isolated per campaign (no cross-campaign leakage) |

### FR-09: Abandonment and Re-engagement (Must Have — M9)

| ID | Requirement |
|---|---|
| FR-09.1 | If a candidate is inactive for 5 minutes during a session, the bot shall send a supportive message: "Tómate tu tiempo" |
| FR-09.2 | At 24 hours of inactivity, the bot shall send a reminder to resume |
| FR-09.3 | At 48 hours, the bot shall send a final reminder |
| FR-09.4 | At 72 hours without response, the session shall be marked as "Abandoned" without penalization |
| FR-09.5 | When a candidate resumes, session context shall be fully preserved |
| FR-09.6 | The dashboard shall show at which question abandonments occur |

### FR-10: Post-Screening Satisfaction Survey (Must Have — M10)

| ID | Requirement |
|---|---|
| FR-10.1 | After screening completion, the bot shall ask the candidate for a satisfaction rating (1-5) |
| FR-10.2 | An optional open text field shall be available for additional feedback |
| FR-10.3 | NPS data shall be aggregated and displayed in campaign metrics |

### FR-11: Campaign Management

| ID | Requirement |
|---|---|
| FR-11.1 | Operators shall be able to create, edit, activate, deactivate, and archive campaigns |
| FR-11.2 | Each campaign shall have: name, role description, assigned rubric, knowledge base, Telegram link, status |
| FR-11.3 | The system shall generate a unique Telegram bot link per campaign |
| FR-11.4 | Campaign metrics shall include: candidates started, completed, completion rate, approval rate, average score, NPS, escalation count, abandonment rate |

### FR-12: Candidate Lifecycle Management

| ID | Requirement |
|---|---|
| FR-12.1 | Each candidate shall have a profile with basic professional data (no sensitive/protected information) |
| FR-12.2 | Candidate states: `Initiated` → `In Screening` → `Completed` → `Pending Review` → `Approved` / `Rejected` |
| FR-12.3 | The system shall detect multiple applications from the same candidate to the same client |
| FR-12.4 | Candidate data shall be segregated by tenant (tenant_id) |

---

## 3. Non-Functional Requirements

### NFR-01: Performance

| ID | Requirement |
|---|---|
| NFR-01.1 | Bot response latency shall be < 10 seconds for conversational messages (including LLM processing) |
| NFR-01.2 | Dashboard page load shall be < 3 seconds |
| NFR-01.3 | Executive summary generation shall complete within 30 seconds of screening completion |
| NFR-01.4 | The system shall support up to 50 concurrent Telegram conversations for MVP |

### NFR-02: Availability and Reliability

| ID | Requirement |
|---|---|
| NFR-02.1 | The Telegram bot shall be available 24/7 (target: 99.5% uptime for MVP) |
| NFR-02.2 | Dashboard availability target: 99% during business hours (8am-8pm COT) |
| NFR-02.3 | Session state shall be persisted — no data loss on server restart |

### NFR-03: Data Privacy and Compliance

| ID | Requirement |
|---|---|
| NFR-03.1 | No biometric data shall be collected (no facial recognition, voice analysis, or emotion detection) |
| NFR-03.2 | No protected/sensitive personal data shall be stored beyond professional information |
| NFR-03.3 | Default data retention: 90 days with automatic purge |
| NFR-03.4 | All data shall be segregated per tenant — no cross-tenant data access |
| NFR-03.5 | Consent records shall be immutable |
| NFR-03.6 | All candidate interactions shall be logged as immutable audit trails |

### NFR-04: Scalability

| ID | Requirement |
|---|---|
| NFR-04.1 | MVP target: 100-500 candidates over 90 days |
| NFR-04.2 | Architecture shall allow horizontal scaling for future growth |
| NFR-04.3 | MongoDB collections shall be designed with tenant_id indexing for efficient multi-tenant queries |

### NFR-05: Usability

| ID | Requirement |
|---|---|
| NFR-05.1 | Time-to-first-value for operators: < 48 hours (from signup to first screening) |
| NFR-05.2 | Dashboard shall be responsive (desktop-first, mobile-friendly) |
| NFR-05.3 | All user-facing content in Spanish neutro |

### NFR-06: Observability

| ID | Requirement |
|---|---|
| NFR-06.1 | Structured logging with Loki integration |
| NFR-06.2 | Application metrics exposed via Prometheus |
| NFR-06.3 | Grafana dashboards for key operational metrics |
| NFR-06.4 | Logs shall include: timestamp, correlation/request ID, log level, and message |

### NFR-07: Deployment and Operations

| ID | Requirement |
|---|---|
| NFR-07.1 | CI/CD via GitHub Actions (build Docker image, push to ECR, deploy to ECS) |
| NFR-07.2 | Deployment to AWS ECS (Fargate) behind Application Load Balancer |
| NFR-07.3 | Modular monolith deployed as a Docker container running Next.js |
| NFR-07.4 | Environment separation: development, staging, production |
| NFR-07.5 | All AWS infrastructure managed via Terraform (ECS, DynamoDB, ALB, Cognito, VPC, ECR, CloudWatch) |

---

## 4. Constraints

| ID | Constraint |
|---|---|
| C-01 | MVP canal: Telegram only (web fallback is Could Have) |
| C-02 | MVP language: Spanish neutro only |
| C-03 | MVP segments: BPO + Tech/SaaS |
| C-04 | MVP geography: Colombia, Mexico, Argentina |
| C-05 | No ATS integrations in MVP |
| C-06 | No auto-reject or auto-advance — HITL mandatory |
| C-07 | No video/voice analysis — NEVER |
| C-08 | Business model: per successful hire (pricing TBD) |

---

## 5. Assumptions

| ID | Assumption |
|---|---|
| A-01 | Target candidates (18-35, BPO/tech) have Telegram access and are comfortable using it |
| A-02 | OpenAI GPT-4o/4.1 provides sufficient reasoning capability for dynamic follow-up questions |
| A-03 | MongoDB's document model is suitable for storing conversational transcripts and evaluation data |
| A-04 | grammY framework supports all required Telegram Bot API features (webhooks, message handling, session management) |
| A-05 | 100-500 candidates is sufficient volume to validate the product hypothesis |
| A-06 | Recruiters will adopt a new dashboard tool if it demonstrably reduces their workload |

---

## 6. Out of Scope (MVP)

- ATS integrations (Buk, Workday, SAP)
- Practical task validation (coding challenges, etc.)
- Multi-language support (Portuguese)
- Native mobile application
- Video/voice interviews
- Auto-reject without human review
- Marketplace of shared rubrics
- WhatsApp channel
- Advanced analytics / predictive insights

---

## 7. Architectural Considerations

### 7.1 Modular Monolith Structure

The Next.js 16 application shall be organized with clear internal module boundaries:

```
src/
  modules/
    conversation/    # Telegram bot + agentic engine
    evaluation/      # Rubrics + scoring + summary generation
    dashboard/       # Recruiter UI + campaign management
    compliance/      # Consent + audit logs + data retention
    candidates/      # Candidate lifecycle + profiles
  shared/            # Cross-cutting: auth, DB, logging, types
```

### 7.2 Key Integration Points

- **Telegram ↔ Conversation Module**: grammY webhook handler → Next.js API route
- **Conversation ↔ Evaluation**: Real-time scoring as candidate responds
- **Conversation ↔ Compliance**: Consent recording, transcript logging
- **Evaluation ↔ Dashboard**: Summary display in review queue
- **Knowledge Base ↔ Vector DB**: RAG retrieval during conversations
- **Auth ↔ Cognito**: JWT validation for dashboard access

### 7.3 Data Model Considerations

- MongoDB collections with `tenantId` field on all documents
- Conversations stored as documents with embedded message arrays
- Evaluations stored as separate documents linked to conversation IDs
- Immutable audit log collection (append-only)
- Vector embeddings stored in dedicated vector database (not MongoDB)
