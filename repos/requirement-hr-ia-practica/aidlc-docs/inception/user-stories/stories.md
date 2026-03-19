# User Stories — EntreVista AI

## Story Format
- **Breakdown**: Epic-Based (high-level epics decomposed into sub-stories)
- **Granularity**: Medium (~30-50 stories)
- **Acceptance Criteria**: Given/When/Then (BDD)
- **Prioritization**: MoSCoW (from PRD)
- **Personas**: María (Candidate), Carlos (Recruiter), Laura (Director/VP TA), Andrés (Head of People)

---

# Epic 1: Conversational Screening via Telegram

**Description**: Enable candidates to complete AI-driven screening interviews through Telegram with onboarding, consent, requirements verification, and dynamic competency-based questioning.

**PRD References**: M1, M2, M3, M7, M9 | FR-01, FR-02, FR-03, FR-07, FR-09

---

## US-1.1: Bot Onboarding and AI Disclosure

**As** María (Candidate),
**I want** to receive a clear introduction identifying the interviewer as AI, explaining the process and my rights,
**So that** I can make an informed decision about participating.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María clicks the campaign Telegram link, When the bot initiates the conversation, Then the first message includes: AI identity disclosure, purpose of the interview, how data will be used, and the right to opt out
- Given María asks "¿Eres IA?" at any point during the conversation, When the bot processes the question, Then it confirms truthfully that it is an AI assistant
- Given all messages are in Spanish neutro, When María reads the onboarding, Then the language is clear, professional, and free of regional slang

---

## US-1.2: Consent Collection

**As** María (Candidate),
**I want** to explicitly consent before any evaluative questions begin,
**So that** my participation is voluntary and recorded.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the onboarding message has been displayed, When María is asked for consent, Then she must provide an affirmative response ("Sí" or equivalent) before proceeding
- Given María provides consent, When the system records it, Then an immutable timestamp is stored in the audit log
- Given María declines consent, When she responds negatively, Then the bot thanks her, ends the session gracefully, and no evaluative data is recorded
- Given María has not yet consented, When the bot processes messages, Then no screening or evaluative questions are asked

---

## US-1.3: Basic Requirements Verification

**As** María (Candidate),
**I want** to verify that I meet the basic requirements for the position early in the process,
**So that** I don't waste time on a role I'm not eligible for.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María has consented, When the bot begins verification, Then it asks about configurable basic requirements (availability, location/zone, documentation)
- Given María meets all requirements, When verification is complete, Then the bot proceeds to the screening phase
- Given María fails a mandatory requirement, When the bot detects the disqualification, Then it informs her respectfully and ends the session without penalization
- Given the requirements are configurable, When Carlos sets up a campaign, Then he can define which basic requirements to verify

---

## US-1.4: Competency-Based Screening Questions

**As** María (Candidate),
**I want** to answer competency-based questions in a natural conversational format,
**So that** I can demonstrate my abilities through real examples rather than multiple-choice answers.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María has passed requirements verification, When the screening begins, Then the bot asks 3-5 competency-based questions guided by the campaign's rubric
- Given María provides a response, When the bot processes it, Then it may generate a contextual follow-up question (repregunta) to deepen understanding
- Given the conversation is in progress, When María responds, Then the bot maintains full conversational context from all previous messages
- Given there is no time limit, When María takes a break mid-answer, Then the bot waits patiently without pressuring her

---

## US-1.5: Anti-Hallucination and Out-of-Scope Handling

**As** María (Candidate),
**I want** to receive honest responses when I ask about salary, benefits, or company policies,
**So that** I'm not misled by fabricated information.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María asks about salary or benefits not in the knowledge base, When the bot processes the question, Then it responds: "No tengo esa información. El equipo de reclutamiento te dará los detalles en la siguiente etapa."
- Given María asks a question outside the bot's scope, When the bot cannot answer from the knowledge base, Then it admits it doesn't have the information and logs the question for knowledge base improvement
- Given María insists on getting an answer the bot doesn't have, When she asks repeatedly, Then the bot escalates (Level 2) suggesting human follow-up without inventing information
- Given the bot is responding to any factual query, When generating a response, Then it never speculates, invents data, or makes contractual promises

