# Story Generation Plan — EntreVista AI

## Overview
This plan defines the methodology for converting the EntreVista AI PRD and requirements into user stories following INVEST criteria.

---

## Clarification Questions

Please answer the following questions to guide story generation. Fill in the letter choice after each `[Answer]:` tag.

### Question 1
What story breakdown approach do you prefer for organizing the user stories?

A) **User Journey-Based** — Stories follow the end-to-end user workflows (candidate journey, recruiter journey, admin journey). Best for understanding flows and handoffs between personas.
B) **Feature-Based** — Stories organized around system features/modules (Telegram bot, evaluation engine, dashboard, compliance, candidate management). Best for development sprint planning.
C) **Epic-Based** — High-level epics decomposed into sub-stories. Best for roadmap planning and stakeholder communication.
D) **Hybrid: Epics by Module + Stories by Journey** — Epics map to PRD modules, but individual stories follow user journeys within each epic. Balances sprint planning with user-centric thinking.
E) Other (please describe after [Answer]: tag below)

[Answer]:C

### Question 2
What level of granularity do you want for user stories?

A) **Coarse** — One story per major feature (e.g., "As a candidate, I can complete a screening interview via Telegram"). ~15-25 stories total. Good for high-level planning.
B) **Medium** — Stories broken down by distinct user interactions within each feature (e.g., separate stories for consent, basic verification, screening questions, follow-ups). ~30-50 stories total. Good for sprint planning.
C) **Fine** — Detailed stories for each atomic interaction and edge case (e.g., separate stories for "candidate asks about salary", "candidate requests human contact", "candidate abandons at question 3"). ~60-100 stories total. Good for detailed implementation guidance.
D) Other (please describe after [Answer]: tag below)

[Answer]:B

### Question 3
What acceptance criteria format do you prefer?

A) **Given/When/Then** (BDD-style) — Structured, testable, good for automated testing. Example: "Given a candidate has given consent, When they answer a competency question, Then the agent generates a contextual follow-up question."
B) **Checklist** — Simple bullet list of conditions that must be true. Example: "- Consent is recorded with timestamp. - Candidate can opt out at any time."
C) **Scenario-Based** — Named scenarios with expected outcomes. Example: "Scenario: Candidate asks about salary → Agent responds with 'No tengo esa información' message."
D) Other (please describe after [Answer]: tag below)

[Answer]:A

### Question 4
How should the PRD's 30/60/90-day delivery plan map to story prioritization?

A) **Tag stories by delivery phase** — Each story tagged as Day 1-30, Day 31-60, or Day 61-90 based on the PRD plan
B) **MoSCoW only** — Use the PRD's Must/Should/Could/Won't classification without timeline mapping
C) **Both** — Tag stories with both MoSCoW priority AND delivery phase
D) Other (please describe after [Answer]: tag below)

[Answer]:B

### Question 5
The PRD defines 3 buyer personas (Director TA, Head of People, Recruiter Operativo) and 1 candidate persona. Should the stories include additional technical/system personas?

A) **PRD personas only** — Use only the 4 personas from the PRD (Director TA, Head of People, Recruiter, Candidate)
B) **Add System Admin** — Add a system admin persona for platform-level configuration and monitoring
C) **Add System Admin + System Actor** — Add both a system admin persona and a "System" actor for automated behaviors (re-engagement messages, data purge, summary generation)
D) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Execution Plan

After questions are answered, the following steps will be executed:

- [x] **Step 1**: Define personas in `personas.md` based on PRD buyer personas + approved additional personas
- [x] **Step 2**: Create epics/groupings based on approved breakdown approach
- [x] **Step 3**: Generate user stories for Module 1 — Conversational Engine (FR-01 to FR-03, FR-07, FR-09)
- [x] **Step 4**: Generate user stories for Module 2 — Evaluation Engine (FR-04, FR-05)
- [x] **Step 5**: Generate user stories for Module 3 — Recruiter Dashboard (FR-06, FR-11)
- [x] **Step 6**: Generate user stories for Module 4 — Compliance (NFR-03, consent, audit)
- [x] **Step 7**: Generate user stories for Module 5 — Candidate Management (FR-10, FR-12)
- [x] **Step 8**: Generate user stories for Knowledge Base / RAG (FR-08)
- [x] **Step 9**: Cross-reference stories against PRD requirements for completeness
- [x] **Step 10**: Apply prioritization tags per approved method
- [x] **Step 11**: Final review — verify INVEST criteria compliance for all stories
