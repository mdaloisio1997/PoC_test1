# Design — Hiring Management Web Application

## Purpose of This Document

This document defines the **technical and functional design** of the Hiring Management Web Application. It covers: application architecture, data model, API design, key UI/UX patterns, workflow state machines, notification system design, and integration architecture. It is the primary reference for the development team during implementation.

---

## 1. Architecture Overview

### 1.1 Application Architecture

The application follows a **layered monolith** architecture with clear internal module boundaries, optimized for a small-to-medium team with predictable traffic patterns. The monolith can be decomposed into microservices in a future phase without breaking the external API contract.

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                         │
│          React SPA (Vite + Tailwind CSS)                    │
│          Candidate Portal (public, server-rendered)         │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS / REST + JSON
┌──────────────────────▼──────────────────────────────────────┐
│                      API LAYER (Node.js / Express)          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Auth     │ │Workflow  │ │Analytics │ │Notification  │  │
│  │ Module   │ │Engine    │ │Module    │ │Dispatcher    │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
│  │Candidate │ │Evaluation│ │Onboarding│                    │
│  │Module    │ │Module    │ │Module    │                    │
│  └──────────┘ └──────────┘ └──────────┘                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   PERSISTENCE LAYER                         │
│  PostgreSQL (primary data)    Redis (sessions, SLA timers)  │
│  S3-compatible storage (CV files, offer PDFs, audit logs)   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Deployment Architecture

```
                    [Nginx reverse proxy]
                          │
              ┌───────────┴──────────────┐
              │                          │
        [Node.js App]              [Node.js App]
         (primary)                  (replica)
              │                          │
              └─────────────┬────────────┘
                            │
                    [PostgreSQL (primary)]
                    [PostgreSQL (read replica)]
                            │
                         [Redis]
```

All components containerized via Docker. `docker-compose.yml` provided for local development; production deployment via Docker Swarm or Kubernetes (both supported).

---

## 2. Technology Stack — Detailed

| Concern | Technology | Version | Notes |
|---------|-----------|---------|-------|
| Runtime | Node.js | 20 LTS | Required by client |
| Framework | Express.js | 4.x | Minimal footprint, middleware ecosystem |
| Language | TypeScript | 5.x | Type safety across API layer |
| ORM | Prisma | 5.x | Schema migrations, type-safe queries |
| Database | PostgreSQL | 16 | Full ACID, JSONB for flexible fields |
| Cache / Queue | Redis | 7.x | Session store, SLA timer queue, notification queue |
| Job Scheduler | BullMQ | 4.x | Background jobs: SLA checks, email dispatch, report generation |
| Authentication | JWT (access) + Refresh tokens | — | Stored in httpOnly cookies |
| Password hashing | bcrypt | — | Cost factor 12 |
| Validation | Zod | 3.x | Request schema validation at API boundary |
| Frontend | React | 18.x | SPA for internal users |
| Bundler | Vite | 5.x | Fast HMR, optimized builds |
| CSS | Tailwind CSS | 3.x | Utility-first, consistent design tokens |
| Charts | Recharts | 2.x | KPI dashboard visualizations |
| Email | Nodemailer + MJML | — | Responsive HTML email templates |
| PDF Generation | Puppeteer | — | Offer letter PDF rendering from HTML template |
| File Uploads | Multer + AWS S3 SDK | — | CV and document storage |
| Testing | Vitest + Supertest | — | Unit + integration testing |
| Linting | ESLint + Prettier | — | Enforced in CI |
| API Documentation | Swagger / OpenAPI 3.0 | — | Auto-generated from route definitions |

---

## 3. Data Model

### 3.1 Core Entities and Relationships