---

## US-1.6: Escalation to Human

**As** María (Candidate),
**I want** to be able to request speaking with a real person,
**So that** I can get answers the AI cannot provide.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María explicitly requests to speak with a human, When the bot detects the request, Then it acknowledges, notifies the recruiter via dashboard alert, and preserves session context
- Given an escalation alert is created, When Carlos views the dashboard, Then he sees the alert with full conversation context and pending questions
- Given the escalation has been created, When María receives confirmation, Then the bot informs her that a recruiter will follow up and her progress is saved

---

## US-1.7: Session Pause and Re-engagement

**As** María (Candidate),
**I want** to pause the interview and come back later without losing my progress,
**So that** I can complete the screening at my convenience.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María stops responding for 5 minutes, When the bot detects inactivity, Then it sends a supportive message: "Tómate tu tiempo"
- Given María has been inactive for 24 hours, When the timer triggers, Then the bot sends a friendly reminder to resume
- Given María has been inactive for 48 hours, When the timer triggers, Then the bot sends a final reminder
- Given María has been inactive for 72 hours without response, When the timer triggers, Then the session is marked as "Abandoned" without penalization
- Given María resumes after any pause, When she sends a message, Then the bot restores full session context and continues from where she left off

---

## US-1.8: Screening Completion and Closing

**As** María (Candidate),
**I want** to know clearly when the interview is over and what happens next,
**So that** I'm not left wondering about the process.

**Priority**: Must Have

**Acceptance Criteria**:
- Given all screening questions and follow-ups are complete, When the bot concludes the interview, Then it thanks María, explains the next steps in the process, and invites her to provide feedback
- Given the screening is complete, When the session ends, Then the session state is set to `completed`

---

# Epic 2: Evaluation Engine and Rubrics

**Description**: Enable operators to configure evaluation rubrics and automatically generate scored executive summaries with cited evidence from screening transcripts.

**PRD References**: M4, M5 | FR-04, FR-05

---

## US-2.1: Rubric Creation and Configuration

**As** Carlos (Recruiter),
**I want** to create evaluation rubrics with competencies, weights, and scoring criteria,
**So that** every candidate is evaluated against the same structured framework.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Carlos accesses the rubric editor, When he creates a new rubric, Then he can define competencies, assign weights, and set criteria per level (1-5)
- Given the system provides templates, When Carlos starts a new rubric, Then he can choose from BPO and Tech/SaaS default templates
- Given a rubric is created, When Carlos assigns it to a campaign, Then all screenings in that campaign use that rubric for evaluation

---

## US-2.2: Real-Time Evaluation During Screening

**As** Carlos (Recruiter),
**I want** the AI to evaluate candidate responses in real-time against the rubric,
**So that** scores are generated automatically as the interview progresses.

**Priority**: Must Have

**Acceptance Criteria**:
- Given a candidate is being screened, When the candidate responds to a competency question, Then the system generates a partial score for that competency
- Given the rubric defines criteria per level, When the AI evaluates a response, Then the score reflects the closest matching level (1-5)
- Given each score is assigned, When the evaluation is stored, Then it includes at least one verbatim quote from the candidate's response as evidence

---

## US-2.3: Executive Summary Generation

**As** Carlos (Recruiter),
**I want** to receive an executive summary for each completed screening with scores, evidence, and a recommendation,
**So that** I can make informed decisions quickly without reading full transcripts.

**Priority**: Must Have

**Acceptance Criteria**:
- Given a screening is completed, When the system generates the summary, Then it includes: global score, per-competency scores, verbatim quotes as evidence, key signals, and a recommendation (Highly Recommended / Recommended / Not Recommended)
- Given every score in the summary, When Carlos reviews it, Then each score is linked to at least one direct quote from the transcript
- Given the summary is generated, When it is stored, Then it appears automatically in the recruiter's review queue
- Given the summary generation, When it completes, Then it takes no longer than 30 seconds after screening completion

---

# Epic 3: Recruiter Dashboard

**Description**: Provide recruiters and operators with a web dashboard to manage campaigns, review candidate screenings with HITL decisions, and monitor metrics.

