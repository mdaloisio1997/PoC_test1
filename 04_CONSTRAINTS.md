# Constraints — Hiring Management Web Application

## Purpose of This Document

This document catalogs all **constraints** that bound the design, development, and operation of the Hiring Management Web Application. Constraints are non-negotiable conditions that the solution must comply with — they are not requirements to implement, but boundaries within which the implementation must operate.

Constraints are organized into six categories: Technical, Business Process, Regulatory/Legal, Organizational, Operational, and Performance. For each constraint, the document specifies its origin, impact on design decisions, and the relevant mitigation or compliance measure.

---

## CATEGORY 1 — Technical Constraints

### TC-01 — Runtime: Node.js (Mandatory)

**Statement**: The backend application must be implemented using Node.js. No other server-side runtime (Python, Java, .NET, Go, Ruby) is permitted.

**Origin**: Client technology mandate.

**Impact on Design**:
- All backend logic, workflow engine, SLA scheduler, and API layer must be written in Node.js (TypeScript).
- Third-party libraries and integrations must have Node.js-compatible packages. No library requiring a native runtime other than Node.js can be used as a primary dependency.
- The frontend choice is unconstrained by this mandate (React/Vue/Angular are all compatible), but must consume the Node.js API.

**Compliance Measure**: Technology stack locked to Node.js 20 LTS + Express.js + TypeScript. Stack documented in `03_DESIGN.md`. Any proposed deviation from Node.js requires formal change request.

---

### TC-02 — Web Application (No Native Mobile App in v1)

**Statement**: The application must be delivered as a web application accessible from any modern browser. Native iOS or Android applications are out of scope for v1.

**Origin**: Scope decision (cost and timeline constraint); native mobile planned for v2.

**Impact on Design**:
- The frontend must be fully responsive (mobile-first CSS) to support Line Managers and SMEs who may review scorecards from mobile browsers.
- The Candidate Application Portal must be usable on mobile browsers without degradation.
- Push notifications are not available (web notifications via browser API only, with limited reach); email remains the primary notification channel.

**Compliance Measure**: All UI components built with Tailwind CSS responsive utilities. Mobile viewport testing included in QA acceptance criteria. No React Native or Flutter dependencies introduced.

---

### TC-03 — No Full ATS Replacement in v1

**Statement**: The existing ATS (Teamtailor) must not be decommissioned in v1. The new application must coexist with the ATS, consuming data from it via read-only API integration.

**Origin**: Business continuity risk management. The ATS contains historical data, active pipelines, and institutional knowledge built up over years. A big-bang replacement would risk data loss and operational disruption.

**Impact on Design**:
- The application's internal data model must be able to represent ATS data imported via nightly sync without losing information.
- The ATS Adapter (see `03_DESIGN.md`, Section 8.1) must be designed as a replaceable interface so that bidirectional sync or full ATS migration can be added in v2 without redesigning the core data model.
- Users should be clearly informed (via UI) whether a candidate record originated from the ATS or was created natively in the new system.
- Duplicate candidate detection must be implemented at the ATS sync boundary to prevent ghost records.

**Compliance Measure**: ATSAdapter interface defined and implemented as TeamtailorAdapter. All ATS-synced records flagged with `source: 'ATS_SYNC'` and `externalId` field. Reconciliation report generated after each nightly sync.

---

### TC-04 — Self-Hosted Deployment Capability

**Statement**: The application must be deployable on the client's own infrastructure (on-premise or private cloud). It must not have hard dependencies on any specific cloud provider's managed services (no AWS Lambda, no Azure-specific APIs, no GCP Cloud Run).

**Origin**: Client data sovereignty requirements and IT infrastructure policy.

**Impact on Design**:
- S3-compatible object storage must be supported through an abstracted storage adapter (supporting both AWS S3 and on-premise MinIO).
- Redis must be deployable as a self-hosted instance, not as a managed service.
- PostgreSQL must be deployable as a self-hosted instance.
- Docker + docker-compose must be sufficient for a full local and production deployment without requiring cloud-provider tooling.

**Compliance Measure**: All external service dependencies abstracted behind interface adapters. `docker-compose.yml` ships with the repository and covers all dependencies. Environment variables documented in `.env.example`. No AWS SDK calls that are not wrapped behind the storage adapter.