```
User ──────────────── UserRole (HRBP, Recruiter, LM, SME, Finance, Legal, Director, IT, Facility, Payroll)
  │
  ├── RFR (Richiesta Fabbisogno Risorse)
  │     ├── RFRApprovalStep [] (step, approver, status, timestamp, notes)
  │     └── JobRequisition
  │           ├── JobDescription (sections: summary, responsibilities, requirements, compensation)
  │           ├── ScreeningQuestion []
  │           ├── PublicationChannel [] (channel, url, publishedAt, status)
  │           ├── CandidateApplication []
  │           │     ├── Candidate (profile, CV file ref, GDPR consent)
  │           │     ├── ApplicationSource (Direct | Referral | Agency | Internal)
  │           │     ├── ScreeningEvaluation
  │           │     ├── ShortlistDecision [] (reviewer, decision, notes)
  │           │     └── InterviewRound []
  │           │           ├── InterviewSchedule (datetime, format, interviewers)
  │           │           └── EvaluationScorecard (scores per dimension, composite, submittedAt)
  │           ├── AlignmentMeeting (date, participants, selectedCandidate, rationale)
  │           └── Offer
  │                 ├── OfferApprovalStep []
  │                 └── OfferResponse (Accept | Decline | CounterOffer, signature, timestamp)
  │
  ├── InternalTalentPool
  │     └── TalentProfile (category, skills, availability, linkedUserId?)
  │
  ├── OnboardingBoard
  │     └── OnboardingTrack [] (IT | Facility | Payroll | HR | LM)
  │           └── OnboardingTask [] (title, owner, dueDate, completedAt)
  │
  └── Referral
        ├── referrer (User)
        ├── referred (Candidate)
        └── incentive (amount, vestingDate, paidAt)
```

### 3.2 Key Schema Definitions (Prisma)

```prisma
// Core workflow entity
model RFR {
  id              String        @id @default(cuid())
  createdBy       User          @relation(fields: [createdById], references: [id])
  createdById     String
  status          RFRStatus     // DRAFT | PENDING_HRBP | PENDING_FINANCE | PENDING_LEGAL | PENDING_DIRECTION | APPROVED | REJECTED
  roleTitle       String
  seniorityLevel  SeniorityLevel // JUNIOR | MID | SENIOR | SPECIALIST | EXECUTIVE
  costCenter      String
  ralMin          Decimal
  ralMax          Decimal
  requiredSkills  Json          // Array of skill tags
  softSkills      Json
  targetStartDate DateTime
  commessaRef     String?       // Optional client reference
  justification   String
  approvalSteps   RFRApprovalStep[]
  jobRequisition  JobRequisition?
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  auditLog        AuditEntry[]
}

model CandidateApplication {
  id                String              @id @default(cuid())
  jobRequisitionId  String
  jobRequisition    JobRequisition      @relation(fields: [jobRequisitionId], references: [id])
  candidateId       String
  candidate         Candidate           @relation(fields: [candidateId], references: [id])
  source            ApplicationSource   // DIRECT | REFERRAL | AGENCY | INTERNAL
  referralId        String?
  status            ApplicationStatus   // NEW | SCREENING | SHORTLISTED | INTERVIEWING | OFFER_SENT | HIRED | REJECTED | TALENT_POOL
  screeningEval     ScreeningEvaluation?
  shortlistDecisions ShortlistDecision[]
  interviewRounds   InterviewRound[]
  createdAt         DateTime            @default(now())
}

model EvaluationScorecard {
  id                String          @id @default(cuid())
  interviewRoundId  String          @unique
  interviewRound    InterviewRound  @relation(fields: [interviewRoundId], references: [id])
  submittedBy       User            @relation(fields: [submittedById], references: [id])
  submittedById     String
  submittedAt       DateTime?
  // Dimension scores (1-5)
  hardSkillScore    Decimal?        // Weight: 0.50
  softSkillScore    Decimal?        // Weight: 0.25
  culturalFitScore  Decimal?        // Weight: 0.15
  motivationScore   Decimal?        // Weight: 0.10
  compositeScore    Decimal?        // Computed: (HS*0.5)+(SS*0.25)+(CF*0.15)+(MOT*0.10)
  notes             String?
  criteriaScores    Json            // [{criterion, score, note}] for granular breakdown
  slaDeadline       DateTime        // submittedAt + 24h from interview
  reminderSentAt    DateTime?
  escalatedAt       DateTime?
}
```

### 3.3 Status State Machines

