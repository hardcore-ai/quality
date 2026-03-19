# Unit of Work Plan — EntreVista AI

## Overview
Decompose the EntreVista AI modular monolith into ordered units of work for the CONSTRUCTION phase. Since this is a single deployable Next.js application, units represent logical modules with clear boundaries — not independent services.

**Key constraint from user**: Design everything completely, but implement prioritized by MVP. MVP core = "screening conversacional agéntico con evidencia para el reclutador."

---

## Clarification Questions

### Question 1
How should the construction phase units be ordered for implementation?

A) **Foundation-first** — Start with shared infrastructure (DynamoDB, auth, project scaffold), then build modules bottom-up. Each unit builds on the previous. Units: (1) Foundation, (2) Conversation Engine, (3) Evaluation Engine, (4) Dashboard + HITL, (5) Compliance, (6) Campaign Management, (7) Knowledge Base.
B) **Vertical slice** — Build one complete end-to-end flow first (Telegram → screening → evaluation → dashboard review), then extend with remaining features. Units: (1) E2E Core Slice (all layers for happy path), (2) Dashboard Extensions, (3) Compliance & Audit, (4) Knowledge Base, (5) Edge Cases & Re-engagement.
C) **MVP-first as single unit** — Implement the entire MVP as one unit (foundation + conversation + evaluation + basic dashboard), then add post-MVP features as separate units. Units: (1) MVP Core, (2) Compliance & Audit, (3) Knowledge Base / RAG, (4) Advanced Features.
D) Other (please describe after [Answer]: tag below)

[Answer]:C

---

## Execution Plan

After question is answered, the following steps will be executed:

- [x] **Step 1**: Define units of work with responsibilities and scope (`unit-of-work.md`)
- [x] **Step 2**: Map dependencies between units (`unit-of-work-dependency.md`)
- [x] **Step 3**: Map user stories to units (`unit-of-work-story-map.md`)
- [x] **Step 4**: Validate completeness — all 28 stories assigned, all components covered
