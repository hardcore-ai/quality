# Business Logic Model — Unit 1: MVP Core

## Overview

This document describes the end-to-end business flows, use case orchestration, and data transformations for the MVP Core unit. It is technology-agnostic — no infrastructure specifics.

---

## 1. Screening Flow (End-to-End)

### 1.1 New Candidate Initiates Screening

```
TRIGGER: Candidate clicks Telegram deep link → sends /start {campaignId}

FLOW:
  1. Extract campaignId from /start parameter
  2. Load Campaign by campaignId
     - IF campaign not found OR status != 'active' → send error message, end
  3. Extract telegramUserId from Telegram update
  4. Create new Candidate record (state: 'initiated')
  5. Create new Conversation record (state: 'active', phase: 'onboarding')
  6. Log AuditEvent: screening_started
  7. Ask for candidate's name: "¡Hola! Soy el asistente de EntreVista AI. ¿Cuál es tu nombre?"
  8. Persist SessionState
  9. Send onboarding message to candidate
```

### 1.2 Onboarding Phase

```
TRIGGER: Candidate sends their name

FLOW:
  1. Store name in Candidate.name
  2. Send AI disclosure + process explanation:
     "Hola {name}. Soy un asistente de inteligencia artificial que realizará
      una entrevista de preselección para [roleDescription]. 
      Esta conversación será evaluada por el equipo de reclutamiento.
      Tienes derecho a no participar en cualquier momento."
  3. Transition phase: onboarding → consent
  4. Ask for consent: "¿Aceptas participar en esta entrevista? (Sí / No)"
  5. Persist SessionState
```

### 1.3 Consent Phase

```
TRIGGER: Candidate responds to consent question

FLOW — Consent GRANTED (response matches affirmative pattern):
  1. Create ConsentRecord (granted: true, timestamp: now)
  2. Log AuditEvent: consent_granted
  3. Transition phase: consent → verification
  4. IF campaign has basicRequirements:
       Ask first requirement question
     ELSE:
       Transition phase: verification → screening
       Initialize screening (see 1.4)
  5. Persist SessionState

FLOW — Consent DENIED (response matches negative pattern):
  1. Create ConsentRecord (granted: false, timestamp: now)
  2. Log AuditEvent: consent_denied
  3. Send: "Gracias por tu tiempo. Si cambias de opinión, puedes volver cuando quieras."
  4. Transition conversation: state = 'abandoned', phase = 'consent'
  5. Transition candidate: state = 'abandoned'
  6. Persist SessionState

Affirmative patterns: "sí", "si", "yes", "acepto", "de acuerdo", "claro", "ok"
Negative patterns: "no", "no acepto", "no quiero", "prefiero no"
Ambiguous → ask for clarification once, then treat as denied
```

### 1.4 Verification Phase

```
TRIGGER: Candidate responds to a requirement question

FLOW — Per requirement:
  1. Evaluate candidate's response against requirement
     - boolean: check for yes/no
     - text: store as-is
     - choice: match against options
  2. Store RequirementResult in SessionState
  3. IF mandatory requirement FAILED:
       Log AuditEvent: requirements_failed
       Send: "Gracias por tu interés. En este momento no cumples con los requisitos
              mínimos para esta posición. Te deseamos mucho éxito."
       Transition conversation: state = 'abandoned'
       Transition candidate: state = 'abandoned'
       END FLOW
  4. IF more requirements pending:
       Ask next requirement question
     ELSE:
       All requirements passed → transition phase: verification → screening
       Initialize screening (see 1.5)
  5. Persist SessionState
```

### 1.5 Screening Phase Initialization

```
FLOW:
  1. Load Rubric for campaign
  2. Shuffle competency IDs → store in sessionState.competencyOrder
  3. Set sessionState.currentCompetencyIndex = 0
  4. Set sessionState.followUpPending = false
  5. Update candidate.state = 'in_screening'
  6. Generate first competency question via LLM:
     - Input: competency name + criteria + role description + knowledge base (if any)
     - Output: natural language question in Spanish neutro
  7. Send question to candidate
  8. Persist SessionState
```

### 1.6 Screening Phase — Message Processing

```
TRIGGER: Candidate sends a message during screening phase

PRE-CHECK (runs before any routing):
  - IF message matches AI disclosure query → respond truthfully, continue
  - IF message matches explicit human request → trigger L3 escalation, continue

FLOW — Normal message:
  1. Check if followUpPending = true:
     YES (receiving follow-up response):
       a. Store message in conversation
       b. Mark current competency as covered: add to competenciesCovered
       c. Set followUpPending = false
       d. Increment currentCompetencyIndex
       e. IF isScreeningComplete (all competencies covered):
            → Transition to closing phase (see 1.7)
          ELSE:
            → Generate next competency question
            → Send question
     NO (receiving main question response):
       a. Store message in conversation
       b. Check for out-of-scope / unanswerable content:
            IF out-of-scope → handle escalation (see 1.8)
            ELSE → continue
       c. Generate follow-up question via LLM
       d. Set followUpPending = true
       e. Send follow-up question
  2. Persist SessionState after every message
```