**RFR Status Flow**:
```
DRAFT → PENDING_HRBP → PENDING_FINANCE → [PENDING_LEGAL] → PENDING_DIRECTION → APPROVED
                ↘           ↘                   ↘                  ↘
              REJECTED    REJECTED            REJECTED           REJECTED
                ↓            ↓                    ↓                  ↓
              REVISION     REVISION             REVISION           REVISION
                                                                      ↓
                                                                  (back to PENDING_HRBP)
```

**Application Status Flow**:
```
NEW → SCREENING → SHORTLISTED → INTERVIEWING → OFFER_PENDING → OFFER_SENT → HIRED
 ↓        ↓            ↓              ↓               ↓
REJECTED  REJECTED  TALENT_POOL   REJECTED         DECLINED → (next candidate)
```

---

## 4. API Design

### 4.1 API Architecture Principles

- **RESTful** with resource-oriented URLs
- **JSON** request and response bodies throughout
- **Versioned** under `/api/v1/`
- **Auth**: Bearer JWT in Authorization header (internal) or httpOnly cookie (browser SPA)
- **Pagination**: cursor-based for lists (`?cursor=&limit=`)
- **Error format**: `{ error: { code, message, details? } }`
- **Audit**: all mutating endpoints log to audit trail automatically via middleware

### 4.2 Key API Endpoints

```
# Authentication
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh

# RFR Management
POST   /api/v1/rfrs                          # Create RFR (LM)
GET    /api/v1/rfrs                          # List RFRs (filtered by role)
GET    /api/v1/rfrs/:id                      # Get RFR detail
PUT    /api/v1/rfrs/:id                      # Update RFR (only in DRAFT or REVISION status)
POST   /api/v1/rfrs/:id/submit               # Submit for approval
POST   /api/v1/rfrs/:id/approve              # Approve current step (role-gated)
POST   /api/v1/rfrs/:id/reject               # Reject with notes (role-gated)

# Job Requisitions
POST   /api/v1/job-requisitions              # Create JR (Recruiter)
GET    /api/v1/job-requisitions/:id          # Get JR detail
PUT    /api/v1/job-requisitions/:id          # Update JR
POST   /api/v1/job-requisitions/:id/publish  # Mark as published
POST   /api/v1/job-requisitions/:id/sign-off # LM/SME sign-off on JD

# Candidates
POST   /api/v1/applications                  # Public endpoint — candidate applies
GET    /api/v1/applications                  # List by JR (Recruiter/LM/SME)
GET    /api/v1/applications/:id              # Candidate detail
PATCH  /api/v1/applications/:id/status       # Advance/reject candidate
POST   /api/v1/applications/:id/screen       # Submit screening evaluation

# Shortlist
POST   /api/v1/applications/:id/shortlist-vote  # LM/SME submit shortlist vote

# Interviews
POST   /api/v1/interviews                    # Schedule interview round
GET    /api/v1/interviews/:id                # Interview detail
POST   /api/v1/interviews/:id/scorecard      # Submit evaluation scorecard
GET    /api/v1/interviews/:id/scorecard      # Get scorecard (role-gated)

# Offers
POST   /api/v1/offers                        # Create offer (Recruiter)
GET    /api/v1/offers/:id                    # Offer detail
POST   /api/v1/offers/:id/approve            # Finance/Legal approval
POST   /api/v1/offers/:id/send               # Send to candidate
POST   /api/v1/offers/:id/respond            # Candidate accept/decline/counter

# Onboarding
GET    /api/v1/onboarding                    # List onboarding boards
GET    /api/v1/onboarding/:id                # Board detail
PATCH  /api/v1/onboarding/:id/tasks/:taskId  # Mark task complete

# KPI Dashboard
GET    /api/v1/kpi/summary                   # All 10 KPIs current values + RAG
GET    /api/v1/kpi/:kpiId/trend              # 12-month trend for single KPI
GET    /api/v1/kpi/:kpiId/breakdown          # Drill-down by seniority/channel/function

# Talent Pool
GET    /api/v1/talent-pool                   # Search internal talent pool
POST   /api/v1/talent-pool                   # Add profile to pool
DELETE /api/v1/talent-pool/:id               # Remove (GDPR)

# Referrals
POST   /api/v1/referrals                     # Submit referral (authenticated employee)
GET    /api/v1/referrals                     # List referrals (Recruiter)
PATCH  /api/v1/referrals/:id/incentive       # Mark incentive as paid

# Admin
GET    /api/v1/admin/config                  # Platform configuration
PUT    /api/v1/admin/config                  # Update config
GET    /api/v1/admin/audit-log               # Audit trail (HRBP/Director only)
```

