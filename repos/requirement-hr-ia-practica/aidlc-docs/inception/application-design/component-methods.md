# Component Methods — EntreVista AI

> **Note**: Method signatures define high-level interfaces. Detailed business rules will be defined in Functional Design (CONSTRUCTION phase).

---

## Domain Layer Methods

### C1: Conversation Domain

```typescript
// domain/conversation/entities/Conversation.ts
interface Conversation {
  id: string;
  tenantId: string;
  campaignId: string;
  candidateId: string;
  state: ConversationState;
  messages: Message[];
  sessionState: SessionState;
  createdAt: Date;
  updatedAt: Date;
}

type ConversationState = 'active' | 'paused' | 'abandoned' | 'completed';

interface SessionState {
  currentPhase: 'onboarding' | 'consent' | 'verification' | 'screening' | 'closing';
  competenciesCovered: string[];
  partialScores: Record<string, number>;
  questionsAsked: number;
  followUpsAsked: number;
  escalationLevel: 0 | 1 | 2 | 3;
  lastActivityAt: Date;
}

interface Message {
  role: 'agent' | 'candidate';
  content: string;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}
```

```typescript
// domain/conversation/ConversationRules.ts
canProceedToScreening(consent: boolean, requirementsMet: boolean): boolean;
shouldAskFollowUp(sessionState: SessionState, rubric: Rubric): boolean;
getNextCompetency(sessionState: SessionState, rubric: Rubric): string | null;
isScreeningComplete(sessionState: SessionState, rubric: Rubric): boolean;
getEscalationLevel(sessionState: SessionState): 0 | 1 | 2 | 3;
shouldTriggerReEngagement(lastActivityAt: Date): 'none' | '5min' | '24h' | '48h' | '72h_abandon';
```

### C2: Evaluation Domain

```typescript
// domain/evaluation/entities/Rubric.ts
interface Rubric {
  id: string;
  tenantId: string;
  name: string;
  competencies: Competency[];
  template: 'bpo' | 'tech' | 'custom';
}

interface Competency {
  id: string;
  name: string;
  weight: number;
  criteria: Record<1 | 2 | 3 | 4 | 5, string>; // level → description
}

// domain/evaluation/entities/ExecutiveSummary.ts
interface ExecutiveSummary {
  conversationId: string;
  globalScore: number;
  competencyScores: CompetencyScore[];
  recommendation: 'highly_recommended' | 'recommended' | 'not_recommended';
  keySignals: string[];
  generatedAt: Date;
}

interface CompetencyScore {
  competencyId: string;
  competencyName: string;
  score: number; // 1-5
  evidence: Evidence[];
}

interface Evidence {
  quote: string;        // verbatim candidate quote
  messageIndex: number; // reference to transcript position
  relevance: string;    // why this quote supports the score
}
```

```typescript
// domain/evaluation/EvaluationRules.ts
scoreResponse(response: string, competency: Competency, context: string[]): CompetencyScore;
calculateGlobalScore(scores: CompetencyScore[], rubric: Rubric): number;
determineRecommendation(globalScore: number, scores: CompetencyScore[]): Recommendation;
validateEvidence(score: CompetencyScore): boolean; // every score must have ≥1 quote
```

### C3: Campaign Domain

```typescript
// domain/campaign/entities/Campaign.ts
interface Campaign {
  id: string;
  tenantId: string;
  name: string;
  roleDescription: string;
  rubricId: string;
  telegramLink: string;
  status: CampaignStatus;
  knowledgeBaseId?: string;
  basicRequirements: BasicRequirement[];
  createdAt: Date;
  updatedAt: Date;
}

type CampaignStatus = 'draft' | 'active' | 'inactive' | 'archived';

interface BasicRequirement {
  field: string;
  question: string;
  type: 'text' | 'boolean' | 'choice';
  mandatory: boolean;
  options?: string[];
}
```

```typescript
// domain/campaign/CampaignRules.ts
generateTelegramLink(campaignId: string, botUsername: string): string;
canActivate(campaign: Campaign): boolean;
canArchive(campaign: Campaign): boolean;
```

