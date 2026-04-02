# Overview — Hiring Management Web Application

## Project Summary

This document provides the executive overview for the **Hiring Management Web Application**, a Node.js-based platform designed to digitize, automate, and govern the end-to-end employee hiring process. The application is being built to replace the current fragmented AS-IS process — characterized by manual handoffs, ATS-dependent bottlenecks, and the absence of real-time visibility — with a structured, data-driven TO-BE workflow aligned with the KPI framework defined in the strategic analysis.

---

## Business Problem

The current hiring process (AS-IS) is an eight-phase sequential workflow managed primarily through a third-party ATS (Teamtailor), email chains, and manual coordination. An internal audit and BA analysis surfaced **21 operational improvement opportunities**, with the following critical pain points:

- **No real-time KPI visibility**: HR, Line Managers, and senior management operate without a consolidated view of process performance. Bottlenecks are identified reactively, after significant delay.
- **High time-to-fill variability**: Average time-to-fill reaches 43–52 days in critical months, well above the ≤30-day target for junior/mid profiles.
- **Accountability gaps**: SME and Line Manager feedback after interviews is inconsistently tracked; 6 out of 11 observed months exceeded the ≤2-day feedback target.
- **Fragmented stakeholder coordination**: Scheduling, approvals, and offer negotiations involve 5–7 actors with no centralized orchestration layer.
- **Onboarding failures**: 5 out of 11 months registered incomplete onboarding readiness by Day 1, due to late activation of IT and Facility workflows.
- **Manual approval chain**: The RFR (Richiesta di Fabbisogno Risorse) approval cycle passes through HRBP → Finance → Legal → Direzione Funzione with no SLA enforcement or automated escalation.

---

## Solution Vision

The application is a **self-hosted Node.js web platform** that acts as the central orchestration layer for the hiring process. It does not replace the ATS entirely but integrates with it and adds the missing governance, automation, and analytics layers.

The system implements the **TO-BE Trasformativo** process model, incorporating:

1. **Structured digital workflow** for all eight hiring phases, with enforced role-based routing, SLA timers, and automated escalation.
2. **Real-time KPI Dashboard** (Opportunity I2 from the TO-BE analysis) with drill-down by seniority, role type, and sourcing channel.
3. **Proactive alert system** that notifies relevant actors when KPI thresholds are breached, converting reactive management into preventive governance.
4. **Evaluation module** with standardized interview scorecards, weighted scoring (Hard Skill 50% / Soft Skill 25% / Cultural Fit 15% / Motivation 10%), and inter-rater consistency tracking.
5. **Onboarding activation engine** that triggers IT, Facility, and Payroll tasks in parallel at offer acceptance, with T-5 day pre-alerts.
6. **Referral management portal** with a dedicated pipeline, guaranteed 48-hour screening SLA, and incentive tracking.

---

## Strategic Objectives

| # | Objective | Primary KPI | Target |
|---|-----------|-------------|--------|
| O1 | Reduce average Time to Fill | KPI-01 | ≤30 days (Junior/Mid), ≤45 days (Senior) |
| O2 | Enforce approval cycle compliance | KPI-02 | ≤5 working days |
| O3 | Increase internal sourcing effectiveness | KPI-03 | ≥20% positions closed internally |
| O4 | Improve referral conversion | KPI-04 | ≥25% conversion rate |
| O5 | Accelerate post-interview feedback | KPI-07 | ≤2 days from SME/LM |
| O6 | Ensure Day-1 onboarding completeness | KPI-10 | ≥85% readiness index |
| O7 | Maintain offer acceptance rate | KPI-08 | ≥85% |

---

## Scope

### In Scope

- Digital RFR creation, routing, and multi-level approval workflow
- Internal talent pool search (pre-sourcing phase)
- Job Requisition management and multi-channel publication tracking
- Candidate pipeline management with referral sub-pipeline
- Interview scheduling coordination (Round 1, 2, 3)
- Structured scorecard-based evaluation with weighted scoring
- Offer generation, digital signature workflow, and counter-offer handling
- Parallel onboarding task activation for IT, Facility, Payroll, HR
- Real-time KPI dashboard with 10-KPI framework, RAG status, monthly trends
- Automated SLA alerts and escalation notifications
- GDPR-compliant data retention management (12–36 months by candidate type)
- Role-based access control (HRBP, Recruiter, Line Manager, SME, Finance, Legal, Director, IT, Facility, Payroll)

### Out of Scope (v1.0)

- Full replacement of the ATS (Teamtailor integration is read-only in v1)
- Direct LinkedIn / Indeed API posting (tracked manually, automated in v2)
- Payroll system integration (notification only, no bidirectional sync)
- AI-based CV ranking or automated candidate scoring
- Mobile native application (responsive web only)

---