---

## 5. Workflow Engine Design

### 5.1 Approach

The workflow engine is implemented as a **state machine service** within the API layer. Each workflow entity (RFR, CandidateApplication, Offer, Onboarding) has a defined set of valid states and transitions. Transitions are triggered by explicit API calls and validated by the engine before execution.

```typescript
// Simplified workflow transition example
interface WorkflowTransition {
  from: string | string[];   // Valid source states
  to: string;                // Target state
  guard: (actor: User, entity: any) => boolean;  // Role/condition check
  effects: Effect[];         // Side effects: notifications, SLA creation, etc.
}

// Example: RFR approval step
const approveRFR_Finance: WorkflowTransition = {
  from: 'PENDING_FINANCE',
  to: 'PENDING_LEGAL',       // or PENDING_DIRECTION if Legal not required
  guard: (actor) => actor.role === 'FINANCE',
  effects: [
    createSLATimer({ target: 'HRBP', durationHours: 8, escalateAfterHours: 24 }),
    sendNotification({ template: 'rfr_next_approver', to: 'LEGAL_TEAM' })
  ]
};
```

### 5.2 SLA Timer System

SLA timers are implemented as **BullMQ delayed jobs** backed by Redis:

```
[Approval Step Created]
       ↓
[BullMQ: schedule job at T + SLA_HOURS]
       ↓ (at T + SLA_HOURS * 0.5)
[Reminder notification dispatched]
       ↓ (at T + SLA_HOURS)
[SLA check: if step still pending → escalation notification]
       ↓ (at T + SLA_HOURS * 1.5)
[Critical escalation: notify superior]
```

---

## 6. UI/UX Design Principles

### 6.1 Design System

The UI is built on a **Tailwind CSS design system** with consistent tokens:

| Token | Value | Usage |
|-------|-------|-------|
| `primary` | `#1F3864` (deep blue) | Primary actions, nav, headers |
| `success` | `#1F6B21` (green) | Approved states, positive KPIs |
| `warning` | `#F59E0B` (amber) | Yellow SLA alerts, "approaching threshold" |
| `danger` | `#DC2626` (red) | Critical alerts, rejected states, KPI red |
| `neutral` | `#6B7280` (gray) | Secondary text, borders, disabled |

RAG status colors follow the same palette. The color scheme aligns with the Mermaid diagram colors defined in the AS-IS document for visual continuity.

### 6.2 Key UI Patterns

**Role-Based Navigation**: Each user role sees a tailored sidebar. HRBP sees all modules; LM sees only "My RFRs", "Shortlist Reviews", and "My Interview Scorecards"; Candidate sees only the application portal.

**Workflow Progress Indicator**: Every entity (RFR, Application, Offer) has a horizontal stepper component showing current position in the workflow with timestamps for completed steps.

**KPI Card Pattern**:
```
┌─────────────────────────────────┐
│  KPI-07  Time to Feedback       │
│                                 │
│       1.4 days         🟢       │
│  ─────────────────────          │
│  Target: ≤2 days                │
│  vs last month: ↓ 0.3d (better) │
│  [View Trend →]                 │
└─────────────────────────────────┘
```

**Evaluation Scorecard UI**:
```
┌─────────────────────────────────────────────┐
│  Round 2 — Technical Interview              │
│  Candidate: Marco R. | SME: Luca B.         │
├─────────────────────────────────────────────┤
│  HARD SKILLS (weight: 50%)                  │
│  System Design         ○ ○ ● ○ ○  3/5      │
│  Code Quality          ○ ○ ○ ● ○  4/5      │
│  Problem Solving       ○ ○ ● ○ ○  3/5      │
├─────────────────────────────────────────────┤
│  Composite Score: [auto-calculated: 3.4/5]  │
│  Notes: [                              ]    │
│  [Save Draft]          [Submit Scorecard]   │
│  ⚠️ Deadline: 14:00 today (4h remaining)    │
└─────────────────────────────────────────────┘
```