---

### TC-05 — No AI/ML-Based Candidate Ranking in v1

**Statement**: The application must not include automated AI/ML-based candidate scoring, ranking, or screening decision systems in v1.

**Origin**: Regulatory risk (EU AI Act, Article 6 — high-risk AI systems in employment context). Explainability requirement for hiring decisions under GDPR recital 71. Internal HR policy requiring human decision authority in all candidate selection steps.

**Impact on Design**:
- The ATS keyword screening (current AS-IS) is deliberately NOT replicated in the new system. Screening must be human-led (structured checklist, CAP-5.1).
- Weighted composite scoring in evaluation scorecards is NOT an AI system: it is a deterministic formula applied to human-entered scores. This is permissible, as the formula is transparent and the inputs are human-provided.
- No integration with AI recruiting tools (HireVue, Pymetrics, etc.) in v1.
- A placeholder architecture note must be included in the codebase indicating where an AI scoring module could be added in v3 with appropriate AI Act compliance measures.

**Compliance Measure**: Architecture Decision Record (ADR-001) documenting the AI exclusion rationale filed in the `/docs/adr/` directory. Peer review gate added to CI pipeline that flags any `import from 'openai'` or similar AI SDK imports for manual approval.

---

### TC-06 — Single-Tenant Architecture in v1

**Statement**: The application is designed for a single company (single tenant). Multi-tenancy (serving multiple companies from a single instance) is out of scope for v1.

**Origin**: Scope and timeline constraint. Multi-tenancy requires significant additional complexity in data isolation, billing, and tenant-specific configuration.

**Impact on Design**:
- No `tenantId` partitioning in the database schema.
- Configuration (KPI thresholds, SLA durations, roles) is global, not per-tenant.
- If multi-tenancy is required in future, a database-per-tenant strategy is preferred over row-level tenant isolation (simpler migration path from v1).

**Compliance Measure**: ADR-002 documents the single-tenant decision and the recommended multi-tenancy migration path. No tenant-related abstractions added prematurely.

---

### TC-07 — Browser Compatibility

**Statement**: The application must function correctly on the current and previous major version of: Chrome, Firefox, Safari, and Edge. Internet Explorer is explicitly excluded.

**Origin**: Standard modern web compatibility requirement.

**Impact on Design**:
- CSS Grid and Flexbox are the primary layout tools (all supported in target browsers).
- No `import()` with Safari <15 quirks (all target versions support dynamic import).
- Date handling must use `date-fns` or equivalent library rather than relying on inconsistent native `Date` parsing across browsers.

**Compliance Measure**: Vite targets set to `["chrome>=110", "firefox>=110", "safari>=16", "edge>=110"]` in `vite.config.ts`. BrowserStack or Playwright cross-browser tests included in the QA suite.

---

## CATEGORY 2 — Business Process Constraints

### BP-01 — Internal Search Must Precede External Publication

**Statement**: No Job Requisition may be published externally (Career Page, LinkedIn, Indeed, InfoJobs) until the internal talent pool search and mobility assessment phase has been completed or formally closed by an authorized user (HRBP or Recruiter with mandatory reason code).

**Origin**: AS-IS process rule (Section 1.3 of AS-IS document). This is an HR policy constraint, not a legal one, but it is considered mandatory by the HR function.

**Impact on Design**:
- The "Publish" action on a Job Requisition must be blocked by a system gate that checks whether the internal search window has been formally closed.
- The internal search closure must be logged with timestamp and reason code (stored in audit trail).
- Circumventing this gate must require HRBP-level authorization and a logged override reason.

**Compliance Measure**: Server-side guard on `POST /api/v1/job-requisitions/:id/publish` that checks `internalSearchStatus === 'CLOSED'`. UI "Publish" button is disabled and displays a tooltip explaining the prerequisite.

---

### BP-02 — Three-Round Interview Structure is the Standard

**Statement**: The standard interview process consists of three sequential rounds: HR Motivational (Round 1), Technical (Round 2), and Client Final (Round 3 — conditional for commessa roles). The system must support this structure by default.

**Origin**: AS-IS process design. The three-round structure is a deliberate HR policy reflecting the quality standard for candidate evaluation.

