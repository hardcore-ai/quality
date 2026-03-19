# Frontend Components — Unit 1: MVP Core

## Overview

The recruiter dashboard is built with Next.js 16 App Router, React Server Components (RSC), Tailwind CSS, and shadcn/ui. All components are in `src/app/(dashboard)/`.

---

## Component Hierarchy

```
app/
├── page.tsx                          # Root redirect → /login or /dashboard
├── login/
│   └── page.tsx                      # LoginPage
└── (dashboard)/
    ├── layout.tsx                    # DashboardLayout
    ├── campaigns/
    │   ├── page.tsx                  # CampaignListPage
    │   └── new/
    │       └── page.tsx              # CampaignCreatePage
    ├── review/
    │   └── page.tsx                  # ReviewQueuePage
    └── candidates/
        └── [candidateId]/
            └── page.tsx              # CandidateDetailPage
```

---

## 1. LoginPage

**Location**: `src/app/login/page.tsx`
**Type**: Server Component

**Purpose**: Entry point for unauthenticated users. Redirects to Cognito hosted UI via NextAuth.js.

**Props**: none

**State**: none (server component)

**Behavior**:
- If session exists → redirect to `/review`
- If no session → render login button

**UI Elements**:
- EntreVista AI logo + tagline
- "Iniciar sesión" button → triggers `signIn('cognito')` from NextAuth.js
- Minimal centered layout

**shadcn/ui components**: `Button`, `Card`

---

## 2. DashboardLayout

**Location**: `src/app/(dashboard)/layout.tsx`
**Type**: Server Component (with auth check)

**Purpose**: Shared layout for all authenticated dashboard pages. Validates session.

**Props**: `{ children: React.ReactNode }`

**Behavior**:
- Reads session via `getServerSession()`
- If no session → redirect to `/login`
- Renders sidebar navigation + main content area

**UI Elements**:
- Sidebar: logo, nav links (Campañas, Revisión, Candidatos)
- Top bar: tenant name, user email, logout button
- Main content area: `{children}`

**shadcn/ui components**: `Button`, `Badge` (for pending count in nav)

---

## 3. CampaignListPage

**Location**: `src/app/(dashboard)/campaigns/page.tsx`
**Type**: Server Component

**Purpose**: List all campaigns for the tenant with status and quick actions.

**Props**: none (reads tenantId from session)

**Data fetching**: Server-side via `CampaignRepository.findByTenant(tenantId)`

**UI Elements**:
- Page header: "Campañas" + "Nueva campaña" button
- Campaign table columns: Nombre, Rol, Estado, Candidatos, Enlace Telegram, Acciones
- Status badge per campaign (draft/active/inactive/archived)
- Copy Telegram link button per row
- Activate/Deactivate toggle per row
- Empty state: "No tienes campañas aún. Crea tu primera campaña."

**shadcn/ui components**: `Table`, `TableHeader`, `TableRow`, `TableCell`, `Badge`, `Button`

---

## 4. CampaignCreatePage

**Location**: `src/app/(dashboard)/campaigns/new/page.tsx`
**Type**: Client Component (`'use client'`)

**Purpose**: Form to create a new campaign.

**Props**: none

**State**:
```typescript
{
  name: string;              // required
  roleDescription: string;   // required
  rubricId: string;          // required — select from available rubrics
  knowledgeBaseContent: string; // optional — textarea
  basicRequirements: BasicRequirement[]; // optional — dynamic list
  isSubmitting: boolean;
  errors: Record<string, string>;
}
```

**Form fields**:

| Field | Type | Required | Validation |
|---|---|---|---|
| Nombre de la campaña | text input | Yes | Non-empty, max 100 chars |
| Descripción del rol | textarea | Yes | Non-empty, max 1000 chars |
| Rubrica de evaluación | select | Yes | Must select from list |
| Base de conocimiento | textarea | No | Max 10,000 chars |
| Requisitos básicos | dynamic list | No | Each item: question (required), type, mandatory flag |

**Behavior**:
- Validate on submit (client-side)
- POST to `/api/campaigns`
- On success → redirect to `/campaigns` with success toast
- On error → display field-level error messages

**Basic requirements builder**:
- "Agregar requisito" button adds a new row
- Each row: question text input + type select (Sí/No, Texto, Opción múltiple) + mandatory checkbox + delete button
- Drag-to-reorder not required for MVP

**shadcn/ui components**: `Input`, `Textarea`, `Select`, `SelectItem`, `Button`, `Checkbox`, `Card`, `Label`, `Badge`

**API integration**: `POST /api/campaigns` → `CreateCampaignUseCase`

---

## 5. ReviewQueuePage

**Location**: `src/app/(dashboard)/review/page.tsx`
**Type**: Client Component (`'use client'`)

**Purpose**: HITL review queue showing all candidates pending review.

**Props**: none

**State**:
```typescript
{
  candidates: CandidateReview[];
  filters: {
    campaignId?: string;
    recommendation?: Recommendation;
    scoreMin?: number;
    scoreMax?: number;
    dateFrom?: string;
    dateTo?: string;
  };
  sortBy: 'createdAt' | 'globalScore';
  sortOrder: 'asc' | 'desc';
  isLoading: boolean;
}
```

**Default state**: all pending candidates, sorted by `createdAt ASC` (FIFO — BR-DASH-01)

**UI Elements**:

