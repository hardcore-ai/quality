# AI-DLC State Tracking

## Project Information
- **Project Name**: EntreVista AI — Plataforma de Entrevistas Agénticas para América Latina
- **Project Type**: Greenfield
- **Start Date**: 2026-03-15T00:00:00Z
- **Current Stage**: INCEPTION - Units Generation.

## Workspace State
- **Existing Code**: No
- **Reverse Engineering Needed**: No
- **Workspace Root**: /home/andrescaicedom/projects/30x/requirement-hr-ia-practica/
- **Input Artifacts**: requirement.md (PRD completo)

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Extension Configuration
| Extension | Enabled | Decided At |
|---|---|---|
| Security Baseline | No | Requirements Analysis |

## Execution Plan Summary
- **Total Stages**: 12
- **Stages to Execute**: Application Design, Units Generation, Functional Design (per-unit), NFR Requirements (per-unit), NFR Design (per-unit), Infrastructure Design (per-unit), Code Generation (per-unit), Build and Test
- **Stages Skipped**: Reverse Engineering (greenfield)

## Stage Progress

### INCEPTION PHASE
- [x] Workspace Detection (COMPLETED)
- [x] Reverse Engineering (SKIPPED — greenfield)
- [x] Requirements Analysis (COMPLETED)
- [x] User Stories (COMPLETED)
- [x] Workflow Planning (COMPLETED)
- [x] Application Design (COMPLETED)
- [x] Units Generation (COMPLETED)

### CONSTRUCTION PHASE (per-unit)
- [x] Functional Design - COMPLETED (Unit 1: MVP Core)
- [x] NFR Requirements - COMPLETED (Unit 1: MVP Core)
- [x] NFR Design - COMPLETED (Unit 1: MVP Core)
- [x] Infrastructure Design - COMPLETED (Unit 1: MVP Core)
- [x] Code Generation - COMPLETED (Unit 1: MVP Core)
- [ ] NFR Design (Unit 1: MVP Core) — N/A (already completed above)
- [ ] Infrastructure Design (Unit 1: MVP Core) — N/A (already completed above)
- [ ] Code Generation (Unit 1: MVP Core) — N/A (already completed above)
- [ ] Build and Test

### OPERATIONS PHASE
- [ ] Operations - PLACEHOLDER

## Current Status
- **Lifecycle Phase**: CONSTRUCTION
- **Current Stage**: Build and Test
- **Next Stage**: Operations (placeholder)
- **Status**: Code Generation COMPLETED — ready for Build and Test

## MVP Scope Decision
- **MVP Core**: Screening conversacional agéntico con evidencia para el reclutador
- **Strategy**: Design everything completely, implement prioritized by MVP
- **MVP Simplifications**: In-context loading (no vector DB), CloudWatch logging (no Prometheus/Grafana), hardcoded rubric templates, no re-engagement/NPS/metrics dashboard
- **MVP Infrastructure**: AWS ECS (Fargate) + ALB + DynamoDB + Cognito + VPC + ECR, all via Terraform
- **UI Stack**: Tailwind CSS + shadcn/ui
- **Unit Prioritization**: (1) Technical foundation + deploy, (2) Conversational engine + evaluation, (3) Dashboard + HITL review