**Impact on Design**:
- Interview rounds are created as an ordered sequence. Round 2 cannot be scheduled until Round 1 scorecard is submitted. Round 3 cannot be scheduled until Round 2 scorecard is submitted.
- Exceptions (e.g., compressing Round 1 and Round 2 for urgent roles) require HRBP authorization and are logged.
- The system does not enforce that all three rounds must occur for all roles (Round 3 is conditional), but it enforces the sequential dependency between rounds that are activated.

**Compliance Measure**: State machine guard on interview round creation. `canScheduleRound(roundNumber, application)` function checks prerequisite completion before allowing scheduling.

---

### BP-03 — Candidate Response Window is 3 Calendar Days (Default)

**Statement**: The standard offer response window for candidates is 3 calendar days from the moment the offer is delivered. This window is configurable per offer (HRBP/Recruiter can extend) but cannot be set below 1 day.

**Origin**: AS-IS process (Section 1.7). Legal/HR policy constraint to balance candidate decision time with operational tempo.

**Impact on Design**:
- Offer expiry timer must be implemented accurately. Expiry is evaluated in calendar days (not working days), from the exact timestamp the offer email was delivered, not when it was sent.
- The candidate portal must display a countdown to expiry.
- Upon expiry, the offer is automatically set to `EXPIRED` status and the Recruiter is notified. The HRBP can manually re-open an expired offer with a logged justification.

**Compliance Measure**: BullMQ delayed job created at offer dispatch. Expiry time stored as UTC timestamp in `offer.expiresAt`. Client-side countdown computed from server-provided timestamp to avoid timezone discrepancies.

---

### BP-04 — Weighted Scoring Formula is Fixed (Cannot Be Modified by Users)

**Statement**: The composite evaluation score formula — Hard Skill 50%, Soft Skill 25%, Cultural Fit 15%, Motivation/Availability 10% — is a fixed HR policy. Individual users cannot modify the weightings per role or per requisition.

**Origin**: AS-IS process (Section 1.7). The HR function has deliberately standardized these weights to ensure cross-candidate comparability. Allowing per-role modification would undermine the standardization objective.

**Impact on Design**:
- The composite score formula is hardcoded in the scoring module, not stored as a configurable parameter.
- The formula is surfaced transparently in the UI so evaluators understand how their scores are weighted.
- A future version (v3+) could introduce configurable weights with appropriate access controls, documented as an out-of-scope enhancement.

**Compliance Measure**: Formula defined as a constant in a dedicated `scoring.service.ts` module. Any proposed change to weights requires a formal process change request, not a configuration change.

---

### BP-05 — Referral Incentive Vesting Period Must Be Enforced Systematically

**Statement**: Referral incentives (€300–€1,000) are only disbursable after the referred employee has been in employment for the vesting period (3–6 months). The system must enforce this timing and prevent premature incentive marking.

**Origin**: AS-IS process (Section 1.4). HR/Finance policy to align incentive with retention outcome.

**Impact on Design**:
- The "Mark Incentive as Paid" action on a referral record is locked until `hire.startDate + vestingPeriodDays >= today`.
- Vesting period is configurable globally (default: 90 days) and can be overridden per referral by HRBP.
- A BullMQ job notifies HR at vesting date: "Referral incentive for [employee] is now eligible for payment."

**Compliance Measure**: Server-side guard on incentive payment endpoint. UI button is disabled with countdown display until vesting date.

---

### BP-06 — Approval Rejection Requires Written Justification

**Statement**: Any rejection at any step in any approval chain (RFR, JD sign-off, Shortlist, Offer) must be accompanied by a mandatory written justification note. Rejections without notes are not accepted by the system.

**Origin**: Audit trail requirement and HR process governance. Rejection notes are the primary mechanism for LMs to understand why a process step was rejected and what corrective action is required.

**Impact on Design**:
- All rejection API endpoints must validate that `notes` field is non-empty (minimum 20 characters).
- Rejection notes are stored in the audit trail and surfaced in the entity's history view.
- The UI must prevent form submission with an empty rejection note and display a clear validation message.