### 1.7 Closing Phase

```
FLOW:
  1. Transition phase: screening → closing
  2. Send closing message:
     "¡Gracias {name}! Has completado la entrevista de preselección.
      El equipo de reclutamiento revisará tu perfil y se pondrá en contacto contigo
      en los próximos días. ¡Mucho éxito!"
  3. Transition conversation: state = 'completed'
  4. Transition candidate: state = 'completed'
  5. Log AuditEvent: screening_completed
  6. Trigger evaluation: GenerateSummaryUseCase (see Section 2)
  7. After summary generated: transition candidate: state = 'pending_review'
  8. Persist SessionState
```

### 1.8 Escalation Handling

```
FLOW — Level 1 (info not available):
  1. Add topic to sessionState.unansweredTopics
  2. Log AuditEvent: escalation_l1
  3. Check if same topic appeared before in unansweredTopics:
     YES → trigger Level 2
     NO  → respond: "No tengo esa información. El equipo de reclutamiento
                     te dará los detalles en la siguiente etapa."
  4. Continue conversation normally

FLOW — Level 2 (repeated question):
  1. Set sessionState.escalationLevel = 2
  2. Log AuditEvent: escalation_l2
  3. Respond: "Entiendo que necesitas más información sobre este tema.
               Un reclutador podrá ayudarte en la siguiente etapa del proceso."
  4. Continue conversation normally

FLOW — Level 3 (explicit human request):
  1. Set sessionState.escalationLevel = 3
  2. Create EscalationRequest record
  3. Log AuditEvent: escalation_l3
  4. Respond: "Entendido. He notificado al equipo de reclutamiento.
               Tu progreso está guardado y un reclutador se pondrá en contacto contigo."
  5. Continue conversation (session NOT terminated)
```

---

## 2. Evaluation Flow

### 2.1 Executive Summary Generation

```
TRIGGER: CompleteScreeningUseCase → calls GenerateSummaryUseCase

FLOW:
  1. Load full Conversation (all messages)
  2. Load Rubric for campaign
  3. FOR EACH competency in rubric:
     a. Extract candidate messages related to this competency
        (messages where metadata.competencyId = competency.id)
     b. Call LLM with:
        - Competency name + criteria (levels 1-5)
        - Candidate messages for this competency
        - Full conversation context (for coherence)
     c. LLM returns structured JSON: { score, evidence: { quote, messageIndex, relevance } }
     d. Validate evidence (BR-EVAL-03): retry once if no quote
     e. Store CompetencyScore
  4. Calculate globalScore (BR-EVAL-04):
     globalScore = round(sum(score_i × weight_i), 1)
  5. Determine recommendation (BR-EVAL-05)
  6. Generate keySignals via LLM (single call with full transcript + scores)
  7. Save ExecutiveSummary to DynamoDB
  8. Log AuditEvent: evaluation_generated
  9. Return ExecutiveSummary
```

### 2.2 LLM Prompt Structure for Scoring

```
System prompt:
  "Eres un evaluador experto de entrevistas de trabajo. Tu tarea es evaluar
   la respuesta de un candidato para la competencia '{competencyName}'.
   
   Criterios de evaluación:
   Nivel 1: {criteria[1]}
   Nivel 2: {criteria[2]}
   Nivel 3: {criteria[3]}
   Nivel 4: {criteria[4]}
   Nivel 5: {criteria[5]}
   
   Responde ÚNICAMENTE con JSON válido en este formato:
   {
     \"score\": <número del 1 al 5>,
     \"evidence\": {
       \"quote\": \"<cita textual del candidato>\",
       \"messageIndex\": <índice del mensaje>,
       \"relevance\": \"<explicación de por qué esta cita apoya el puntaje>\"
     }
   }"

User prompt:
  "Mensajes del candidato para esta competencia:
   [lista de mensajes con índices]
   
   Contexto completo de la conversación:
   [últimos N mensajes]"
```

---

## 3. Campaign Management Flow

### 3.1 Create Campaign

```
FLOW:
  1. Validate mandatory fields: name, roleDescription, rubricId (BR-CAMP-01)
  2. Verify rubricId exists and belongs to tenantId
  3. Generate telegramLink: `https://t.me/${BOT_USERNAME}?start=${campaignId}` (BR-CAMP-02)
  4. Set status = 'draft'
  5. Save Campaign to DynamoDB
  6. Log AuditEvent
  7. Return Campaign with generated telegramLink