Filter bar:
- Campaign dropdown (all campaigns for tenant)
- Recommendation filter: All / Altamente recomendado / Recomendado / No recomendado
- Score range: min/max number inputs
- Date range: from/to date pickers
- "Aplicar filtros" button + "Limpiar" link

Candidates table columns:
- Nombre del candidato
- Campaña
- Puntaje global (1 decimal)
- Recomendación (badge with color: green/yellow/red)
- Fecha de completado
- Acción: "Revisar" button → navigates to `/candidates/{candidateId}`

Empty state: "No hay candidatos pendientes de revisión."

**shadcn/ui components**: `Table`, `TableHeader`, `TableRow`, `TableCell`, `Badge`, `Button`, `Select`, `Input`

**API integration**: `GET /api/candidates?state=pending_review&...filters`

---

## 6. CandidateDetailPage

**Location**: `src/app/(dashboard)/candidates/[candidateId]/page.tsx`
**Type**: Client Component (`'use client'`)

**Purpose**: Full candidate detail view with executive summary, scores, evidence, transcript, and review decision form.

**Props**: `{ params: { candidateId: string } }`

**State**:
```typescript
{
  candidate: Candidate;
  evaluation: ExecutiveSummary;
  conversation: Conversation;
  decision: 'approved' | 'rejected' | null;
  reason: string;
  disagreementReason: string;        // Required when disagreesWithAI = true (E4)
  showDisagreementField: boolean;    // Auto-shown when decision differs from AI
  isSubmitting: boolean;
  submitError: string | null;
}
```

**Layout** (top to bottom):

**Header section**:
- Candidate name + campaign name
- Current state badge
- "← Volver a la cola" link

**Executive Summary card**:
- Global score (large, prominent): e.g., "4.2 / 5.0"
- Recommendation badge (color-coded)
- Key signals list (3-5 bullet points)

**Competency Scores section**:
- One card per competency
- Competency name + score (1-5) + weight
- Evidence quote(s) in blockquote style
- Relevance explanation in smaller text

**Full Transcript section** (E2 — full scrollable, always visible):
- Scrollable container with max-height
- Messages alternating left (candidate) / right (agent)
- Timestamp per message
- Evidence quotes highlighted in yellow within transcript

**Review Decision form**:
- Only shown if candidate.state = 'pending_review'
- Two buttons: "Aprobar" (green) / "Rechazar" (red)
- Optional reason textarea: "Motivo (opcional)"
- Disagreement field (E4):
  - Hidden by default
  - Auto-appears when selected decision differs from AI recommendation
  - Label: "Tu decisión difiere de la recomendación de la IA. Por favor indica el motivo:"
  - Required text input (cannot submit without it)
- "Confirmar decisión" submit button

**Behavior**:
- On decision button click: set `decision` state, check for disagreement
- If disagreement detected: show `disagreementReason` field, mark as required
- On submit: validate, POST to `/api/candidates/{candidateId}/review`
- On success: show success toast, redirect to `/review`

**shadcn/ui components**: `Card`, `CardHeader`, `CardContent`, `Badge`, `Button`, `Textarea`, `Input`, `Separator`, `ScrollArea`

**API integration**:
- `GET /api/candidates/{candidateId}` → candidate + evaluation + conversation
- `POST /api/candidates/{candidateId}/review` → `ReviewCandidateUseCase`

---

## 7. Shared UI Patterns

### 7.1 Recommendation Badge Colors

| Recommendation | Label | Color |
|---|---|---|
| `highly_recommended` | Altamente recomendado | Green (`bg-green-100 text-green-800`) |
| `recommended` | Recomendado | Yellow (`bg-yellow-100 text-yellow-800`) |
| `not_recommended` | No recomendado | Red (`bg-red-100 text-red-800`) |

### 7.2 Candidate State Badge Colors

| State | Label | Color |
|---|---|---|
| `pending_review` | Pendiente de revisión | Blue |
| `approved` | Aprobado | Green |
| `rejected` | Rechazado | Red |
| `abandoned` | Abandonado | Gray |
| `in_screening` | En entrevista | Purple |

### 7.3 Form Validation Pattern

All forms use client-side validation before API calls:
- Required fields: show red border + error message below field
- API errors: show toast notification (top-right, auto-dismiss 5s)
- Success: show green toast + redirect

### 7.4 Loading States

- Page-level loading: skeleton cards (shadcn/ui `Skeleton`)
- Button loading: spinner inside button, button disabled
- Table loading: skeleton rows

### 7.5 Empty States

Each list/table has a centered empty state with:
- Icon (from lucide-react)
- Message in Spanish
- CTA button where applicable

---

## 8. API Route → Use Case Mapping

| Route | Method | Use Case | Auth |
|---|---|---|---|
| `/api/campaigns` | GET | `ListCampaignsUseCase` | Required |
| `/api/campaigns` | POST | `CreateCampaignUseCase` | Required |
| `/api/campaigns/[id]` | PATCH | `UpdateCampaignUseCase` | Required |
| `/api/candidates` | GET | `ListCandidatesForReviewUseCase` | Required |
| `/api/candidates/[id]` | GET | `GetCandidateDetailUseCase` | Required |
| `/api/candidates/[id]/review` | POST | `ReviewCandidateUseCase` | Required |
| `/api/telegram` | POST | `TelegramWebhookHandler` | Webhook secret |
| `/api/auth/[...nextauth]` | * | NextAuth.js handlers | Public |