**Compliance Measure**: Zod validation schema for all rejection request bodies requires `notes: z.string().min(20)`. Backend validation redundantly enforces this even if client validation is bypassed.

---

## CATEGORY 3 — Regulatory and Legal Constraints

### RL-01 — GDPR Compliance (EU Regulation 2016/679)

**Statement**: The application processes personal data of EU residents (candidates) and must comply with GDPR in all aspects of data collection, storage, processing, retention, and deletion.

**Origin**: EU Regulation 2016/679 (GDPR). Mandatory for all organizations processing EU personal data.

**Specific Requirements**:

| Requirement | Implementation |
|-------------|---------------|
| Lawful basis for processing | Consent collected at application submission with explicit checkbox and versioned consent text |
| Data minimization | Only data necessary for hiring process is collected; no unnecessary PII fields |
| Purpose limitation | Candidate data used only for the specific recruitment purpose it was collected for |
| Retention limits | 12 months (rejected after screening), 24 months (interviewed), 36 months (offer received) — configurable, enforced by BullMQ nightly job |
| Right of access | Candidates can request their data export via email/portal; admin endpoint to generate JSON export |
| Right to erasure | Admin endpoint to anonymize candidate records (PII replaced with hashed tokens) |
| Right to rectification | Candidates can update their profile via the portal during active application |
| Data breach notification | Incident response process documented in operational runbook; system logs all data access |
| Data Processing Agreement | Legal template provided for client to sign with any sub-processors (email provider, cloud storage) |

**Impact on Design**:
- Consent and timestamp stored immutably per candidate record
- Audit log records all data access and mutations by system users
- Anonymization function replaces: name, email, phone, address, CV file with salted hashes; interview scores and process outcomes are retained (anonymized) for HR analytics

---

### RL-02 — EU AI Act Compliance (Regulation 2024/1689) — Preventive Constraint

**Statement**: The EU AI Act classifies recruitment, selection, and assessment AI systems as **high-risk** (Annex III, Category 4). Any automated system making or significantly influencing employment decisions must meet strict transparency, human oversight, and accuracy requirements before deployment.

**Origin**: EU AI Act, entered into force August 2024. Full application of high-risk AI provisions: August 2026.

**Impact on Design**:
- AI/ML-based candidate scoring is excluded from v1 (see TC-05).
- The deterministic weighted scoring formula (BP-04) is NOT an AI system under the Act's definition (no machine learning, no probabilistic output). It is fully explainable and human-controlled.
- If AI features are added in future versions, a Fundamental Rights Impact Assessment (FRIA) and technical documentation package must be prepared before deployment.

**Compliance Measure**: TC-05 enforces the exclusion. ADR-001 documents the regulatory rationale. A pre-built compliance checklist for future AI feature introduction is included in `/docs/compliance/ai-act-checklist.md`.

---

### RL-03 — Labor Law — Equal Opportunity and Non-Discrimination

**Statement**: The hiring process and its digital implementation must not introduce or amplify discriminatory selection criteria. Data fields and UI elements must not prompt or store information related to protected characteristics (age, gender, ethnicity, disability, religion, sexual orientation, marital status, pregnancy).

**Origin**: Italian Legislative Decree 198/2006 (Codice delle Pari Opportunità), EU Equal Treatment Directive, and general labor law principles.

**Impact on Design**:
- The candidate application form must not include fields for date of birth, gender, nationality, marital status, or photograph. These are prohibited fields.
- CV upload allows candidates to include this information in their document, but the system's structured data model must not extract or store these fields.
- Scorecard criteria must be limited to job-relevant competencies. No "cultural fit" scoring dimension can be used in a way that operationalizes demographic preference (the "cultural fit" criterion in the system refers to organizational values alignment, not demographic similarity, and this must be documented in the HR evaluation guidelines).
- The anonymization feature in the shortlist review (CAP-6.3 in Capabilities document) is designed to reduce bias; while optional in v1, it should be recommended as best practice.

**Compliance Measure**: Application form field list reviewed and approved by Legal before launch. Prohibited fields list maintained in system config (cannot be added by users). HR evaluation guidelines document defines "cultural fit" in operationally neutral terms.

---

### RL-04 — Electronic Signature Legal Validity