```

### 3.2 Activate Campaign

```
FLOW:
  1. Load Campaign
  2. Validate status transition: draft/inactive → active (BR-CAMP-03)
  3. Validate mandatory fields present
  4. Set status = 'active'
  5. Save Campaign
```

---

## 4. HITL Review Flow

### 4.1 List Candidates for Review

```
FLOW:
  1. Extract tenantId from authenticated session
  2. Query DynamoDB: candidates WHERE tenantId = X AND state = 'pending_review'
  3. Default sort: createdAt ASC (oldest first — FIFO, BR-DASH-01)
  4. Apply optional filters: campaignId, recommendation, scoreRange, dateRange
  5. For each candidate: join with ExecutiveSummary (globalScore, recommendation)
  6. Return paginated result
```

### 4.2 Submit Review Decision

```
FLOW:
  1. Load Candidate (validate state = 'pending_review')
  2. Load ExecutiveSummary for candidate
  3. Determine if decision disagrees with AI recommendation (BR-CAND-04):
     - approved + not_recommended → disagreement
     - rejected + recommended/highly_recommended → disagreement
  4. IF disagreement AND disagreementReason is empty → reject with validation error
  5. Create ReviewDecision object
  6. Update Candidate: state = decision, reviewDecision = ReviewDecision
  7. Log AuditEvent: candidate_approved / candidate_rejected
  8. Return updated Candidate
```

---

## 5. Authentication Flow

### 5.1 Dashboard Login

```
FLOW:
  1. Unauthenticated request → NextAuth.js middleware redirects to Cognito hosted UI
  2. Recruiter authenticates with Cognito credentials
  3. Cognito returns authorization code → NextAuth.js exchanges for tokens
  4. NextAuth.js stores session (JWT with tenantId claim from Cognito user attributes)
  5. All subsequent API requests include session cookie
  6. API route middleware validates session and extracts tenantId
```

### 5.2 Tenant Extraction

```
FLOW:
  1. Read Cognito JWT from NextAuth.js session
  2. Extract custom attribute: custom:tenantId
  3. Use tenantId to scope all DynamoDB queries
  4. IF tenantId missing from token → return 403 Forbidden
```

---

## 6. Prompt Architecture

### 6.1 Screening Conversation Prompt

```
System prompt (built once at session start):
  - Role: "Eres un entrevistador de IA para [roleDescription]..."
  - Anti-hallucination rules: "Solo puedes responder con información del knowledge base..."
  - Escalation rules: "Si el candidato pregunta algo que no puedes responder..."
  - Language rules: "Responde siempre en español neutro..."
  - Knowledge base content (if any): full text injected here (C1)
  - Current rubric context: competency names and what to assess

User messages: last N messages from conversation (sliding window)
  - N = min(all messages, token budget)
  - Full context maintained for MVP given short session length
```

### 6.2 Follow-Up Question Generation Prompt

```
System prompt: same as screening prompt
User prompt:
  "El candidato respondió a la pregunta sobre '{competencyName}':
   '{candidateResponse}'
   
   Genera una pregunta de seguimiento que profundice en su experiencia.
   La pregunta debe ser conversacional, en español neutro, y no más de 2 oraciones."
```

### 6.3 Next Competency Question Generation Prompt

```
System prompt: same as screening prompt
User prompt:
  "Ahora debes preguntar sobre la competencia: '{competencyName}'
   Descripción: '{competencyCriteria[3]}' (nivel medio como referencia)
   
   Genera una pregunta abierta y conversacional en español neutro.
   No menciones el nombre de la competencia directamente."
```

---

## 7. Data Flow Diagram

```
Telegram Update
    |
    v
[TelegramWebhookHandler]
    |
    v
[ScreeningOrchestrator]
    |
    +---> [Load Campaign + Rubric] ---> DynamoDB
    |
    +---> [Load/Create Conversation] ---> DynamoDB
    |
    +---> [Route by Phase]
    |         |
    |         +-- onboarding  --> [Collect name, send disclosure]
    |         +-- consent     --> [Record consent] --> DynamoDB (ConsentRecord)
    |         +-- verification --> [Check requirements]
    |         +-- screening   --> [Build prompt] --> [OpenAI] --> [Response]
    |         +-- closing     --> [GenerateSummary] --> [OpenAI x N competencies]
    |
    +---> [Log AuditEvent] ---> DynamoDB (audit_events)
    |
    +---> [Persist SessionState] ---> DynamoDB (conversations)
    |
    v
[TelegramBotService] --> Telegram API --> Candidate
```
