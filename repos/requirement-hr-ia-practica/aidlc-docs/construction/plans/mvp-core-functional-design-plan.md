# Functional Design Plan — Unit 1: MVP Core

## Unit Context
- **Unit**: `mvp-core`
- **Scope**: End-to-end screening flow — foundation + conversation + evaluation + campaign + candidate + compliance + dashboard UI
- **Stories**: 18 stories across 7 epics
- **Architecture**: DDD layered monolith (Next.js 16, DynamoDB, OpenAI GPT-4o, grammY, Cognito, Tailwind + shadcn/ui)

---

## Execution Checklist

- [x] Step 1: Analyze unit context (DONE — loaded all artifacts)
- [x] Step 2: Create functional design plan (this file)
- [x] Step 3: Generate clarifying questions (below)
- [x] Step 4: Store plan
- [x] Step 5: Collect and analyze answers
- [x] Step 6: Generate functional design artifacts
  - [x] domain-entities.md
  - [x] business-rules.md
  - [x] business-logic-model.md
  - [x] frontend-components.md
- [ ] Step 7: Present completion message
- [ ] Step 8: Wait for explicit approval
- [ ] Step 9: Record approval and update progress

---

## Clarifying Questions

### Section A: Conversation State Machine

**A1. Conversation phases and transitions**

The conversation flows through: `onboarding → consent → verification → screening → closing`. When a candidate fails basic requirements verification, the session ends. What should happen to the conversation state?

A) Set to `abandoned` (same as inactivity timeout)
B) Set to a distinct `disqualified` state
C) Set to `completed` (the process completed normally, just with a negative outcome)
D) Set to `completed` with a `disqualificationReason` field

[Answer]: A

---

**A2. Follow-up question (repregunta) logic**

The rubric guides when to ask follow-ups. What determines whether a follow-up is needed for a given response?

A) Always ask one follow-up per competency question (fixed)
B) Ask a follow-up only if the score for that response is below a threshold (e.g., score < 3)
C) The LLM decides dynamically based on response depth and rubric criteria
D) Ask a follow-up if the response lacks a concrete example (STAR method check)

[Answer]: A

---

**A3. Competency question ordering**

When the screening phase begins, in what order are competency questions asked?

A) Fixed order defined in the rubric (rubric defines question sequence)
B) Random order from the rubric competencies
C) LLM selects the next competency dynamically based on conversation flow
D) Ordered by competency weight (highest weight first)

[Answer]: B

---

**A4. Session state persistence granularity**

The `SessionState` object is stored in DynamoDB. When is it persisted?

A) After every single message exchange (most durable, higher write cost)
B) After each phase transition (onboarding → consent → verification → screening → closing)
C) After each competency question is answered (including follow-ups)
D) Both A and C — after every message AND after phase transitions

[Answer]: A

---

**A5. Escalation Level 1 vs Level 2 trigger**

The requirements define: Level 1 = info not available (log and continue), Level 2 = repeated requests (suggest human follow-up). What defines "repeated"?

A) Same question asked 2+ times in the same session
B) Same topic asked 3+ times in the same session
C) Any question the bot cannot answer from the knowledge base (Level 1 always, Level 2 never in MVP)
D) The LLM classifies the urgency and decides the level

[Answer]: A

---

### Section B: Evaluation Engine

**B1. Partial scoring timing**

`EvaluateResponseUseCase` scores responses in real-time. Does evaluation happen:

A) Immediately after each candidate message (synchronous — blocks the response)
B) Asynchronously after sending the bot's next message (non-blocking)
C) Only at the end of each competency (after main question + follow-up)
D) Only once at the end of the entire screening (batch evaluation)

[Answer]: D

---

**B2. Evidence extraction mechanism**

Each competency score must include at least one verbatim quote. How is the quote extracted?

A) The LLM is prompted to extract the quote as part of the scoring call (single LLM call returns score + quote)
B) A separate LLM call extracts quotes after scoring
C) The system uses the full candidate response as the quote (no extraction needed)
D) The recruiter manually selects quotes during HITL review

[Answer]: A

---

**B3. Global score calculation**

Given competency weights in the rubric, how is the global score calculated?

A) Weighted average: sum(score_i × weight_i) / sum(weight_i), result normalized to 1-5
B) Simple average of all competency scores (weights ignored for global score)
C) Weighted average, result expressed as a percentage (0-100%)
D) Weighted average normalized to 1-5, rounded to 1 decimal place

[Answer]: A

---

**B4. Recommendation thresholds**

The recommendation levels are: Highly Recommended / Recommended / Not Recommended. What score thresholds define each level?

A) Highly Recommended: ≥ 4.0 | Recommended: ≥ 2.5 | Not Recommended: < 2.5
B) Highly Recommended: ≥ 4.5 | Recommended: ≥ 3.0 | Not Recommended: < 3.0
C) Highly Recommended: ≥ 4.0 | Recommended: ≥ 3.0 | Not Recommended: < 3.0
D) Thresholds are configurable per rubric (stored in rubric definition)