**Statement**: The e-signature mechanism used for offer acceptance must meet the minimum legal requirements for contractual validity under applicable Italian and EU law. For employment contracts, a Qualified Electronic Signature (QES) or Advanced Electronic Signature (AdES) may be required in specific contexts.

**Origin**: EU eIDAS Regulation (Regulation 910/2014), Italian Civil Code Art. 1326-1329 (contract formation rules), Italian D.Lgs. 82/2005 (Codice Digitale).

**Impact on Design**:
- For v1, the offer acceptance mechanism captures: candidate email confirmation, IP address, timestamp, and explicit checkbox consent. This constitutes a **Simple Electronic Signature (SES)** under eIDAS, sufficient for most employment offer acceptances (non-derogable rights contracts excluded).
- The system must document clearly in the offer that the acceptance via portal constitutes a binding agreement.
- If the Legal team determines that specific role types require AdES or QES, a DocuSign/Yousign integration must be planned for v2 and a manual override process must exist in v1 for those cases.

**Compliance Measure**: Legal review of v1 signature mechanism before go-live. Manual override process documented: for roles requiring higher-grade signature, the offer is sent via DocuSign externally and uploaded to the system record. ADR-003 documents the e-signature decision.

---

### RL-05 — Data Localization and Transfer Restrictions

**Statement**: Personal data of EU residents processed by this system must not be transferred to countries outside the EEA without appropriate safeguards (Standard Contractual Clauses, adequacy decision, etc.).

**Origin**: GDPR Chapter V (international transfers).

**Impact on Design**:
- If the application is deployed on infrastructure hosted in non-EEA countries (e.g., US-based cloud providers), appropriate transfer mechanisms must be in place.
- The S3-compatible storage adapter must be configured to use EEA-located buckets by default.
- Email delivery provider must either be EEA-hosted or covered by SCCs.
- The technical design allows for purely EEA deployment (all components self-hosted or on MinIO) as the default recommended configuration.

**Compliance Measure**: Default `docker-compose.yml` uses MinIO for object storage (EEA-deployable). Email provider configured via environment variable with SCC documentation template provided in `/docs/legal/`.

---

## CATEGORY 4 — Organizational Constraints

### OC-01 — Existing Role Structure Must Be Preserved

**Statement**: The application's access model must map exactly to the existing organizational role structure (LM, HRBP, Recruiter, SME, Finance, Legal, Direzione, IT, Facility, Payroll). No new organizational roles should be introduced by the system.

**Origin**: HR organizational design decision. Introducing new roles (e.g., "System Administrator") would require HR/legal re-evaluation of job descriptions and responsibilities.

**Impact on Design**:
- The "HR Analytics Owner" referenced in the TO-BE analysis is NOT a new role: it is an additional responsibility assigned to the existing HRBP role. In the system, this is implemented as a configurable flag (`isAnalyticsOwner: true`) on a User record with HRBP role, not as a separate role.
- System administration tasks are handled by the HRBP role (the most technically sophisticated HR role in the current structure).

**Compliance Measure**: RBAC role list is fixed in code (not configurable by users) and maps 1:1 to the RACI matrix in the AS-IS document.

---

### OC-02 — The RACI Matrix Governs All Access Decisions

**Statement**: Permission decisions for every capability must be traceable to the RACI matrix defined in Section 3 of the AS-IS document. No capability should be accessible by an actor not listed as R, A, C, or I for the corresponding activity.

**Origin**: AS-IS RACI matrix (Section 3 of AS-IS document). Governance constraint.

**Impact on Design**:
- Every protected API endpoint must have a documented permission entry in the permissions matrix (`/docs/permissions-matrix.md`), referencing the RACI row that authorizes access.
- Changes to permissions require review against the RACI matrix. Deviations require sign-off from HRBP.

**Compliance Measure**: Permissions matrix document maintained alongside the codebase. Code review checklist item: "Is the new endpoint permission documented in the permissions matrix?"

---

### OC-03 — HR Function Retains Decision Authority

**Statement**: The system must never make autonomous hiring decisions. All selection, rejection, shortlisting, and offer decisions must be made by human actors and recorded in the system. The system may recommend, alert, and facilitate, but not decide.

