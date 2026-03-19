# Domain Entities — Unit 1: MVP Core

## Overview

All domain entities are pure TypeScript — no framework dependencies. They live in `src/domain/` and define the core business objects, their state, and their invariants.

---

## 1. Conversation Aggregate

**Location**: `src/domain/conversation/`

### 1.1 Conversation

The root aggregate for a screening session. Owns the full message history and session state.

```typescript
interface Conversation {
  id: string;                        // UUID
  tenantId: string;                  // Composite PK prefix: tenantId#id
  campaignId: string;
  candidateId: string;
  state: ConversationState;
  phase: ConversationPhase;
  messages: Message[];               // Embedded — acceptable for 15-25 min sessions
  sessionState: SessionState;
  escalationCount: number;           // Tracks repeated unanswerable questions
  createdAt: Date;
  updatedAt: Date;
}

type ConversationState =
  | 'active'
  | 'paused'
  | 'abandoned'    // Covers both: inactivity timeout AND failed requirements
  | 'completed';

type ConversationPhase =
  | 'onboarding'
  | 'consent'
  | 'verification'
  | 'screening'
  | 'closing';
```

**Design notes**:
- `state` tracks the lifecycle (active/paused/abandoned/completed)
- `phase` tracks the current step within an active conversation
- Failed requirements → `state: 'abandoned'` (same as inactivity timeout — A1)
- Messages embedded in item — acceptable for MVP session length (A1, F1)

### 1.2 Message

```typescript
interface Message {
  role: 'agent' | 'candidate';
  content: string;
  timestamp: Date;
  phase: ConversationPhase;          // Phase when message was sent
  metadata?: MessageMetadata;
}

interface MessageMetadata {
  isFollowUp?: boolean;              // True if this is a repregunta
  competencyId?: string;             // Which competency this message relates to
  escalationLevel?: 1 | 2 | 3;      // Set if message triggered escalation
  isOutOfScope?: boolean;            // True if candidate asked unanswerable question
}
```

### 1.3 SessionState

Persisted to DynamoDB after every message exchange (A4).

```typescript
interface SessionState {
  currentPhase: ConversationPhase;
  competencyOrder: string[];         // Shuffled competency IDs at session start (A3)
  currentCompetencyIndex: number;    // Index into competencyOrder
  followUpPending: boolean;          // True when waiting for follow-up response
  competenciesCovered: string[];     // IDs of fully covered competencies
  questionsAsked: number;            // Total questions asked
  followUpsAsked: number;            // Total follow-ups asked
  escalationLevel: 0 | 1 | 2 | 3;
  unansweredTopics: string[];        // Topics that triggered L1 escalation (for L2 detection)
  requirementsVerified: boolean;
  requirementResults: RequirementResult[];
  lastActivityAt: Date;
  knowledgeBaseLoaded: boolean;      // Whether KB was injected into system prompt
}

interface RequirementResult {
  requirementId: string;
  question: string;
  answer: string;
  passed: boolean;
}
```

### 1.4 EscalationRequest

```typescript
interface EscalationRequest {
  id: string;
  conversationId: string;
  tenantId: string;
  level: 1 | 2 | 3;
  trigger: 'info_not_available' | 'repeated_question' | 'explicit_human_request';
  candidateQuestion: string;         // The question that triggered escalation
  context: string;                   // Last N messages for context
  resolvedAt?: Date;
  createdAt: Date;
}
```

---

## 2. Evaluation Aggregate

**Location**: `src/domain/evaluation/`

### 2.1 Rubric

```typescript
interface Rubric {
  id: string;
  tenantId: string;
  name: string;
  template: 'bpo' | 'tech_saas' | 'custom';
  competencies: Competency[];
  createdAt: Date;
  updatedAt: Date;
}

interface Competency {
  id: string;
  name: string;
  weight: number;                    // 0-1, all weights must sum to 1.0
  criteria: CompetencyCriteria;
}

interface CompetencyCriteria {
  1: string;   // Level 1 description (lowest)
  2: string;
  3: string;
  4: string;
  5: string;   // Level 5 description (highest)
}
```

**BPO Template competencies** (B5):
- Comunicación (weight: 0.25)
- Orientación al cliente (weight: 0.25)
- Manejo de objeciones (weight: 0.20)
- Trabajo bajo presión (weight: 0.15)
- Adaptabilidad (weight: 0.15)

**Tech/SaaS Template competencies** (B5):
- Resolución de problemas (weight: 0.25)
- Comunicación técnica (weight: 0.20)
- Trabajo en equipo (weight: 0.20)
- Aprendizaje continuo (weight: 0.20)
- Orientación a resultados (weight: 0.15)

### 2.2 ExecutiveSummary

Generated once at the end of the full screening (B1 — batch evaluation).