### C4: Candidate Domain

```typescript
// domain/candidate/entities/Candidate.ts
interface Candidate {
  id: string;
  tenantId: string;
  telegramUserId: string;
  name?: string;
  state: CandidateState;
  campaignId: string;
  conversationId?: string;
  evaluationId?: string;
  reviewDecision?: ReviewDecision;
  createdAt: Date;
  updatedAt: Date;
}

type CandidateState = 'initiated' | 'in_screening' | 'completed' | 'pending_review' | 'approved' | 'rejected';

interface ReviewDecision {
  decision: 'approved' | 'rejected';
  reviewerId: string;
  reason?: string;
  disagreesWithAI: boolean;
  disagreementReason?: string;
  decidedAt: Date;
}
```

```typescript
// domain/candidate/CandidateRules.ts
canTransitionTo(currentState: CandidateState, targetState: CandidateState): boolean;
isDuplicate(telegramUserId: string, tenantId: string, existingCandidates: Candidate[]): boolean;
```

### C5: Compliance Domain

```typescript
// domain/compliance/entities/ConsentRecord.ts
interface ConsentRecord {
  candidateId: string;
  tenantId: string;
  granted: boolean;
  timestamp: Date; // immutable
  telegramUserId: string;
}

// domain/compliance/entities/AuditEvent.ts
interface AuditEvent {
  id: string;
  tenantId: string;
  eventType: AuditEventType;
  entityId: string;
  entityType: 'conversation' | 'candidate' | 'evaluation' | 'campaign';
  details: Record<string, unknown>;
  timestamp: Date; // immutable
  actorId: string;
  actorType: 'system' | 'recruiter' | 'candidate';
}

type AuditEventType =
  | 'consent_granted' | 'consent_denied'
  | 'screening_started' | 'screening_completed' | 'screening_abandoned'
  | 'evaluation_generated'
  | 'candidate_approved' | 'candidate_rejected'
  | 'escalation_requested'
  | 'data_purged';
```

---

## Application Layer Methods

### C6: Conversation Application

```typescript
// application/conversation/StartScreeningUseCase.ts
execute(input: { telegramUserId: string; campaignId: string }): Promise<ConversationStartResult>;

// application/conversation/ProcessMessageUseCase.ts
execute(input: { conversationId: string; message: string }): Promise<AgentResponse>;

// application/conversation/HandleEscalationUseCase.ts
execute(input: { conversationId: string; reason: string }): Promise<void>;

// application/conversation/ResumeSessionUseCase.ts
execute(input: { conversationId: string }): Promise<SessionResumeResult>;

// application/conversation/CompleteScreeningUseCase.ts
execute(input: { conversationId: string }): Promise<ExecutiveSummary>;
```

### C7: Evaluation Application

```typescript
// application/evaluation/EvaluateResponseUseCase.ts
execute(input: { conversationId: string; response: string; competencyId: string }): Promise<CompetencyScore>;

// application/evaluation/GenerateSummaryUseCase.ts
execute(input: { conversationId: string }): Promise<ExecutiveSummary>;

// application/evaluation/GetEvaluationDetailUseCase.ts
execute(input: { evaluationId: string; tenantId: string }): Promise<EvaluationDetail>;
```

### C8: Campaign Application

```typescript
// application/campaign/CreateCampaignUseCase.ts
execute(input: CreateCampaignInput): Promise<Campaign>;

// application/campaign/UpdateCampaignUseCase.ts
execute(input: { campaignId: string; tenantId: string; updates: Partial<Campaign> }): Promise<Campaign>;

// application/campaign/GetCampaignMetricsUseCase.ts
execute(input: { campaignId: string; tenantId: string }): Promise<CampaignMetrics>;

// application/campaign/ManageKnowledgeBaseUseCase.ts
execute(input: { campaignId: string; action: 'upload' | 'delete'; document?: Buffer }): Promise<void>;
```

### C9: Candidate Application