**Origin**: HR governance principle. Aligned with EU AI Act (RL-02) and GDPR recital 71 (right not to be subject to solely automated decisions).

**Impact on Design**:
- Automated actions are limited to: notifications, SLA tracking, data routing, status transitions triggered by human actions.
- "Auto-rejection" based on screening question answers (CAP-4.1) is permissible because it is a deterministic rule explicitly configured by HR (not a learned model), and candidates are informed of the auto-rejection reason.
- The system cannot mark a candidate as "Rejected" without a human actor triggering the action (except for the auto-rejection at application screening based on explicit knock-out questions).

**Compliance Measure**: Status transition audit trail records the actor for every state change. `actorId` is a non-nullable field on all state change records.

---

### OC-04 — Final Client Involvement is Strictly Conditional

**Statement**: Client involvement in the hiring process (JD review, final interview round, shortlist co-validation) is only applicable for commessa (client-project) roles. The system must not expose client-facing features for standard internal hiring positions.

**Origin**: AS-IS process design. Client involvement without a commessa relationship would be inappropriate (confidentiality, commercial sensitivity).

**Impact on Design**:
- All Final Client features are hidden unless the Job Requisition has `isCommessa: true` and a `clientContact` email is configured.
- Client users access the system via a special tokenized guest link (no account creation required) with read-only access scoped to the specific JR they are associated with.

**Compliance Measure**: Feature flag `isCommessa` gates all client-facing UI elements and API endpoints. Client guest tokens are single-use or time-limited (24-hour validity).

---

## CATEGORY 5 — Operational Constraints

### OC-OP-01 — System Availability

**Statement**: The application must be available during working hours (Monday–Friday, 07:00–21:00 CET) with 99.5% uptime target. Planned maintenance windows should be scheduled outside these hours.

**Origin**: Operational SLA agreed with client.

**Impact on Design**:
- BullMQ maintenance jobs (GDPR retention checks, KPI recalculation, nightly ATS sync) are scheduled for 02:00–04:00 CET.
- Database backups scheduled for 01:00 CET (daily full backup) and every 6 hours (incremental).
- Graceful shutdown implemented: the Node.js process finishes in-flight requests before terminating during deployments.

---

### OC-OP-02 — Data Backup and Recovery

**Statement**: All operational data must be backed up daily with a Recovery Point Objective (RPO) of 24 hours and a Recovery Time Objective (RTO) of 4 hours.

**Origin**: Client IT governance policy.

**Impact on Design**:
- PostgreSQL: daily pg_dump backups retained for 30 days, stored in S3-compatible storage in a different physical location.
- Redis: RDB persistence enabled (snapshot every 900 seconds or after 10 key changes). AOF optional for lower RPO.
- CV and document files: S3 versioning enabled; deletion is soft-delete only (marked `deletedAt`, not physically removed until GDPR anonymization).

---

### OC-OP-03 — Monitoring and Observability

**Statement**: The application must emit structured logs and metrics sufficient to diagnose performance issues and errors in production without requiring code changes.

**Origin**: Client IT operations requirement.

**Impact on Design**:
- Structured JSON logging via `pino` logger with correlation IDs on all request logs.
- `/health` and `/ready` endpoints for container health checks.
- Prometheus-compatible metrics endpoint (`/metrics`) exposing: request count/latency by route, BullMQ job queue depths, database connection pool utilization.
- Error tracking via Sentry (self-hosted Sentry or Glitchtip is preferred given TC-04).

---

### OC-OP-04 — Environment Parity

**Statement**: Development, staging, and production environments must be functionally equivalent. The application must not have environment-specific code paths that could mask production defects.

**Origin**: Engineering quality constraint.

**Impact on Design**:
- All environment-specific configuration via environment variables (`.env.*` files, never hardcoded).
- The same Docker image is used across all environments; only the environment variables differ.
- Seed scripts for test data are scoped to non-production environments via `NODE_ENV` check.

---

## CATEGORY 6 — Performance Constraints

### PF-01 — API Response Time

**Statement**: 95% of API responses must complete within 500ms under normal load conditions. The KPI dashboard summary endpoint may have a 1,500ms limit given its aggregation complexity.

**Origin**: UX usability threshold. HR users will be navigating between screens during active recruiting sessions.