**PRD References**: M6 | FR-06, FR-11

---

## US-3.1: Campaign Creation and Management

**As** Carlos (Recruiter),
**I want** to create and manage recruitment campaigns with associated rubrics and knowledge bases,
**So that** I can organize and track my screening efforts per role.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Carlos is authenticated, When he creates a campaign, Then he can set: name, role description, rubric, knowledge base, and status
- Given a campaign is created, When the system generates it, Then a unique Telegram bot link is produced for that campaign
- Given a campaign exists, When Carlos manages it, Then he can activate, deactivate, or archive it
- Given campaigns are multi-tenant, When Carlos views campaigns, Then he only sees campaigns belonging to his tenant

---

## US-3.2: HITL Review Queue

**As** Carlos (Recruiter),
**I want** to see a queue of completed screenings awaiting my review,
**So that** I can efficiently process candidates and make hiring decisions.

**Priority**: Must Have

**Acceptance Criteria**:
- Given screenings are completed, When Carlos opens the review queue, Then he sees all candidates with status "Pending Review"
- Given the queue is displayed, When Carlos applies filters, Then he can filter by campaign, recommendation level, score range, and date
- Given the queue supports sorting, When Carlos clicks a column header, Then he can sort by score, date, or recommendation
- Given dashboard data updates, When Carlos wants fresh data, Then he refreshes the page manually to see new entries

---

## US-3.3: Candidate Detail View and Decision

**As** Carlos (Recruiter),
**I want** to see a detailed view of each candidate's screening results with full evidence,
**So that** I can make a well-informed approval or rejection decision.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Carlos selects a candidate from the queue, When the detail view loads, Then it shows: executive summary, per-competency scores with cited evidence, and the full transcript
- Given the detail view is displayed, When Carlos makes a decision, Then he can mark the candidate as "Approved" or "Rejected"
- Given Carlos's decision differs from the AI recommendation, When he submits, Then the system captures the disagreement reason
- Given a decision is made, When Carlos submits it, Then the candidate's status updates and the entry leaves the pending queue

---

## US-3.4: Campaign Metrics Dashboard

**As** Laura (Director/VP TA),
**I want** to see aggregate metrics for each campaign,
**So that** I can measure the effectiveness of the screening process and report to leadership.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Laura selects a campaign, When the metrics view loads, Then it displays: candidates started, completed, completion rate, approval rate, average score, NPS, escalation count, and abandonment rate
- Given the metrics include abandonment data, When Laura reviews it, Then she can see at which question abandonments most frequently occur

---

## US-3.5: Authentication and Access Control

**As** Carlos (Recruiter),
**I want** to log in securely to the dashboard,
**So that** only authorized team members can access candidate data.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Carlos navigates to the dashboard, When he is not authenticated, Then he is redirected to AWS Cognito login
- Given Carlos authenticates via Cognito, When authentication succeeds, Then he receives a JWT token and is redirected to the dashboard
- Given Carlos is authenticated, When he accesses any endpoint, Then his tenant is determined from the token and data is scoped accordingly
- Given Carlos's session expires, When he makes a request, Then he is redirected to re-authenticate

---

# Epic 4: Compliance and Audit

**Description**: Ensure transparency, consent tracking, immutable audit trails, and data retention policies to support regulatory compliance.

**PRD References**: M7 | NFR-03

---

## US-4.1: Immutable Audit Trail

**As** Laura (Director/VP TA),
**I want** every interaction, evaluation, and decision to be recorded in an immutable audit log,
**So that** we can demonstrate compliance and traceability in any audit.

**Priority**: Must Have

**Acceptance Criteria**:
- Given any candidate interaction occurs (message, evaluation, decision, escalation), When the event is processed, Then it is appended to the immutable audit log with a timestamp
- Given the audit log exists, When any system component attempts to modify or delete entries, Then the operation is rejected
- Given Laura needs to review an audit trail, When she accesses a candidate's record, Then the full chronological log of interactions is available

---

## US-4.2: Data Retention and Purge

