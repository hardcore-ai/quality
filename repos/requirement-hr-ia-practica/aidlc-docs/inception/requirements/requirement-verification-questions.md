# Requirements Verification Questions — EntreVista AI

Please answer the following questions to clarify areas that are TBD or ambiguous in the PRD (`requirement.md`). Fill in the letter choice after each `[Answer]:` tag. If none of the options match, choose the last option (Other) and describe your preference.

---

## Question 1
What backend programming language and framework should be used for the core API and business logic?

A) Python + FastAPI
B) Node.js + NestJS (TypeScript)
C) Go + Gin/Echo
D) Java + Spring Boot
E) Other (please describe after [Answer]: tag below)

[Answer]: E. Como plataforma fullstack vamos a usar NextJS 16, con Tailwind y React. Las APIs van a estar en NextJS también. 

---

## Question 2
What LLM provider and model should power the agentic conversational engine?

A) Anthropic Claude (claude-sonnet-4-20250514 or claude-opus-4-20250514)
B) OpenAI GPT-4o / GPT-4.1
C) Google Gemini 2.5
D) Multi-provider with abstraction layer (specify preferred primary)
E) Other (please describe after [Answer]: tag below)

[Answer]:B (tanto para modelo de chat como para embeddings)

---

## Question 3
What database technology should be used for primary data storage (campaigns, candidates, evaluations, transcripts)?

A) PostgreSQL (relational)
B) MongoDB (document-oriented)
C) DynamoDB (AWS managed NoSQL)
D) PostgreSQL + Redis (relational + cache)
E) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## Question 4
What cloud infrastructure provider should be used for deployment?

A) AWS (Amazon Web Services)
B) Google Cloud Platform (GCP)
C) Microsoft Azure
D) Self-hosted / On-premises
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 5
What frontend technology should be used for the Recruiter Dashboard?

A) React + TypeScript (Next.js)
B) Vue.js + TypeScript (Nuxt)
C) Angular + TypeScript
D) React + TypeScript (Vite + SPA)
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 6
What authentication/authorization mechanism should be used for the recruiter dashboard?

A) Email/password + JWT tokens (self-managed)
B) OAuth 2.0 / OpenID Connect with external provider (Auth0, Cognito, Clerk)
C) Magic link / passwordless authentication
D) SSO integration (SAML/OIDC) for enterprise clients
E) Other (please describe after [Answer]: tag below)

[Answer]:B. Cognito.

---

## Question 7
What is the expected multi-tenancy isolation model for client data?

A) Shared database with row-level tenant isolation (tenant_id column)
B) Shared database with schema-per-tenant isolation
C) Database-per-tenant (full isolation)
D) Shared database (row-level) for MVP, with migration path to schema-per-tenant
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 8
What is the target deployment architecture for the MVP?

A) Monolithic application (single deployable unit)
B) Modular monolith (single deployment, internal module boundaries)
C) Microservices (separate deployable services per module)
D) Serverless (Lambda/Cloud Functions per module)
E) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## Question 9
How should the Telegram bot integration be implemented?

A) Direct Telegram Bot API integration (webhooks)
B) Using a Telegram bot framework (e.g., python-telegram-bot, telegraf.js, grammY)
C) Through a messaging abstraction layer (to support future channels like WhatsApp/Web)
D) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## Question 10
What is the expected candidate volume for MVP pilot (first 90 days)?

A) Low: 100-500 candidates total
B) Medium: 500-2,000 candidates total
C) High: 2,000-10,000 candidates total
D) Very High: 10,000+ candidates total
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 11
What observability and monitoring stack should be used?

A) Cloud-native (CloudWatch / Cloud Monitoring / Azure Monitor)
B) Open-source stack (Prometheus + Grafana + Loki)
C) SaaS observability (Datadog, New Relic, or similar)
D) Minimal logging for MVP (structured logs to cloud provider)
E) Other (please describe after [Answer]: tag below)

[Answer]:B

---

## Question 12
How should the RAG (Retrieval Augmented Generation) system for the knowledge base work?

A) Vector database (Pinecone, Weaviate, or pgvector) with embedding-based retrieval
B) Simple document chunking + keyword search (no vector DB for MVP)
C) Managed RAG service (AWS Bedrock Knowledge Bases, Vertex AI Search)
D) In-context loading (full documents in LLM context window, no retrieval needed for MVP)
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 13
What CI/CD and DevOps tooling should be used?

A) GitHub Actions
B) GitLab CI/CD
C) AWS CodePipeline / CodeBuild
D) Manual deployment for MVP (scripts + documentation)
E) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 14
Should the MVP support real-time WebSocket communication for the recruiter dashboard, or is polling/SSE sufficient?

A) Real-time WebSocket (live updates as candidates complete screenings)
B) Server-Sent Events (SSE) for one-way live updates
C) Polling (periodic refresh, simpler for MVP)
D) Manual refresh (simplest — recruiter refreshes page)
E) Other (please describe after [Answer]: tag below)

[Answer]:D

---

## Question 15
What is the preferred language for all code, documentation, and technical artifacts?

A) English (code + docs in English, product UI in Spanish)
B) Spanish (everything in Spanish including code comments and docs)
C) Mixed (code in English, user-facing content and docs in Spanish)
D) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## Question 16: Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)
B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)
C) Other (please describe after [Answer]: tag below)

[Answer]:B

---