### 6.3 Page Structure

| Page | Path | Primary Actor |
|------|------|---------------|
| Dashboard / Home | `/` | HRBP, Recruiter |
| KPI Dashboard | `/analytics` | HRBP, Recruiter, Director |
| RFR List | `/rfrs` | All internal |
| RFR Detail + Approval | `/rfrs/:id` | All internal |
| Open Positions | `/positions` | All internal |
| Position Detail | `/positions/:id` | Recruiter, LM, SME |
| Candidate Pipeline | `/positions/:id/candidates` | Recruiter, LM, SME |
| Candidate Profile | `/candidates/:id` | Recruiter, LM, SME |
| Scorecard Submission | `/interviews/:id/scorecard` | Interviewer |
| Offer Detail | `/offers/:id` | Recruiter, Finance, Legal |
| Onboarding Board | `/onboarding/:id` | IT, Facility, Payroll, HR, LM |
| Talent Pool | `/talent-pool` | HRBP, Recruiter |
| Referral Portal | `/referrals` | All employees |
| Candidate Application | `/apply/:jrSlug` | Candidate (public) |
| Offer Acceptance | `/offer/:token` | Candidate (public, token-protected) |
| Admin / Config | `/admin` | HRBP |

---

## 7. Notification System Design

### 7.1 Notification Architecture

Notifications flow through a dedicated **Notification Dispatcher** module backed by BullMQ:

```
[System Event] → [Notification Dispatcher]
                         ↓
                 [Template Resolver]
                 (selects template by event type + locale)
                         ↓
                 [Channel Router]
                 ┌────────┴────────┐
          [In-App Notif]    [Email Dispatch]
          (stored in DB)    (Nodemailer + MJML)
```

### 7.2 Notification Event Catalog

| Event | Recipients | Channel | SLA Impact |
|-------|-----------|---------|------------|
| RFR submitted | HRBP | Email + In-App | Starts KPI-02 timer |
| RFR approval step completed | Next approver | Email + In-App | — |
| RFR approved | LM, Recruiter | Email + In-App | — |
| RFR rejected | LM | Email + In-App | — |
| Internal search period ending (T-1 day) | HRBP, Recruiter | In-App | — |
| JR published | LM (FYI) | In-App | Starts KPI-01 timer |
| New application received | Recruiter | In-App (batched hourly) | — |
| Referral application received | Recruiter | Email + In-App | Starts 48h SLA |
| Shortlist package ready | LM, SME | Email + In-App | Starts 48h SLA |
| Interview scheduled | Candidate, Interviewers | Email | — |
| Scorecard SLA reminder (T+12h) | Interviewer | Email + In-App | — |
| Scorecard SLA breach (T+24h) | Interviewer + HRBP | Email + In-App | KPI-07 breach |
| Offer sent to candidate | Candidate | Email | Starts KPI-08 timer |
| Offer accepted | Recruiter, HR, HRBP, LM | Email + In-App | Triggers onboarding |
| Offer declined | Recruiter | Email + In-App | — |
| Onboarding T-5 alert | IT, Facility, Payroll, HR, LM | Email + In-App | KPI-10 at risk |
| Onboarding T-2 alert (incomplete) | HRBP | Email + In-App | KPI-10 breach |
| KPI threshold breach | KPI owner (role-specific) | Email + In-App | — |
| Referral milestone updates | Referrer employee | Email | — |
| GDPR retention expiry | Candidate | Email | — |

---

## 8. Integration Architecture

### 8.1 ATS Integration (v1: Read-Only)

The system consumes candidate data from Teamtailor via its REST API in read-only mode. A nightly sync job imports new candidates and updates existing ones. The integration is implemented as an **adapter pattern** to ensure the internal system is decoupled from ATS-specific data structures.

```typescript
interface ATSAdapter {
  fetchCandidates(since: Date): Promise<CandidateDTO[]>;
  fetchApplications(jobId: string): Promise<ApplicationDTO[]>;
}

class TeamtailorAdapter implements ATSAdapter { ... }
class NullATSAdapter implements ATSAdapter { ... }  // Used when ATS not configured
```