**As** Laura (Director/VP TA),
**I want** candidate data to be automatically purged after a configurable retention period,
**So that** we comply with data minimization requirements.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the default retention period is 90 days, When a candidate's data reaches the retention limit, Then it is automatically purged from the system
- Given the purge process runs, When data is deleted, Then the deletion event is logged in the audit trail
- Given tenant data is segregated, When purge runs for one tenant, Then no other tenant's data is affected

---

## US-4.3: Escalation Logging

**As** Carlos (Recruiter),
**I want** all escalated questions from candidates to be logged and accessible,
**So that** I can improve the knowledge base and identify recurring information gaps.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the bot escalates a candidate question, When the escalation occurs, Then the question, context, and timestamp are logged
- Given escalation logs exist, When Carlos reviews them, Then he can see patterns of frequently asked questions not covered by the knowledge base

---

# Epic 5: Candidate Management and Feedback

**Description**: Manage candidate lifecycle, detect duplicate applications, and collect post-screening satisfaction feedback.

**PRD References**: M10 | FR-10, FR-12

---

## US-5.1: Candidate Profile and Lifecycle

**As** Carlos (Recruiter),
**I want** each candidate to have a profile that tracks their progression through the screening stages,
**So that** I can see where each candidate stands at a glance.

**Priority**: Must Have

**Acceptance Criteria**:
- Given a candidate starts a screening, When the system creates their profile, Then it stores basic professional data (no sensitive/protected information)
- Given a candidate progresses, When their state changes, Then it follows the lifecycle: `Initiated` → `In Screening` → `Completed` → `Pending Review` → `Approved` / `Rejected`
- Given candidate data is multi-tenant, When any query runs, Then results are scoped by tenant_id

---

## US-5.2: Duplicate Application Detection

**As** Carlos (Recruiter),
**I want** the system to detect when the same candidate applies multiple times to the same client,
**So that** I can avoid duplicate evaluations and potential gaming.

**Priority**: Must Have

**Acceptance Criteria**:
- Given a candidate starts a new screening, When the system checks for duplicates, Then it identifies if the same candidate has previously applied to the same tenant
- Given a duplicate is detected, When Carlos reviews the candidate, Then a flag indicates the previous application with link to prior results

---

## US-5.3: Post-Screening Satisfaction Survey (NPS)

**As** María (Candidate),
**I want** to provide feedback about my screening experience,
**So that** my opinion is valued and the process can improve.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María's screening is complete, When the bot presents the closing message, Then it asks for a satisfaction rating (1-5)
- Given María provides a rating, When she submits it, Then she is offered an optional open text field for additional feedback
- Given NPS data is collected, When Laura views campaign metrics, Then aggregated NPS scores are displayed

---

# Epic 6: Knowledge Base and RAG

**Description**: Enable operators to upload documents per campaign and provide the AI agent with a retrieval-augmented generation system to answer factual questions accurately.

**PRD References**: M8 | FR-08

---

## US-6.1: Document Upload and Processing

**As** Carlos (Recruiter),
**I want** to upload documents (PDF, DOCX, TXT) to a campaign's knowledge base,
**So that** the AI agent can answer candidate questions accurately using verified information.

**Priority**: Must Have

**Acceptance Criteria**:
- Given Carlos is managing a campaign, When he uploads a document, Then the system accepts PDF, DOCX, and TXT formats
- Given a document is uploaded, When it is processed, Then it is chunked and embedded into the vector database
- Given the knowledge base is per-campaign, When documents are processed, Then they are isolated and not accessible from other campaigns

---

## US-6.2: RAG-Powered Factual Responses

**As** María (Candidate),
**I want** the AI to answer my questions about the role using verified company information,
**So that** I get accurate and relevant answers.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María asks a factual question about the role or company, When the bot processes the question, Then it queries the campaign's knowledge base via vector similarity search before responding
- Given relevant information is found in the knowledge base, When the bot responds, Then it uses the retrieved information accurately
- Given no relevant information is found, When the bot cannot answer, Then it defaults to the anti-hallucination response ("No tengo esa información...")
- Given knowledge bases are isolated per campaign, When the bot queries, Then it only searches the current campaign's knowledge base

---

# Epic 7: Non-Functional — Platform Foundation

**Description**: Cross-cutting concerns that support all modules: multi-tenancy, performance, observability, and deployment.

