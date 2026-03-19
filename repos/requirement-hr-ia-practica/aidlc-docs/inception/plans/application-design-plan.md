# Application Design Plan — EntreVista AI

## Overview
Define the high-level component architecture for EntreVista AI as a Next.js 16 modular monolith. This covers component identification, interfaces, service layer, and dependencies.

---

## Clarification Questions

### Question 1
How should the Next.js 16 modular monolith be organized internally?

A) **Feature folders under `src/`** — Each module is a folder under `src/modules/` with its own components, API routes, services, and types. Shared code in `src/shared/`. Dashboard pages in `src/app/`.
B) **Next.js App Router groups** — Use route groups `(dashboard)`, `(api)` to organize by feature. Modules live within the app directory structure itself.
C) **Domain-Driven layers** — Organize by layer first (`src/domain/`, `src/application/`, `src/infrastructure/`), then by module within each layer.
D) Other (please describe after [Answer]: tag below)

[Answer]:C

### Question 2
How should the Telegram bot process run in relation to the Next.js application?

A) **Webhook via API route** — grammY receives Telegram updates via a Next.js API route (`/api/telegram/webhook`). Stateless — each message is an independent request. Bot runs within the Next.js process.
B) **Separate long-polling process** — grammY runs as a separate Node.js process using long-polling. Communicates with Next.js via internal API or shared database. Requires separate deployment.
C) **Webhook + background worker** — Webhook receives messages via API route, but heavy processing (LLM calls, evaluation) is offloaded to a background job queue (e.g., BullMQ + Redis).
D) Other (please describe after [Answer]: tag below)

[Answer]:A

### Question 3
How should the OpenAI conversation context be managed for multi-turn screening sessions?

A) **Full history in DB** — Store all messages in MongoDB. On each new message, load full conversation history and send to OpenAI as message array. Simple but increases token usage over time.
B) **Sliding window + summary** — Keep last N messages in context. Periodically summarize older messages into a condensed context. Balances token usage with context quality.
C) **Session state object** — Maintain a structured state object (current question, competencies covered, partial scores) alongside raw messages. Send state + recent messages to OpenAI. Most structured.
D) Other (please describe after [Answer]: tag below)

[Answer]:C. El objeto puede vivir en base de datos DynamoDB (no vamos a usar MongoDB)

---

## Execution Plan

- [x] **Step 1**: Define module components and responsibilities (`components.md`)
- [x] **Step 2**: Define component method signatures and interfaces (`component-methods.md`)
- [x] **Step 3**: Define service layer and orchestration patterns (`services.md`)
- [x] **Step 4**: Map component dependencies and communication patterns (`component-dependency.md`)
- [x] **Step 5**: Consolidate into unified application design document (`application-design.md`)