### 8.2 Email Integration

Nodemailer is configured to use SMTP (configurable via environment variables). MJML templates produce responsive HTML emails. Template variables are validated at compile time via TypeScript generics.

### 8.3 Calendar Integration (v2)

Planned for v2: bidirectional Google Calendar / Microsoft 365 Calendar sync for interview scheduling. In v1, interview slots are entered manually and iCal (.ics) files are attached to invitation emails.

### 8.4 Digital Signature

Offer letters are generated as PDFs via Puppeteer (HTML template → headless Chrome → PDF). For v1, e-signature is implemented as a **timestamped checkbox consent** captured on the candidate portal, with IP address and timestamp stored. A DocuSign or equivalent integration is planned for v2 where legally binding signatures are required.

---

## 9. Security Design

### 9.1 Authentication and Authorization

- **JWT Access Tokens**: 15-minute expiry, signed with HS256, contain `userId`, `role`, `businessUnitId`
- **Refresh Tokens**: 7-day expiry, stored as httpOnly cookie, rotated on each refresh
- **RBAC enforcement**: Every protected route validates the JWT role claim against a permission matrix before handler execution
- **Rate limiting**: `express-rate-limit` on all auth endpoints (10 req/min per IP)

### 9.2 Data Security

- All PII stored encrypted at rest in PostgreSQL (pgcrypto for sensitive fields: salary data, candidate personal data)
- CV files stored with server-side encryption in S3
- All internal-to-DB connections use SSL
- Audit log is stored in an append-only table with no UPDATE/DELETE permissions granted to application user

### 9.3 Public Portal Security

- Candidate application portal is rate-limited (5 applications per IP per hour)
- File uploads validated for MIME type (CV: PDF/DOCX only) and scanned via ClamAV before storage
- Offer acceptance pages use time-limited, one-use tokens (UUID + expiry in Redis)
- All public endpoints protected by CSRF tokens

### 9.4 GDPR Technical Controls

- Data minimization: only fields required for the hiring process are collected
- Right to erasure: admin endpoint to anonymize candidate records (replaces PII with hashed tokens)
- Data portability: candidates can request a JSON export of their data via the portal
- Retention automation: BullMQ jobs run nightly to check retention expiry dates and trigger the notification/anonymization workflow

---

## 10. KPI Calculation Logic

Each KPI is calculated from live data in the system. Below are the core formulas:

| KPI | Calculation |
|-----|-------------|
| KPI-01 Time to Fill | `DATEDIFF(offer.acceptedAt, rfr.approvedAt)` averaged over closed positions in period |
| KPI-02 Approval Cycle Time | `DATEDIFF(rfr.approvedAt, rfr.submittedAt)` in working days |
| KPI-03 Internal Sourcing Rate | `COUNT(applications WHERE source=INTERNAL AND status=HIRED) / COUNT(hires)` |
| KPI-04 Referral Conversion Rate | `COUNT(referrals WHERE status=HIRED) / COUNT(referrals_total)` |
| KPI-05 Screening-to-Interview Rate | `COUNT(applications WHERE status IN [SHORTLISTED, INTERVIEWING, ...]) / COUNT(applications_screened)` |
| KPI-06 Interview-to-Offer Rate | `COUNT(offers_sent) / COUNT(candidates_interviewed)` |
| KPI-07 Time to Feedback | `AVG(DATEDIFF(scorecard.submittedAt, interview.completedAt))` by LM/SME |
| KPI-08 Offer Acceptance Rate | `COUNT(offers_accepted) / COUNT(offers_sent)` |
| KPI-09 Quality of Hire | Composite score (to be configured with performance management system; placeholder in v1) |
| KPI-10 Onboarding Readiness | `COUNT(tasks_completed_by_day1) / COUNT(total_tasks)` per onboarding board |

RAG thresholds per KPI are stored in the system configuration table and are fully configurable by the HRBP administrator.

---

*Document owner: Senior Business Analyst — HR Technology Program*
*Version: 1.0 | Status: Draft for Review*