```typescript
// application/candidate/ListCandidatesForReviewUseCase.ts
execute(input: { tenantId: string; filters: ReviewFilters }): Promise<PaginatedResult<CandidateReview>>;

// application/candidate/ReviewCandidateUseCase.ts
execute(input: { candidateId: string; tenantId: string; decision: ReviewDecision }): Promise<void>;
```

### C10: Compliance Application

```typescript
// application/compliance/RecordConsentUseCase.ts
execute(input: { candidateId: string; tenantId: string; granted: boolean }): Promise<ConsentRecord>;

// application/compliance/LogAuditEventUseCase.ts
execute(input: Omit<AuditEvent, 'id' | 'timestamp'>): Promise<void>;

// application/compliance/GetAuditTrailUseCase.ts
execute(input: { entityId: string; tenantId: string }): Promise<AuditEvent[]>;
```

---

## Infrastructure Layer Methods

### C11: Telegram Infrastructure

```typescript
// infrastructure/telegram/TelegramWebhookHandler.ts
handleUpdate(update: TelegramUpdate): Promise<void>;

// infrastructure/telegram/TelegramBotService.ts
sendMessage(chatId: number, text: string): Promise<void>;
sendMessageWithKeyboard(chatId: number, text: string, options: string[]): Promise<void>;
```

### C12: OpenAI Infrastructure

```typescript
// infrastructure/openai/OpenAIChatClient.ts
generateResponse(input: {
  systemPrompt: string;
  sessionState: SessionState;
  recentMessages: Message[];
  knowledgeContext?: string;
}): Promise<{ response: string; usage: TokenUsage }>;

// infrastructure/openai/OpenAIEmbeddingClient.ts
generateEmbedding(text: string): Promise<number[]>;
generateEmbeddings(texts: string[]): Promise<number[][]>;
```

### C13: DynamoDB Infrastructure

```typescript
// infrastructure/dynamodb/repositories/
// Each domain entity has a corresponding repository implementing the domain interface

interface ConversationRepository {
  save(conversation: Conversation): Promise<void>;
  findById(id: string, tenantId: string): Promise<Conversation | null>;
  findByTelegramUser(telegramUserId: string, campaignId: string): Promise<Conversation | null>;
  updateState(id: string, tenantId: string, state: ConversationState): Promise<void>;
  updateSessionState(id: string, tenantId: string, sessionState: SessionState): Promise<void>;
}

interface CampaignRepository {
  save(campaign: Campaign): Promise<void>;
  findById(id: string, tenantId: string): Promise<Campaign | null>;
  findByTenant(tenantId: string, status?: CampaignStatus): Promise<Campaign[]>;
  update(id: string, tenantId: string, updates: Partial<Campaign>): Promise<void>;
}

interface CandidateRepository {
  save(candidate: Candidate): Promise<void>;
  findById(id: string, tenantId: string): Promise<Candidate | null>;
  findByTelegramUser(telegramUserId: string, tenantId: string): Promise<Candidate[]>;
  findForReview(tenantId: string, filters: ReviewFilters): Promise<PaginatedResult<Candidate>>;
  updateState(id: string, tenantId: string, state: CandidateState): Promise<void>;
}

interface EvaluationRepository {
  save(summary: ExecutiveSummary): Promise<void>;
  findByConversation(conversationId: string, tenantId: string): Promise<ExecutiveSummary | null>;
}

interface AuditEventRepository {
  append(event: AuditEvent): Promise<void>; // append-only
  findByEntity(entityId: string, tenantId: string): Promise<AuditEvent[]>;
}
```

### C14: Auth Infrastructure

```typescript
// infrastructure/auth/authOptions.ts
// NextAuth.js configuration with Cognito provider
// Cognito User Pool ID and App Client ID from environment variables

// infrastructure/auth/middleware.ts
withAuth(handler: NextApiHandler): NextApiHandler;
getTenantId(session: Session): string;
requireRole(role: string): MiddlewareFunction;
validateCognitoToken(token: string): Promise<CognitoTokenPayload>;
```