## Primary Stakeholders

| Role | Type | Primary Interaction |
|------|------|---------------------|
| Recruiter / Talent Acquisition | Internal — Power User | Daily operations across all phases |
| HR Business Partner (HRBP) | Internal — Process Owner | RFR validation, internal sourcing, KPI dashboard, escalation |
| Line Manager | Internal — Contributor | RFR creation, shortlist review, technical interview, offer sign-off |
| SME (Subject Matter Expert) | Internal — Contributor | JD input, technical interview, scorecard submission |
| Finance & Accounting | Internal — Approver | Budget validation in RFR approval chain |
| Legal / Compliance | Internal — Approver | Conditional step in RFR chain; offer contract review |
| Direzione di Funzione | Internal — Final Approver | Final RFR sign-off; executive KPI view |
| IT / Facility / Payroll | Internal — Notified | Onboarding task recipients |
| Candidate | External | Application portal, interview scheduling, offer acceptance |
| Final Client (commessa roles) | External — Conditional | JD review, final interview round, onboarding notification |

---

## High-Level Process Flow (TO-BE)

```
[Fabbisogno Identificato]
        ↓
[RFR Creation — Line Manager]
        ↓
[Multi-Level Approval Workflow — HRBP → Finance → Legal* → Direzione]
        ↓
[Internal Talent Pool Search — 2-5 days]
        ↓ (if no internal candidate)
[Job Requisition Opening + Multi-channel Publication]
        ↓
[Candidate Intake — Direct + Referral Pipeline]
        ↓
[Screening — Auto-filter + Manual Recruiter Review]
        ↓
[Shortlist Co-validation — LM + SME + Client*]
        ↓
[Round 1: HR Interview → Round 2: Technical → Round 3: Client*]
        ↓
[Weighted Scoring + Alignment Meeting]
        ↓
[Offer Preparation — HR + Finance + Legal → Digital Signature]
        ↓
[Parallel Onboarding Activation — IT / Facility / Payroll / HR / LM]
        ↓
[Risorsa Operativa]
```
*Conditional steps

---

## Technology Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Runtime | Node.js (v20 LTS) | Requested by client; excellent async I/O for workflow engine |
| Web Framework | Express.js | Lightweight, well-documented, large ecosystem |
| Database | PostgreSQL | Relational integrity for complex workflow state; audit log support |
| ORM | Prisma | Type-safe DB access, schema migrations, excellent Node.js DX |
| Authentication | JWT + bcrypt | Stateless auth suitable for REST API; role claims in token |
| Frontend | React (Vite) + Tailwind CSS | Single-page app for dashboard and workflow UI |
| Charting / KPI | Recharts or Chart.js | Embedded KPI dashboard without external BI dependency |
| Notifications | Node-cron + Nodemailer | SLA timer checks + email alert delivery |
| File Storage | Local FS / S3-compatible | CV uploads, signed offer documents |
| Deployment | Docker + docker-compose | Environment parity; easy self-hosting |

---

## Expected Benefits (TO-BE Simulation — 12 months)

Based on the KPI simulation documented in `TO-BE_Guida_e_Analisi_Impatti.docx`:

| Metric | AS-IS | TO-BE | Improvement |
|--------|-------|-------|-------------|
| Months in target — Time to Fill | 5/11 | 7/11 | +2 months |
| Months in target — Approval Cycle | 9/11 | 10/11 | +1 month |
| Months in target — Referral Conversion | 5/11 | 7/11 | +2 months |
| Months in target — Time to Feedback | 5/11 | 8/11 | **+3 months** |
| Months in target — Onboarding Readiness | 5/11 | 8/11 | **+3 months** |
| Estimated annual benefit (conservative) | — | — | **€58,500–€125,000** |
| Estimated implementation cost (Year 1) | — | — | **€5,700–€17,600** |
| **ROI Year 1** | — | — | **+232% (conservative)** |

---

## Project Phases

| Phase | Description | Deliverable |
|-------|-------------|-------------|
| Phase 0 — Spec & Design | Requirements, wireframes, data model | This spec set (4 .md files) |
| Phase 1 — Core Workflow | RFR, Approval, Internal Search, JR, Candidate Intake | Working workflow engine |
| Phase 2 — Interview & Evaluation | Scheduling, Scorecards, Shortlist | Evaluation module |
| Phase 3 — Offer & Onboarding | Offer workflow, digital sign, onboarding tasks | Offer + onboarding module |
| Phase 4 — KPI Dashboard | 10-KPI real-time dashboard, alerts, RAG status | Analytics module (I2) |
| Phase 5 — Hardening | GDPR, security audit, performance testing | Production-ready release |

---

*Document owner: Senior Business Analyst — HR Technology Program*
*Version: 1.0 | Status: Draft for Review*