[Answer]: A

---

**B5. Hardcoded rubric templates content**

The MVP uses hardcoded BPO and Tech/SaaS templates. What competencies should each template include?

A) BPO: [Comunicación, Orientación al cliente, Manejo de objeciones, Trabajo bajo presión, Adaptabilidad] | Tech/SaaS: [Resolución de problemas, Comunicación técnica, Trabajo en equipo, Aprendizaje continuo, Orientación a resultados]
B) Define them now with a different set — provide your preferred competencies
C) Use a single generic template for MVP (same for BPO and Tech/SaaS)
D) Leave template content as a configuration concern (not hardcoded in domain)

[Answer]: A

---

### Section C: Campaign and Knowledge Base

**C1. Knowledge base in-context loading (MVP)**

Since RAG is post-MVP, the MVP uses in-context loading. How is the knowledge base content included in the prompt?

A) The entire knowledge base document is injected into the system prompt at session start
B) The knowledge base content is injected only when the candidate asks a factual question (lazy loading)
C) A summary of the knowledge base is pre-generated and injected into every prompt
D) No knowledge base in MVP — the bot only uses the rubric and campaign description

[Answer]: A

---

**C2. Campaign Telegram link format**

The system generates a unique Telegram link per campaign. What format should it use?

A) `https://t.me/{botUsername}?start={campaignId}` (standard Telegram deep link)
B) `https://t.me/{botUsername}?start={tenantId}_{campaignId}` (includes tenant for routing)
C) `https://t.me/{botUsername}?start={shortCode}` (short opaque code, campaignId stored in DB)
D) The bot username is per-tenant (each tenant has their own bot)

[Answer]: A

---

### Section D: Candidate Lifecycle

**D1. Candidate identity across sessions**

A candidate is identified by their Telegram user ID. If the same Telegram user starts a new screening for the same campaign (e.g., after abandonment), what happens?

A) Create a new candidate record — the previous one remains as historical data
B) Resume the existing candidate record and conversation
C) Block the new attempt — one attempt per campaign per Telegram user
D) Create a new attempt but link it to the existing candidate profile (attempt history)

[Answer]: A

---

**D2. Candidate name collection**

The candidate profile has an optional `name` field. When is the name collected?

A) During onboarding — the bot asks for the candidate's name as the first question
B) During basic requirements verification — name is one of the required fields
C) Not collected by the bot — the recruiter fills it in manually if needed
D) Extracted from the Telegram profile (first name + last name from Telegram API)

[Answer]: A

---

### Section E: Dashboard UI

**E1. Review queue default view**

When Carlos opens the HITL review queue, what is the default state?

A) All pending candidates, sorted by date (oldest first — FIFO)
B) All pending candidates, sorted by score (highest first)
C) All pending candidates, sorted by date (newest first)
D) Filtered to the most recently active campaign, sorted by date (newest first)

[Answer]: A

---

**E2. Candidate detail view — transcript display**

The candidate detail view shows the full transcript. How should it be displayed?

A) Full scrollable transcript below the executive summary (all messages visible)
B) Collapsible transcript section (collapsed by default, expandable)
C) Separate tab: [Summary] [Transcript] [Scores]
D) Transcript shown inline with scores — each score is linked to its evidence quote in the transcript

[Answer]: A

---

**E3. Campaign creation form — required fields**

When Carlos creates a campaign, which fields are mandatory vs optional?

A) Mandatory: name, role description, rubric | Optional: knowledge base documents, basic requirements config
B) Mandatory: name, rubric | Optional: role description, knowledge base, basic requirements
C) Mandatory: name, role description, rubric, at least one basic requirement | Optional: knowledge base
D) All fields mandatory (name, role description, rubric, basic requirements, knowledge base)

[Answer]: A

---

**E4. Disagreement capture UX**

When Carlos's decision differs from the AI recommendation, the system captures a disagreement reason. How is this presented?

A) A required text field appears automatically when the decision differs from AI recommendation
B) An optional text field always visible regardless of decision
C) A modal dialog appears asking for the reason when a disagreement is detected
D) A dropdown with predefined reasons + optional free text

[Answer]: A

---

### Section F: Data Model

**F1. DynamoDB table design for conversations**

Conversations contain an array of messages. Given DynamoDB's item size limit (400KB), how should messages be stored?

A) Embedded in the Conversation item (messages array) — acceptable for MVP given 15-25 min sessions
B) Separate Messages table with conversationId as partition key
C) Hybrid: last N messages embedded, older messages in a separate table
D) Embedded but with a hard cap (e.g., max 100 messages per conversation)

[Answer]: A

---

**F2. Tenant isolation key pattern**

All DynamoDB tables use tenant isolation. What is the partition key pattern?

A) `tenantId#entityId` as a composite partition key (e.g., `tenant123#conv456`)
B) `tenantId` as partition key, `entityId` as sort key
C) `entityId` as partition key, `tenantId` as a filter attribute (not in key)
D) Separate DynamoDB tables per tenant (physical isolation)

[Answer]: A

---