**Impact on Design**:
- KPI calculations are pre-computed and cached in Redis (recalculated every 15 minutes via BullMQ job), not computed on-demand per request.
- Database queries must use appropriate indexes (defined in Prisma schema). Slow query logging enabled for queries > 200ms.
- N+1 query patterns are prohibited; Prisma `include` with explicit relation loading is used instead of lazy loading.

---

### PF-02 — Concurrent Users

**Statement**: The application must support up to 50 concurrent authenticated users without performance degradation. Peak usage is expected during monthly review meetings and end-of-quarter hiring pushes.

**Origin**: Sizing estimate based on HR team + LM + SME population. Conservative estimate for v1.

**Impact on Design**:
- PostgreSQL connection pool sized at 20 connections (max). BullMQ worker concurrency set to 5 (adjustable).
- Redis connection pool sized at 10 connections.
- Static frontend assets served via Nginx with aggressive cache headers (1 year for hashed bundles).
- Horizontal scaling is possible by adding Node.js replicas behind the Nginx load balancer (session state in Redis, not in-process memory).

---

### PF-03 — File Upload Limits

**Statement**: CV and document uploads are limited to 5MB per file. Total storage per Job Requisition is soft-capped at 500MB (hard limit beyond which the system warns HR).

**Origin**: Infrastructure cost constraint and practical consideration (CVs are never legitimately >5MB).

**Impact on Design**:
- Multer `fileSize` limit set to 5,242,880 bytes (5MB). Oversized uploads are rejected at the API level with a clear error message.
- S3 bucket lifecycle policies archive files older than the GDPR retention period to cold storage automatically.

---

## Constraints Summary Table

| ID | Category | Constraint | Priority | Reviewable in v2? |
|----|----------|-----------|----------|-------------------|
| TC-01 | Technical | Node.js mandatory runtime | Fixed | No |
| TC-02 | Technical | Web app only (no native mobile) | Fixed | Yes |
| TC-03 | Technical | ATS read-only in v1 | Fixed | Yes |
| TC-04 | Technical | Self-hosted deployment capability | Fixed | No |
| TC-05 | Technical | No AI/ML candidate ranking in v1 | Fixed | Yes (v3+, with AI Act compliance) |
| TC-06 | Technical | Single-tenant architecture | Fixed | Yes |
| TC-07 | Technical | Modern browser compatibility | Fixed | No |
| BP-01 | Process | Internal search before external publication | Fixed | No |
| BP-02 | Process | Three-round interview standard | Fixed | Configurable exception |
| BP-03 | Process | 3-day offer response window | Configurable | N/A |
| BP-04 | Process | Fixed scoring formula | Fixed | Yes (v3+) |
| BP-05 | Process | Referral vesting enforcement | Fixed | No |
| BP-06 | Process | Rejection requires written notes | Fixed | No |
| RL-01 | Legal | GDPR compliance | Fixed | No |
| RL-02 | Legal | EU AI Act preventive compliance | Fixed | Evolving |
| RL-03 | Legal | Non-discrimination — prohibited fields | Fixed | No |
| RL-04 | Legal | E-signature legal validity | Fixed | Review at v2 |
| RL-05 | Legal | EEA data localization | Fixed | No |
| OC-01 | Organizational | Existing role structure preserved | Fixed | No |
| OC-02 | Organizational | RACI matrix governs access | Fixed | On RACI update |
| OC-03 | Organizational | Human decision authority | Fixed | No |
| OC-04 | Organizational | Client involvement only for commessa | Fixed | No |
| OC-OP-01 | Operational | 99.5% availability in business hours | Fixed | No |
| OC-OP-02 | Operational | RPO 24h / RTO 4h | Fixed | No |
| OC-OP-03 | Operational | Structured logging + metrics | Fixed | No |
| OC-OP-04 | Operational | Environment parity | Fixed | No |
| PF-01 | Performance | 95th percentile API response ≤500ms | Fixed | No |
| PF-02 | Performance | 50 concurrent users | Fixed | Scale up |
| PF-03 | Performance | 5MB file upload limit | Fixed | Configurable |

---

*Document owner: Senior Business Analyst — HR Technology Program*
*Version: 1.0 | Status: Draft for Review*
