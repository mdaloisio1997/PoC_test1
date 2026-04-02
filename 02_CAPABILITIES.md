# Capabilities — Hiring Management Web Application

## Introduction

This document defines the complete capability inventory of the Hiring Management Web Application. Each capability is described with its business purpose, actors involved, functional behavior, MoSCoW priority, and traceability to the AS-IS pain points and TO-BE improvement opportunities it addresses.

Capabilities are organized into **8 functional domains** that mirror the hiring process phases, plus 2 cross-cutting domains for analytics and platform administration.

---

## Capability Map

```
┌─────────────────────────────────────────────────────────────────────────┐
│ DOMAIN 1: Request & Approval Management                                 │
│ DOMAIN 2: Internal Talent Search                                        │
│ DOMAIN 3: Job Requisition & Publication                                 │
│ DOMAIN 4: Candidate Intake & Referral Management                        │
│ DOMAIN 5: Screening & Shortlist                                         │
│ DOMAIN 6: Interview & Evaluation                                        │
│ DOMAIN 7: Offer Management                                              │
│ DOMAIN 8: Onboarding Activation                                         │
├─────────────────────────────────────────────────────────────────────────┤
│ DOMAIN 9: KPI Dashboard & Analytics (Opportunity I2)                    │
│ DOMAIN 10: Platform Administration & Compliance                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## DOMAIN 1 — Request & Approval Management

### CAP-1.1 — RFR Digital Form with Guided Compilation

**Business Purpose**: Replace unstructured, free-text RFR submissions with a validated digital form that enforces completeness before submission, eliminating the most common source of downstream errors (Error E1 in AS-IS analysis).

**Actors**: Line Manager (creator), HRBP (validator)

**Functional Behavior**:
- The Line Manager accesses a structured form with mandatory fields: role title, seniority level, cost center, authorized RAL range (min/max), required hard skills (multi-select taxonomy), required soft skills, target start date, commessa client reference (conditional), and business justification.
- Fields are validated client-side and server-side. Submission is blocked if mandatory fields are empty or if RAL range is outside pre-configured band for the seniority level.
- The form auto-populates cost center and organizational unit from the LM's profile.
- A preview/PDF summary is shown before submission.

**Priority**: Must Have

**Traces to**: AS-IS Error E1 (incomplete RFR), AS-IS Delay D1 (long approval cycle due to incomplete submissions)

---

### CAP-1.2 — Multi-Level Approval Workflow with SLA Enforcement

**Business Purpose**: Automate the RFR approval chain (HRBP → Finance → Legal → Direzione) with SLA timers, automated reminders, and escalation — replacing the current manual email-based process that has no SLA enforcement.

**Actors**: HRBP, Finance, Legal (conditional), Direzione di Funzione, Recruiter (notified)

**Functional Behavior**:
- Upon RFR submission, the system automatically routes to the HRBP for first-level review (SLA: 1 working day).
- Upon HRBP approval, the system routes to Finance for budget validation (SLA: 1 working day).
- If the role includes special contractual clauses (toggled in the RFR form), the Legal step is activated (SLA: 1 working day).
- Final approval is routed to Direzione di Funzione (SLA: 2 working days).
- Each approver receives an email notification with a direct deep-link to the approval screen.
- If an SLA is breached, an automatic reminder is sent at T+50% of SLA. At T+100% of SLA, the system escalates to the approver's direct superior.
- Rejection at any level returns the RFR to the LM with mandatory rejection notes; the LM can revise and resubmit, triggering only the remaining approval steps (not a full restart).
- All approval decisions are timestamped and audit-logged.

**Priority**: Must Have

**Traces to**: AS-IS Delay D1 (multi-level approval without SLAs), AS-IS Error E1 (cascading errors from unreviewed RFRs), KPI-02 (Approval Cycle Time ≤5 working days)

---

### CAP-1.3 — RFR Dashboard and Tracking

**Business Purpose**: Give HR and Line Managers real-time visibility on the status of open RFRs, replacing the current zero-visibility state where LMs have no insight into where their request is in the approval chain.

**Actors**: Line Manager, HRBP, Recruiter, Direzione di Funzione

**Functional Behavior**:
- A dedicated view lists all RFRs with status (Draft, Pending HRBP, Pending Finance, Pending Legal, Pending Direzione, Approved, Rejected, Closed).
- Each RFR row shows time elapsed since submission, current approval step, and a color-coded SLA indicator (green / yellow / red).
- Line Managers see only their own RFRs. HRBP and Recruiter see all RFRs within their business unit. Direzione sees all.
- Clicking an RFR opens its full detail view including approval history, rejection notes, and linked Job Requisition (if one exists).

**Priority**: Must Have

**Traces to**: AS-IS Delay D1, AS-IS transparency gap

---

## DOMAIN 2 — Internal Talent Search

### CAP-2.1 — Internal Talent Pool Search

**Business Purpose**: Systematize the pre-sourcing phase by providing a searchable internal talent pool that aggregates former interns, unallocated resources, candidates from past selections, and university partnership profiles — enforcing the mandatory 2–5 day internal search before any external publication.

**Actors**: HRBP, Recruiter

**Functional Behavior**:
- Upon RFR approval, the system automatically opens a 3-day Internal Search window (configurable: 2–5 days). External publication is locked during this window.
- The HRBP and Recruiter can query the internal talent pool using skill tags, seniority, and availability filters.
- Profiles in the pool are tagged by category: Ex-Intern, Unallocated Resource, Past Candidate (with previous evaluation scores visible), University Partnership.
- Search results display a match score based on RFR requirements vs. profile skills.
- For each internal candidate, the system allows initiating a Fast Track Internal Interview (HR + LM) directly from the search results.
- If no internal candidate is found or all internal candidates are assessed as unfit, the Recruiter closes the internal search with a mandatory reason code, unlocking external publication.

**Priority**: Must Have

**Traces to**: AS-IS Phase 3 (mandatory internal sourcing), KPI-03 (Internal Sourcing Rate ≥20%)

---

### CAP-2.2 — Internal Mobility Assessment

**Business Purpose**: Facilitate joint HRBP + Line Manager evaluation of internal employees for redeployment, currently managed informally via email and verbal communication.

**Actors**: HRBP, Line Manager

**Functional Behavior**:
- During the internal search window, HRBP can tag existing employees as "Mobility Candidates" and share them with the relevant Line Manager via the platform.
- The LM can review the employee profile and provide a fit/no-fit assessment with notes.
- If mutual agreement is reached, the RFR is marked as "Closed — Internal" and onboarding tasks are triggered (Capability CAP-8.x).

**Priority**: Should Have

**Traces to**: AS-IS Phase 3 (mobilità interna), KPI-03

---

## DOMAIN 3 — Job Requisition & Publication

### CAP-3.1 — Job Requisition Creation with Structured JD Builder

**Business Purpose**: Standardize Job Description creation to eliminate the mismatch between JD and actual role requirements (AS-IS Error E2), ensuring LM, SME, and Recruiter all contribute to and sign off on the JD before publication.

**Actors**: Recruiter (owner), SME (contributor), Line Manager (reviewer), Final Client (conditional reviewer for commessa roles)

**Functional Behavior**:
- The Recruiter creates a Job Requisition linked to the approved RFR. Required fields auto-populate from the RFR (seniority, RAL range, cost center, skills).
- The JD builder includes structured sections: Role Summary, Key Responsibilities, Must-Have Requirements, Nice-to-Have Requirements, Compensation Range, Work Mode (onsite/hybrid/remote).
- The Recruiter can invite the SME to co-edit the technical requirements section. Edit history is tracked.
- Before publication, LM and SME must provide digital sign-off on the JD. For commessa roles, the Final Client must also approve.
- ATS screening questions are configured at this stage (linked to must-have requirements).

**Priority**: Must Have

**Traces to**: AS-IS Error E2 (JD misalignment), AS-IS Delay D3 (multi-channel publication delays)

---

### CAP-3.2 — Multi-Channel Publication Tracker

**Business Purpose**: Track publication status across all job boards (Career Page, LinkedIn, Indeed, InfoJobs) and agencies in a single interface, replacing manual tracking across platforms.

**Actors**: Recruiter

**Functional Behavior**:
- For each Job Requisition, the Recruiter can log publication entries per channel with date, status (Active/Paused/Expired), and source URL.
- The system does not auto-publish (out of scope v1) but provides a checklist-driven workflow to ensure no channel is missed.
- For agency activations, the Recruiter logs agency name, contact, and activation date.
- Publication metrics feed into the KPI dashboard sourcing channel analysis (CAP-9.x).

**Priority**: Should Have

**Traces to**: AS-IS Phase 4 (multi-channel publication), KPI-04 (sourcing channel effectiveness)

---

### CAP-3.3 — Escalation and Sourcing Re-activation

**Business Purpose**: Automate the escalation logic currently managed manually: when time-to-fill thresholds are breached, the system triggers pre-defined corrective actions rather than waiting for HR to notice.

**Actors**: HRBP (notified + action owner), Recruiter, Line Manager (notified)

**Functional Behavior**:
- The system monitors time elapsed since JR opening per seniority tier: 30 days (Junior/Mid), 45–60 days (Specialist), 90 days (Senior/Executive).
- At threshold breach, the system generates an Escalation Alert including: current candidate count, conversion funnel by stage, and a recommended action menu (expand RAL range, revise JD requirements, activate agency, enable remote/international candidates).
- The HRBP must select and log a corrective action within 2 working days of receiving the alert.
- For commessa roles, the Final Client is automatically notified with a templated update message.

**Priority**: Must Have

**Traces to**: AS-IS Delay D9 (late agency activation), AS-IS Escalation mechanisms (Section 1.9), KPI-01

---

## DOMAIN 4 — Candidate Intake & Referral Management

### CAP-4.1 — Candidate Application Portal

**Business Purpose**: Provide a branded, mobile-responsive candidate portal for direct applications, ensuring candidates have a quality experience regardless of the sourcing channel.

**Actors**: Candidate (external)

**Functional Behavior**:
- Each active Job Requisition has a publicly accessible application page with full JD, structured application form, CV upload (PDF/DOCX, max 5MB), and optional cover letter.
- The form includes the screening questions configured in CAP-3.1. Responses that fail mandatory screening questions show a polite auto-rejection with reason.
- Candidates receive an automatic confirmation email upon submission with application reference number.
- GDPR consent is collected at submission. Consent and timestamp are stored in the candidate record.

**Priority**: Must Have

**Traces to**: AS-IS candidate experience, AS-IS Error E10 (no feedback to rejected candidates), GDPR compliance

---

### CAP-4.2 — Referral Submission and Pipeline Management

**Business Purpose**: Formalize the referral program with a dedicated digital channel, guaranteed SLAs, and incentive tracking — replacing the current informal form-based process.

**Actors**: Any Employee (referrer), Recruiter (processor)

**Functional Behavior**:
- Employees access a dedicated "Refer a Candidate" section (authenticated) where they select the open position and submit the candidate's name, email, LinkedIn URL, and a short qualification note.
- The referred candidate receives an automated email invitation to complete their profile and apply.
- Referral applications enter a dedicated sub-pipeline in the system, visually distinct from direct applications.
- The system enforces a 48-hour screening SLA for referral applications (alert sent to Recruiter at T+24h if unreviewed).
- The referrer is notified at key milestones: "Your referral has passed screening", "Your referral has been interviewed", "Your referral has been hired".
- Incentive tracking: once a referred candidate is marked as "Hired" and the vesting period (configurable: 3–6 months) passes, HR can mark the incentive as "Paid" and the referrer receives a notification. Bonus amounts (€300–€1,000) are stored per referral record.

**Priority**: Must Have

**Traces to**: AS-IS Phase 4 (referral program), AS-IS Error E7 (unqualified referrals), KPI-04

---

## DOMAIN 5 — Screening & Shortlist

### CAP-5.1 — Structured CV Screening Workflow

**Business Purpose**: Replace the informal ATS keyword-matching + manual review sequence with a structured, auditable screening workflow that forces explicit evaluation criteria.

**Actors**: Recruiter (primary screener), Line Manager and SME (co-validators)

**Functional Behavior**:
- For each candidate in the pipeline, the Recruiter performs a manual screening evaluation using a structured checklist derived from the JR must-have requirements.
- Each must-have criterion is marked: Met / Partially Met / Not Met. A threshold rule (configurable per JR) determines minimum score to advance.
- The Recruiter adds a qualitative note and sets the candidate status: Advance to Shortlist / Reject / Hold for Later Review.
- Rejected candidates receive an automated, personalized rejection email. "Hold" candidates are preserved in the Talent Pool (CAP-2.1).
- Shortlisted candidates are bundled into a Shortlist Package shared with LM, SME, and (conditionally) the Final Client.

**Priority**: Must Have

**Traces to**: AS-IS Error E3 (ATS keyword mismatches), AS-IS Error E4 (non-uniform shortlist evaluation), E10 (no feedback to candidates), KPI-05

---

### CAP-5.2 — Collaborative Shortlist Review and Sign-off

**Business Purpose**: Enforce structured co-validation of the shortlist by all relevant stakeholders before interviews begin, preventing misaligned expectations that cause round-2 failures.

**Actors**: Recruiter, Line Manager, SME, Final Client (conditional)

**Functional Behavior**:
- The Shortlist Package is shared via the platform as a read-only view of anonymized (configurable) candidate profiles, screening scores, and Recruiter notes.
- Each reviewer (LM, SME) must submit a binary assessment per candidate: Approve for Interview / Reject + mandatory note.
- A candidate advances to the interview stage only if approved by all required reviewers.
- If reviewers disagree, a flag is raised and the Recruiter can call a synchronous alignment meeting (logged in the system).
- SLA for shortlist review: 2 working days from package delivery. Reminder sent at T+1 day.

**Priority**: Must Have

**Traces to**: AS-IS Error E4 (non-uniform shortlist review), AS-IS Delay D5 (scheduling delays cascading from late shortlist agreement)

---

## DOMAIN 6 — Interview & Evaluation

### CAP-6.1 — Interview Scheduling Module

**Business Purpose**: Centralize interview coordination to eliminate the 5–10 day inter-round delays caused by manual calendar synchronization across multiple stakeholders.

**Actors**: Recruiter (coordinator), Candidate, HR Interviewer, SME, Line Manager, Final Client (Round 3)

**Functional Behavior**:
- The Recruiter selects a time slot from a shared availability view (integrated with calendar via iCal/Google Calendar API — v2; manual slot entry in v1) and sends an interview invitation to the candidate.
- The candidate receives a branded email with interview details, interviewer names, format (video/in-person), and a one-click "Confirm / Propose Alternative" action.
- Interview reschedules are tracked with reason codes; more than 2 reschedules per round trigger an alert to HRBP.
- Each interview is linked to its corresponding evaluation scorecard (CAP-6.2) and is not considered complete until the scorecard is submitted.

**Priority**: Must Have

**Traces to**: AS-IS Delay D5 (interview scheduling), AS-IS Delay D10 (non-compressible 3-round structure)

---

### CAP-6.2 — Structured Evaluation Scorecard System

**Business Purpose**: Replace subjective, inconsistent post-interview feedback with a standardized, weighted scoring system that produces comparable outputs across all interviewers and all candidates.

**Actors**: HR Interviewer (Round 1), SME + Line Manager (Round 2), Final Client (Round 3, conditional)

**Functional Behavior**:
- Each round has a pre-defined scorecard template:
  - **Round 1 (HR)**: Soft Skills sub-dimensions (communication, adaptability, collaboration, problem-solving), Cultural Fit indicators, Motivation alignment, Salary expectation gap.
  - **Round 2 (Technical)**: Hard Skill competencies (drawn from JR must-have list), problem-solving test results (optional), technical depth score.
  - **Round 3 (Client)**: Custom criteria defined by the Final Client for the specific commessa.
- Scores are entered on a 1–5 scale per criterion. The system auto-calculates the weighted composite score: Hard Skill 50%, Soft Skill 25%, Cultural Fit 15%, Motivation/Availability 10%.
- Scorecards have a mandatory submission SLA: 24 hours after interview (configurable). At T+12h, an automatic reminder is sent. At T+24h without submission, an escalation alert is sent to HRBP.
- All scorecards are stored with full audit trail (submitted by, submitted at, last modified).

**Priority**: Must Have

**Traces to**: AS-IS Error E5 (non-uniform evaluation), AS-IS Error E6 (missing/late technical feedback), AS-IS Delay D6 (SME/LM feedback delays), KPI-07

---

### CAP-6.3 — Candidate Evaluation Comparison View

**Business Purpose**: Enable data-driven multi-candidate comparison during the alignment meeting, replacing the current ad-hoc mental synthesis with a structured side-by-side view.

**Actors**: Recruiter, HRBP, Line Manager, SME, Direzione di Funzione

**Functional Behavior**:
- Once all rounds are complete for a shortlist cohort, the Recruiter can open a Comparison View showing all candidates side-by-side with: composite score, per-dimension scores, round-by-round progression, and interviewer notes.
- The view can be sorted and filtered by dimension score, round score, or overall rank.
- This view serves as the primary artifact for the alignment meeting (CAP-6.4).
- Candidate names can be anonymized for bias-reduction during review (configurable).

**Priority**: Should Have

**Traces to**: AS-IS Phase 7 (multi-criterio evaluation), AS-IS Error E5

---

### CAP-6.4 — Alignment Meeting Facilitation and Decision Recording

**Business Purpose**: Formalize the decision outcome of the alignment meeting with a structured record, ensuring the selected candidate and rationale are captured and traceable.

**Actors**: HR (facilitator), Line Manager, SME, Direzione (optional), Final Client (commessa)

**Functional Behavior**:
- The Recruiter can schedule an Alignment Meeting from within the platform, with automated calendar invite to participants.
- Post-meeting, the Recruiter records the decision: Selected Candidate, Runner-Up(s), Rejection rationale per non-selected candidate, agreed start date, and any special offer conditions.
- The decision record triggers the Offer workflow (DOMAIN 7).

**Priority**: Must Have

**Traces to**: AS-IS Phase 7 (alignment meeting), AS-IS Error E6

---

## DOMAIN 7 — Offer Management

### CAP-7.1 — Offer Generation and Internal Approval

**Business Purpose**: Streamline offer preparation with pre-approved templates and Finance/Legal validation, eliminating unstructured email chains and ensuring market-aligned compensation.

**Actors**: Recruiter (initiator), Finance (RAL validation), Legal (contract review), HRBP (sign-off)

**Functional Behavior**:
- Upon candidate selection, the Recruiter initiates an Offer Request with: proposed RAL, start date, contract type, benefits package, and any special clauses.
- The system validates the proposed RAL against the authorized range in the approved RFR. If outside range, Finance approval is required.
- Legal review is triggered if the contract includes non-standard clauses (NDA, client-specific terms, exclusivity).
- A configurable market benchmark alert is shown to HR during offer preparation (based on seniority band and role type), addressing AS-IS Error E8.
- Approved offer data is merged into a PDF offer letter template.

**Priority**: Must Have

**Traces to**: AS-IS Phase 7, AS-IS Error E8 (below-market offers), AS-IS Delay D7 (offer negotiation delays), KPI-08

---

### CAP-7.2 — Digital Offer Delivery and Signature

**Business Purpose**: Deliver the offer to the candidate through a secure, auditable digital channel with a time-bounded response window and built-in counter-offer handling.

**Actors**: Candidate, Recruiter, HR

**Functional Behavior**:
- The candidate receives a branded offer portal link via email. The portal displays the offer letter with a clean UI.
- The candidate can: Accept (e-signature captured), Decline (with optional reason), or Submit Counter-Offer (structured form with proposed RAL and/or start date modification).
- The response window is 3 calendar days (configurable per offer). Automated reminders are sent at T+1 and T+2.5 days.
- Counter-offers are routed back through Finance (if RAL adjustment) and trigger a new offer cycle. A maximum of 2 counter-offer rounds is enforced before the offer is escalated to HRBP for decision.
- Upon acceptance, the signed offer PDF is stored and the Onboarding workflow (DOMAIN 8) is automatically triggered.
- Upon decline (or no response), the Runner-Up candidate from the alignment meeting is automatically notified to the Recruiter for next-step decision.

**Priority**: Must Have

**Traces to**: AS-IS Phase 7 (ATS offer with digital signature), AS-IS Delay D7, KPI-08

---

## DOMAIN 8 — Onboarding Activation

### CAP-8.1 — Parallel Onboarding Task Orchestration

**Business Purpose**: Automatically activate all onboarding workstreams (IT, Facility, Payroll, HR, Line Manager) in parallel upon offer acceptance, with T-5 pre-alerts to prevent Day-1 failures.

**Actors**: IT Department, Facility Management, Payroll/HR Admin, HR (onboarding coordinator), Line Manager

**Functional Behavior**:
- Upon offer acceptance, the system auto-creates an Onboarding Task Board for the new hire with parallel task tracks:
  - **IT**: Device procurement, system access setup, email account creation, software licenses
  - **Facility**: Workstation assignment, physical access credentials
  - **Payroll**: Employee record creation, contract upload, payroll system registration
  - **HR**: Onboarding agenda creation, orientation sessions scheduling, documentation checklist
  - **Line Manager**: Buddy assignment, 30-60-90 day plan preparation
- Each track has a configurable task list with individual deadlines.
- The T-5 days alert rule (from Day 1 date) triggers automatic notifications to all task owners if any task is incomplete. Alert escalates to HRBP at T-2.
- For commessa roles, the Final Client receives an automated formal notification email with the new hire's name, start date, and role.

**Priority**: Must Have

**Traces to**: AS-IS Phase 8, AS-IS Delay D8 (late IT/Facility activation), KPI-10 (Onboarding Readiness ≥85%)

---

### CAP-8.2 — Onboarding Readiness Dashboard

**Business Purpose**: Give HR a consolidated view of all pending onboardings and their completion status, enabling proactive intervention before Day-1.

**Actors**: HR, HRBP

**Functional Behavior**:
- A dedicated view lists all upcoming onboardings within the next 30 days with: new hire name, start date, role, and a per-track progress bar (IT, Facility, Payroll, HR, LM).
- Overall readiness score per new hire (% tasks completed) is shown with RAG indicator.
- Drill-down into individual onboardings shows the full task checklist with status, owner, and due date.

**Priority**: Must Have

**Traces to**: AS-IS Delay D8, KPI-10

---

## DOMAIN 9 — KPI Dashboard & Analytics (Opportunity I2)

### CAP-9.1 — Real-Time KPI Dashboard

**Business Purpose**: Implement Opportunity I2 from the TO-BE analysis: a real-time dashboard aggregating all 10 KPIs of the framework with RAG status, trend visualization, and drill-down by seniority, function, and sourcing channel.

**Actors**: HRBP (Analytics Owner), Recruiter, Line Manager (own KPIs only), Direzione di Funzione (executive view)

**Functional Behavior**:
- The dashboard homepage displays all 10 KPIs in card format with: current value, target, RAG status (Green/Yellow/Red), and trend arrow (improving/stable/degrading vs. previous month).
- Each KPI card links to a detail view with: 12-month trend chart, monthly value table, and breakdown by seniority/function/channel.
- The dashboard data refreshes based on live records in the system (updated as workflow steps are completed).
- Role-based views: HRBP and Recruiter see all KPIs across all open positions. Line Managers see only their own Time to Feedback and positions they're involved in.

**KPI Catalog**:

| ID | KPI Name | Target | Direction |
|----|----------|--------|-----------|
| KPI-01 | Time to Fill | ≤30 days (J/M), ≤45 (Senior) | Lower is better |
| KPI-02 | Approval Cycle Time | ≤5 working days | Lower is better |
| KPI-03 | Internal Sourcing Rate | ≥20% | Higher is better |
| KPI-04 | Referral Conversion Rate | ≥25% | Higher is better |
| KPI-05 | Screening-to-Interview Rate | 10–20% | Window |
| KPI-06 | Interview-to-Offer Rate | 15–30% | Window |
| KPI-07 | Time to Feedback (SME/LM) | ≤2 days | Lower is better |
| KPI-08 | Offer Acceptance Rate | ≥85% | Higher is better |
| KPI-09 | Quality of Hire Index | ≥75/100 | Higher is better |
| KPI-10 | Onboarding Readiness Index | ≥85% | Higher is better |

**Priority**: Must Have

**Traces to**: TO-BE Opportunity I2, all KPIs, AS-IS zero-visibility gap

---

### CAP-9.2 — Automated KPI Alerts and Notifications

**Business Purpose**: Convert KPI monitoring from a reactive (monthly review) activity to a proactive (real-time alert) one, enabling corrective action before bottlenecks become critical.

**Actors**: HRBP (alert recipient and action owner), relevant stakeholders per alert type

**Functional Behavior**:
- Alert thresholds are configurable per KPI with two levels: Warning (Yellow) and Critical (Red).
- When a KPI crosses a threshold, an alert is generated and delivered via: in-app notification + email.
- Alert routing is role-based: KPI-07 (Time to Feedback) alerts go to the specific LM/SME + HRBP; KPI-01 alerts go to Recruiter + HRBP; KPI-10 alerts go to HR Onboarding Coordinator.
- Each alert requires acknowledgment + action logging within the platform. Unacknowledged critical alerts escalate after 24 hours.
- Alert fatigue prevention: no more than 3 active critical alerts per user at any time; older alerts auto-group.

**Priority**: Must Have

**Traces to**: TO-BE I2 (alert automatici), AS-IS Delay D6, D8, D9

---

### CAP-9.3 — Monthly Review Report Generation

**Business Purpose**: Support the structured monthly HR + LM + Direzione review meeting with an auto-generated report that provides an objective process snapshot.

**Actors**: HRBP (report owner), HR Analytics Owner

**Functional Behavior**:
- At the end of each calendar month, the system auto-generates a Monthly Review Report as a PDF/Excel export.
- Report sections: KPI scorecard (all 10 KPIs), open positions summary, candidate funnel by stage, top 3 bottlenecks by time elapsed, referral program performance, and onboarding readiness summary.
- The report can be emailed to a configured distribution list automatically.

**Priority**: Should Have

**Traces to**: TO-BE I2 (review cadence), AS-IS absence of structured reporting

---

### CAP-9.4 — Sourcing Channel Performance Analysis

**Business Purpose**: Enable data-driven sourcing decisions by tracking conversion rates per channel (Career Page, LinkedIn, Indeed, InfoJobs, Referral, Agency, Internal).

**Actors**: Recruiter, HRBP

**Functional Behavior**:
- For each open and closed position, the system tracks candidate source at entry.
- A dedicated Sourcing Analytics view shows per-channel: total applications, screening pass rate, interview-to-offer rate, offer acceptance rate, average time-to-fill, and cost per hire (if cost data is entered manually).
- Data is filterable by seniority, time period, and business unit.

**Priority**: Should Have

**Traces to**: KPI-04, TO-BE I2 (segmentazione per canale di sourcing)

---

## DOMAIN 10 — Platform Administration & Compliance

### CAP-10.1 — Role-Based Access Control (RBAC)

**Priority**: Must Have

**Functional Behavior**: 10 distinct roles with differentiated permissions across all modules: HRBP, Recruiter, HR Interviewer, Line Manager, SME, Finance, Legal, Director, IT/Facility/Payroll, Candidate (external portal). Permissions enforce data minimization (e.g., Finance sees only budget-related fields, not interview scorecards).

---

### CAP-10.2 — GDPR Data Retention Management

**Priority**: Must Have

**Functional Behavior**: Automated data retention enforcement per candidate type: 12 months (rejected after screening), 24 months (interviewed but not hired), 36 months (candidates who received an offer). At retention expiry, candidates receive an email asking if they consent to extended storage; absent response triggers anonymization. All retention decisions are audit-logged.

---

### CAP-10.3 — Full Audit Trail

**Priority**: Must Have

**Functional Behavior**: Every state change in the system (RFR submission, approval/rejection, status changes, scorecard submissions, offer sends) is recorded with actor, timestamp, and before/after values. The audit log is read-only, append-only, and exportable for compliance review.

---

### CAP-10.4 — System Configuration and KPI Threshold Management

**Priority**: Should Have

**Functional Behavior**: Platform administrators (HRBP-level) can configure: SLA durations per approval step, KPI alert thresholds, onboarding task checklists per role type, email notification templates, and referral incentive amounts and vesting periods.

---

## MoSCoW Summary

| Priority | Capabilities |
|----------|-------------|
| **Must Have** | CAP-1.1, 1.2, 1.3, 2.1, 3.1, 3.3, 4.1, 4.2, 5.1, 5.2, 6.1, 6.2, 6.4, 7.1, 7.2, 8.1, 8.2, 9.1, 9.2, 10.1, 10.2, 10.3 |
| **Should Have** | CAP-2.2, 3.2, 6.3, 9.3, 9.4, 10.4 |
| **Could Have** | Calendar API integration (v2), LinkedIn auto-posting (v2), AI CV ranking (v2) |
| **Won't Have (v1)** | Full ATS replacement, payroll bidirectional sync, native mobile app |

---

*Document owner: Senior Business Analyst — HR Technology Program*
*Version: 1.0 | Status: Draft for Review*
