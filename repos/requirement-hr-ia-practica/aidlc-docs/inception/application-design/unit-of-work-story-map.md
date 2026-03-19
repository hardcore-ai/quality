# Unit of Work — Story Map — EntreVista AI

## Story Assignment

### Unit 1: MVP Core (18 stories)

| Story | Title | Epic |
|---|---|---|
| **US-1.1** | Bot Onboarding and AI Disclosure | Epic 1: Conversational Screening |
| **US-1.2** | Consent Collection | Epic 1: Conversational Screening |
| **US-1.3** | Basic Requirements Verification | Epic 1: Conversational Screening |
| **US-1.4** | Competency-Based Screening Questions | Epic 1: Conversational Screening |
| **US-1.5** | Anti-Hallucination and Out-of-Scope Handling | Epic 1: Conversational Screening |
| **US-1.6** | Escalation to Human (Level 1-2 only) | Epic 1: Conversational Screening |
| **US-1.8** | Screening Completion and Closing | Epic 1: Conversational Screening |
| **US-2.1** | Rubric Creation and Configuration (hardcoded templates only) | Epic 2: Evaluation Engine |
| **US-2.2** | Real-Time Evaluation During Screening | Epic 2: Evaluation Engine |
| **US-2.3** | Executive Summary Generation | Epic 2: Evaluation Engine |
| **US-3.1** | Campaign Creation and Management | Epic 3: Recruiter Dashboard |
| **US-3.2** | HITL Review Queue | Epic 3: Recruiter Dashboard |
| **US-3.3** | Candidate Detail View and Decision | Epic 3: Recruiter Dashboard |
| **US-3.5** | Authentication and Access Control | Epic 3: Recruiter Dashboard |
| **US-4.1** | Immutable Audit Trail (basic — append-only logging) | Epic 4: Compliance |
| **US-5.1** | Candidate Profile and Lifecycle | Epic 5: Candidate Management |
| **US-7.1** | Multi-Tenant Data Isolation | Epic 7: Platform Foundation |
| **US-7.2** | Bot Response Performance | Epic 7: Platform Foundation |

### Unit 2: Compliance & Audit (4 stories)

| Story | Title | Epic |
|---|---|---|
| **US-4.1** | Immutable Audit Trail (full — IAM protection, viewer) | Epic 4: Compliance |
| **US-4.2** | Data Retention and Purge | Epic 4: Compliance |
| **US-4.3** | Escalation Logging | Epic 4: Compliance |
| **US-1.6** | Escalation to Human (Level 3 — dashboard alerts) | Epic 1: Conversational Screening |

### Unit 3: Knowledge Base & RAG (2 stories)

| Story | Title | Epic |
|---|---|---|
| **US-6.1** | Document Upload and Processing | Epic 6: Knowledge Base |
| **US-6.2** | RAG-Powered Factual Responses | Epic 6: Knowledge Base |

### Unit 4: Advanced Features (7 stories)

| Story | Title | Epic |
|---|---|---|
| **US-1.7** | Session Pause and Re-engagement (automated reminders) | Epic 1: Conversational Screening |
| **US-2.1** | Rubric Creation and Configuration (full editor UI) | Epic 2: Evaluation Engine |
| **US-3.4** | Campaign Metrics Dashboard | Epic 3: Recruiter Dashboard |
| **US-5.2** | Duplicate Application Detection | Epic 5: Candidate Management |
| **US-5.3** | Post-Screening Satisfaction Survey (NPS) | Epic 5: Candidate Management |
| **US-7.3** | Bot Availability (99.5% SLA, enhanced) | Epic 7: Platform Foundation |
| **US-7.4** | Observability and Monitoring (Prometheus + Grafana + Loki) | Epic 7: Platform Foundation |

---

## Coverage Validation

| Metric | Count |
|---|---|
| **Total stories** | 28 |
| **Stories in Unit 1** | 18 |
| **Stories in Unit 2** | 4 |
| **Stories in Unit 3** | 2 |
| **Stories in Unit 4** | 7 |
| **Assigned total** | 31 (3 stories span multiple units: US-1.6, US-2.1, US-4.1) |
| **Unassigned** | 0 |

### Stories Spanning Multiple Units

| Story | Unit 1 (MVP) | Later Unit | Reason |
|---|---|---|---|
| US-1.6 (Escalation) | Level 1-2: log + inform | Unit 2: Level 3 dashboard alerts | Progressive enhancement |
| US-2.1 (Rubrics) | Hardcoded templates | Unit 4: Full editor UI | MVP simplification |
| US-4.1 (Audit Trail) | Basic append-only logging | Unit 2: IAM-protected + viewer | Progressive enhancement |

---

## Epic Coverage by Unit

| Epic | Unit 1 | Unit 2 | Unit 3 | Unit 4 |
|---|---|---|---|---|
| Epic 1: Conversational Screening | 7/8 | 1 | — | 1 |
| Epic 2: Evaluation Engine | 3/3 | — | — | 1 (extends) |
| Epic 3: Recruiter Dashboard | 4/5 | — | — | 1 |
| Epic 4: Compliance & Audit | 1/3 (basic) | 3/3 (full) | — | — |
| Epic 5: Candidate Management | 1/3 | — | — | 2 |
| Epic 6: Knowledge Base & RAG | — | — | 2/2 | — |
| Epic 7: Platform Foundation | 2/4 | — | — | 2 |