**PRD References**: NFR-01, NFR-02, NFR-04, NFR-05, NFR-06, NFR-07

---

## US-7.1: Multi-Tenant Data Isolation

**As** Laura (Director/VP TA),
**I want** my company's candidate data to be completely isolated from other clients,
**So that** there is no risk of data leakage between tenants.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the system uses shared MongoDB with row-level isolation, When any document is created, Then it includes a `tenantId` field
- Given a user is authenticated, When any query executes, Then it is automatically scoped by the user's tenant
- Given tenant isolation is enforced, When a user attempts to access another tenant's data, Then the request is denied

---

## US-7.2: Bot Response Performance

**As** María (Candidate),
**I want** the bot to respond within a reasonable time,
**So that** the conversation feels natural and I don't lose interest.

**Priority**: Must Have

**Acceptance Criteria**:
- Given María sends a message during screening, When the bot processes and responds, Then the response is delivered in less than 10 seconds (including LLM processing)
- Given the system generates an executive summary, When a screening completes, Then the summary is ready within 30 seconds

---

## US-7.3: Bot Availability

**As** María (Candidate),
**I want** the Telegram bot to be available 24/7,
**So that** I can complete the screening at any time that works for me.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the bot is deployed, When María initiates a conversation at any hour, Then the bot responds (target: 99.5% uptime)
- Given a server restart occurs, When the bot recovers, Then all session state is preserved from the database

---

## US-7.4: Observability and Monitoring

**As** Laura (Director/VP TA),
**I want** the platform to have monitoring and logging,
**So that** operational issues are detected and resolved quickly.

**Priority**: Must Have

**Acceptance Criteria**:
- Given the system is running, When events occur, Then structured logs are sent to Loki with timestamp, correlation ID, log level, and message
- Given metrics are exposed, When Prometheus scrapes them, Then key application metrics are available
- Given Grafana dashboards exist, When operators access them, Then they can monitor system health and key operational metrics

---

# Requirements Traceability Matrix

| PRD Ref | Epic | Stories | Priority |
|---|---|---|---|
| M1 (FR-01) | Epic 1 | US-1.1, US-1.2 | Must Have |
| M2 (FR-02) | Epic 1 | US-1.3 | Must Have |
| M3 (FR-03) | Epic 1 | US-1.4, US-1.8 | Must Have |
| M4 (FR-04) | Epic 2 | US-2.1 | Must Have |
| M5 (FR-05) | Epic 2 | US-2.2, US-2.3 | Must Have |
| M6 (FR-06) | Epic 3 | US-3.2, US-3.3 | Must Have |
| M7 (FR-07) | Epic 1 | US-1.5, US-1.6 | Must Have |
| M8 (FR-08) | Epic 6 | US-6.1, US-6.2 | Must Have |
| M9 (FR-09) | Epic 1 | US-1.7 | Must Have |
| M10 (FR-10) | Epic 5 | US-5.3 | Must Have |
| FR-11 | Epic 3 | US-3.1, US-3.4 | Must Have |
| FR-12 | Epic 5 | US-5.1, US-5.2 | Must Have |
| NFR-01 | Epic 7 | US-7.2 | Must Have |
| NFR-02 | Epic 7 | US-7.3 | Must Have |
| NFR-03 | Epic 4 | US-4.1, US-4.2 | Must Have |
| NFR-04 | Epic 7 | US-7.1 | Must Have |
| NFR-06 | Epic 7 | US-7.4 | Must Have |
| NFR-07 | Epic 7 | — (CI/CD, deployment — infrastructure concern) | Must Have |

## Story Summary

| Epic | Story Count | Priority |
|---|---|---|
| Epic 1: Conversational Screening | 8 | Must Have |
| Epic 2: Evaluation Engine | 3 | Must Have |
| Epic 3: Recruiter Dashboard | 5 | Must Have |
| Epic 4: Compliance and Audit | 3 | Must Have |
| Epic 5: Candidate Management | 3 | Must Have |
| Epic 6: Knowledge Base / RAG | 2 | Must Have |
| Epic 7: Platform Foundation | 4 | Must Have |
| **Total** | **28** | |