```typescript
interface ExecutiveSummary {
  id: string;
  conversationId: string;
  candidateId: string;
  tenantId: string;
  globalScore: number;               // Weighted average, 1-5, 1 decimal (B3)
  competencyScores: CompetencyScore[];
  recommendation: Recommendation;
  keySignals: string[];              // 3-5 notable observations
  generatedAt: Date;
}

type Recommendation =
  | 'highly_recommended'   // globalScore >= 4.0 (B4)
  | 'recommended'          // globalScore >= 2.5 (B4)
  | 'not_recommended';     // globalScore < 2.5 (B4)

interface CompetencyScore {
  competencyId: string;
  competencyName: string;
  score: number;                     // 1-5 integer
  evidence: Evidence[];              // At least 1 required
}

interface Evidence {
  quote: string;                     // Verbatim candidate quote (extracted by LLM — B2)
  messageIndex: number;              // Position in conversation transcript
  relevance: string;                 // Why this quote supports the score
}
```

---

## 3. Campaign Aggregate

**Location**: `src/domain/campaign/`

### 3.1 Campaign

```typescript
interface Campaign {
  id: string;
  tenantId: string;
  name: string;
  roleDescription: string;           // Mandatory (E3)
  rubricId: string;                  // Mandatory (E3)
  telegramLink: string;              // Generated: https://t.me/{botUsername}?start={campaignId} (C2)
  status: CampaignStatus;
  knowledgeBaseContent?: string;     // Raw text injected into system prompt (C1)
  basicRequirements: BasicRequirement[];
  createdAt: Date;
  updatedAt: Date;
}

type CampaignStatus = 'draft' | 'active' | 'inactive' | 'archived';

interface BasicRequirement {
  id: string;
  question: string;                  // Question asked to candidate
  type: 'boolean' | 'text' | 'choice';
  mandatory: boolean;
  disqualifyIfFailed: boolean;       // If true, failing this ends the session
  options?: string[];                // For 'choice' type
}
```

---

## 4. Candidate Aggregate

**Location**: `src/domain/candidate/`

### 4.1 Candidate

Each screening attempt creates a new Candidate record (D1). Multiple records can exist for the same Telegram user across campaigns or after abandonment.

```typescript
interface Candidate {
  id: string;
  tenantId: string;
  telegramUserId: string;            // Telegram numeric user ID
  name: string;                      // Collected during onboarding (D2)
  state: CandidateState;
  campaignId: string;
  conversationId?: string;
  evaluationId?: string;
  reviewDecision?: ReviewDecision;
  createdAt: Date;
  updatedAt: Date;
}

type CandidateState =
  | 'initiated'
  | 'in_screening'
  | 'completed'
  | 'pending_review'
  | 'approved'
  | 'rejected';

interface ReviewDecision {
  decision: 'approved' | 'rejected';
  reviewerId: string;
  reason?: string;
  disagreesWithAI: boolean;
  disagreementReason?: string;       // Required text field when disagreesWithAI = true (E4)
  decidedAt: Date;
}
```

**Valid state transitions**:
```
initiated → in_screening → completed → pending_review → approved
                                                       → rejected
initiated → abandoned (failed requirements or inactivity)
in_screening → abandoned (inactivity)
```

---

## 5. Compliance Aggregate

**Location**: `src/domain/compliance/`

### 5.1 ConsentRecord

```typescript
interface ConsentRecord {
  id: string;
  candidateId: string;
  tenantId: string;
  granted: boolean;
  timestamp: Date;                   // Immutable — set at creation, never updated
  telegramUserId: string;
  conversationId: string;
}
```

### 5.2 AuditEvent

Append-only. Never updated or deleted.

```typescript
interface AuditEvent {
  id: string;
  tenantId: string;
  eventType: AuditEventType;
  entityId: string;
  entityType: 'conversation' | 'candidate' | 'evaluation' | 'campaign' | 'consent';
  details: Record<string, unknown>;
  timestamp: Date;                   // Immutable
  actorId: string;
  actorType: 'system' | 'recruiter' | 'candidate';
}

type AuditEventType =
  | 'consent_granted'
  | 'consent_denied'
  | 'screening_started'
  | 'screening_completed'
  | 'screening_abandoned'
  | 'requirements_failed'
  | 'evaluation_generated'
  | 'candidate_approved'
  | 'candidate_rejected'
  | 'escalation_l1'
  | 'escalation_l2'
  | 'escalation_l3';
```

---

## DynamoDB Table Design Summary

All tables use `tenantId#entityId` as the composite partition key (F2).

| Table | PK | SK | Notes |
|---|---|---|---|
| `conversations` | `tenantId#conversationId` | `createdAt` | Messages embedded |
| `campaigns` | `tenantId#campaignId` | `createdAt` | |
| `candidates` | `tenantId#candidateId` | `createdAt` | GSI: telegramUserId |
| `evaluations` | `tenantId#evaluationId` | `conversationId` | |
| `audit_events` | `tenantId#entityId` | `timestamp` | Append-only |
| `consent_records` | `tenantId#candidateId` | `timestamp` | |

**GSIs**:
- `candidates` → GSI on `telegramUserId` + `tenantId` (for duplicate/resume lookup)
- `candidates` → GSI on `campaignId` + `state` (for review queue)
- `conversations` → GSI on `campaignId` + `state` (for campaign metrics)
