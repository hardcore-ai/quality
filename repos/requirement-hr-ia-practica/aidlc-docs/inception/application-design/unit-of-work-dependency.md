# Unit of Work Dependencies — EntreVista AI

## Dependency Matrix

| Unit | Depends On | Depended By |
|---|---|---|
| **Unit 1**: MVP Core | — (foundation) | Units 2, 3, 4 |
| **Unit 2**: Compliance & Audit | Unit 1 | Units 3, 4 |
| **Unit 3**: Knowledge Base & RAG | Unit 1, Unit 2 | Unit 4 |
| **Unit 4**: Advanced Features | Unit 1, Unit 2 | — |

## Dependency Diagram

```
Unit 1: MVP Core
  │
  ├──► Unit 2: Compliance & Audit
  │      │
  │      ├──► Unit 3: Knowledge Base & RAG
  │      │
  │      └──► Unit 4: Advanced Features
  │
  └──► Unit 4: Advanced Features
```

## Implementation Order

```
Phase 1 (MVP):     Unit 1 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━►
Phase 2:                    Unit 2 ━━━━━━━━━━━━━━━━━━━━━►
Phase 3:                              Unit 3 ━━━━━━━━━━━►
Phase 4:                              Unit 4 ━━━━━━━━━━━►
                                      (3 and 4 can be parallel)
```

**Note**: Units 3 and 4 are independent of each other and can be implemented in parallel after Unit 2.

---

## Integration Points Between Units

### Unit 1 → Unit 2
| Integration Point | What Unit 2 Extends |
|---|---|
| Audit event logging | Upgrade from basic append to immutable with IAM protection |
| Compliance service | Add data retention, auto-purge, exportable reports |
| Escalation handling | Upgrade from log-only to dashboard alert (Level 3) |
| Dashboard | Add audit trail viewer, escalation log viewer |

### Unit 1 → Unit 3
| Integration Point | What Unit 3 Extends |
|---|---|
| Knowledge base infrastructure | Replace in-context loading with vector DB RAG |
| Campaign management | Add document upload UI and API |
| Conversation processing | Add semantic retrieval step before OpenAI call |
| OpenAI client | Add embedding generation alongside chat completions |

### Unit 1 → Unit 4
| Integration Point | What Unit 4 Extends |
|---|---|
| Conversation module | Add re-engagement scheduler (24h/48h/72h) |
| Campaign module | Add metrics aggregation and dashboard views |
| Evaluation module | Add configurable rubric editor (replace hardcoded templates) |
| Candidate module | Add duplicate detection, NPS collection |
| Infrastructure | Add Prometheus, Grafana, Loki; migrate auth to Cognito |

---

## Construction Phase Plan

Each unit goes through the full construction cycle:

| Stage | Unit 1: MVP Core | Unit 2: Compliance | Unit 3: KB & RAG | Unit 4: Advanced |
|---|---|---|---|---|
| Functional Design | Full (all MVP logic) | Retention + purge + escalation L3 | Document processing + RAG pipeline | Re-engagement + metrics + rubric editor |
| NFR Requirements | DynamoDB perf, bot latency, multi-tenancy | Immutability, IAM | Vector DB perf, embedding costs | Scheduler, observability |
| NFR Design | Tenant middleware, session state, logging | Append-only DynamoDB, IAM policies | Embedding pipeline, similarity search | Cron jobs, Prometheus metrics |
| Infrastructure Design | AWS ECS + ALB + VPC + DynamoDB + Cognito + ECR (Terraform) | IAM policies, scheduled purge | Vector DB hosting (Pinecone) | Grafana/Loki stack |
| Code Generation | Full MVP implementation | Compliance extensions | RAG implementation | Feature extensions |
| Build and Test | E2E: Telegram → review | Audit integrity tests | RAG accuracy tests | Feature tests |
