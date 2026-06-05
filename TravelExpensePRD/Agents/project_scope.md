# Enterprise Employee Travel & Expense Management System
## Project Scope Document

**Version:** 1.0
**Status:** Approved for Engineering Kickoff
**Based on:** TravelExpensePRD v1.0 & KPI Blueprint v1.0
**Target Go-Live:** Q3 2026
**Last Updated:** 2026-06-05

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [KPI Points with Constraints](#2-kpi-points-with-constraints)
3. [Functional Requirements](#3-functional-requirements)
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [In-Scope Development Portions](#5-in-scope-development-portions)
6. [Out-of-Scope Items](#6-out-of-scope-items)
7. [Stopping Points / Definition of Done](#7-stopping-points--definition-of-done)
8. [Risks & Dependencies](#8-risks--dependencies)

---

## 1. Project Overview

### Problem Statement

The enterprise currently operates employee travel and expense reporting through **decentralized, manual channels** (email attachments, Excel spreadsheets, paper receipts, phone-based approvals), resulting in:

| Pain Point | Current State | Business Impact |
|---|---|---|
| Reimbursement Cycle | 18–22 days average | Employee frustration, cash flow burden |
| Policy Violations | 12–15% of submitted expenses | Post-payment leakage, audit risk |
| Finance Overhead | ~120 person-hours/week | High operational cost, error-prone |
| Spend Visibility | None (real-time) | No accrual tracking, CFO blind spots |
| Audit Evidence | Two consecutive audit failures | Compliance and regulatory exposure |

### Vision

A **single, cloud-native Travel & Expense (T&E) platform** that automates the full lifecycle — from travel request → pre-approval → real-time expense capture → policy-based audit → multi-level approval routing → ERP posting → employee reimbursement — with zero manual handoffs for in-policy expenses.

---

## 2. KPI Points with Constraints

This section maps each business KPI to its target, measurement methodology, technical constraints, and data aggregation rules.

---

### KPI 1 — Expense Processing Cycle Time

| Attribute | Detail |
|---|---|
| **Business Goal** | Reduce median time from expense submission to reimbursement posting |
| **Baseline** | 18–22 days |
| **Target** | **≤ 4 days** (median) from submission to reimbursement |
| **Formula** | `Median(T_reimbursed – T_submitted)` across all eligible reports |
| **Telemetry Sources** | `expense_report.submitted_at` → `reimbursement_transaction.settled_at` OR `finance_ledger.posted_at` |
| **Measurement** | System workflow timestamp delta; p50 and p90 tracked separately |

#### Constraints

- **Include Only:**
  - Reports with `is_test = FALSE`
  - Submission channels: `WEB`, `MOBILE_APP`, `API`
  - Base functional currency (USD) to avoid FX clearing delays
  - Reimbursement method: ACH only (no manual checks)

- **Exclude:**
  - Reports with cash advances (`advanced_cash_flag = TRUE`) — settle on separate cycles
  - Multi-month travel reports (`multi_month_travel = TRUE`, trips > 90 days)
  - Reports held for `FRAUD_INVESTIGATION` or `TAX_CORRECTION`
  - Outliers: `settled_at – submitted_at > 180 days`

- **Reporting Constraint:** CFO dashboard tracks variance against 7-day target (not just 4-day); daily refresh required.

---

### KPI 2 — Travel Policy Compliance Rate

| Attribute | Detail |
|---|---|
| **Business Goal** | Reduce expense policy leakage from 12–15% to < 5% |
| **Baseline** | 72% (pre-audit) |
| **Target** | **≥ 95%** (real-time policy enforcement) |
| **Primary Formula** | `(Line items with no violation / Total submitted line items) × 100` |
| **Secondary Formula** | `(Sum of compliant line amounts / Total submitted amount) × 100` |
| **Measurement** | Policy engine rejection/override logs; both item-count and amount-weighted variants |

#### Constraints

- **Compliant Definition (SQL logic):**
  ```sql
  CASE
    WHEN pv.violation_flag = FALSE THEN 'COMPLIANT'
    WHEN pv.violation_flag = TRUE
      AND ea.approved_at IS NOT NULL
      AND ea.override_type IN ('POLICY_WAIVER', 'CORRECTED_RECEIPT') THEN 'COMPLIANT'
    ELSE 'NON_COMPLIANT'
  END
  ```

- **Include Only:**
  - Reports submitted within the rolling 30-day window
  - Production environment only (not sample/test data)
  - Line items with `status != 'CANCELLED'`

- **Exclude:**
  - Line items where `exception_approval.override_reason = 'SYSTEM_ERROR'` do NOT count as compliant
  - `expense_report.is_sample_data = TRUE` rows

- **Policy Rules Enforced at Submission (P-01 Engine):**

  | Rule ID | Condition | Action |
  |---|---|---|
  | P-01-01 | Meal > $75/person/day (Tier 1 city) | Flag — requires manager justification |
  | P-01-02 | Duplicate receipt hash (same employee) | **Hard block** — submission prevented |
  | P-01-03 | Hotel without corresponding approved travel request | Flag — requires VP override |
  | P-01-04 | Business class flight, domestic route < 4 hours | **Auto-reject** at submission |
  | P-01-05 | Expense date > 60 days past trip end | **Hard block** — expense expired |

---

### KPI 3 — Manual Auditing Overhead

| Attribute | Detail |
|---|---|
| **Business Goal** | Reduce finance team manual effort through automation |
| **Baseline** | ~120 person-hours/week |
| **Target** | **≤ 20 hours/week** (exception-only auditing) |
| **Formula** | `Σ hours_logged (per finance auditor, expense tasks only)` per week |
| **Normalized Formula** | `(Weekly overhead hours / Weekly submitted reports) × 1,000` |
| **Measurement** | Time-tracking system + application audit log |

#### Constraints

- **Task Tags Tracked:** `EXPENSE_AUDIT`, `MANUAL_RECEIPT_OCR_CORRECTION`, `POLICY_ESCALATION_HANDLE`, `APPROVER_CHASE`
- **Include Only:**
  - Users with role `FINANCE_AUDITOR` or `EXPENSE_ADMIN`
  - Time entries Monday 00:00 – Sunday 23:59 (reporting week boundary)
  - Audit log inferred durations: clamped to **30 seconds minimum, 60 minutes maximum** per report

- **Exclude:**
  - System maintenance, policy configuration, and user training time (`PROJECT_OVERHEAD` tag)
  - Automated system-user actions from audit log
  - Time entries with no expense-related tag (even if user is an auditor)

- **Additional Success Criteria:**
  - Zero finance manual intervention for in-policy, single-approval expenses **under $500**
  - `< 0.5%` duplicate reimbursement rate (baseline was ~4%)

---

### KPI 4 — Employee Net Promoter Score (eNPS) — T&E Domain

| Attribute | Detail |
|---|---|
| **Business Goal** | Drive adoption and reduce shadow finance processes |
| **Baseline** | Not formally measured / estimated ~25 |
| **Target** | **≥ +65** within 90 days of launch |
| **Formula** | `((#Promoters – #Detractors) / #Total Respondents) × 100` |
| **Scale** | 0–10; Detractors (0–6), Passives (7–8), Promoters (9–10) |
| **Measurement** | Quarterly eNPS survey embedded in portal, triggered post-reimbursement |

#### Constraints

- **Survey Trigger:** Async job after `reimbursement_transaction.settled_at`; survey expires in 3 days
- **Include Only:**
  - First response per `user_id` per rolling 90 days (no polling bias)
  - Surveys for reports with `total_amount >= $10` (micro-expenses excluded)
  - Responses submitted within 7 days of reimbursement

- **Exclude:**
  - Test users (`is_employee = FALSE`)
  - Contractors with tenure < 3 months
  - Responses submitted > 7 days after reimbursement (memory decay invalidation)

---

### KPI 5 — System Adoption Rate

| Attribute | Detail |
|---|---|
| **Business Goal** | Replace all offline/manual expense channels |
| **Baseline** | N/A (no prior system) |
| **Target** | **≥ 85%** within first 90 days of launch |
| **Formula** | `(Active users who submitted ≥1 report / Total employees with expense need in month) × 100` |
| **Measurement** | HRIS-active employee count vs. system login + expense submission |

#### Constraints

- **Denominator Logic:** Employees with `expense_eligible = TRUE` AND (`travel_frequency_code IN ('HIGH','MEDIUM')` OR corporate card transaction exists with `is_auto_imported_flag = FALSE`)
- **Include Only:**
  - Users with `employment_status = 'ACTIVE'` and `hire_date <= first_day_of_month`
  - Submissions via `WEB`, `MOBILE_APP`, `EMAIL_PARSER`

- **Exclude:**
  - Employees on long-term leave (`leave_of_absence_flag = TRUE`)
  - Legacy system batch imports (not counted as system adoption)
  - One-time project contractors with no repeat need

- **Measurement Frequency:** Monthly batch job on day 1 of the following month

---

### System Performance KPIs (Technical Guardrails)

| Metric | Target | Alert Threshold |
|---|---|---|
| OCR p95 Latency | < 1,200 ms | Alert > 2,000 ms for 5% of requests |
| OCR Amount Accuracy | ≥ 99.0% | Alert if < 95% over 1 hour |
| OCR Merchant Accuracy | ≥ 97.0% | Alert if < 95% over 1 hour |
| OCR Manual Adjustment Rate | < 5% | Alert > 12% per rolling hour |
| HRIS Webhook Failure Rate | < 0.1% | Alert at 2% over 15-min window |
| NetSuite Posting Error Rate | < 0.5%/hour | Alert at 3% |
| Bank API Token Failures | 0 per day | Immediate alert on any failure |
| Bank API Import Latency (p95) | < 45 minutes | Alert > 2 hours |
| P-01 Policy Engine p99 Latency | < 350 ms | Alert > 800 ms |
| Policy Rule Throughput | > 5,000 items/sec | Horizontal scale trigger |
| Executive Dashboard Refresh | ≤ 5 minutes (operational) | Daily aggregated (CFO view) |

---

## 3. Functional Requirements

### Module A — Travel Request & Pre-Approval Workflow

#### A1. Employee Travel Request Submission
- Employee logs in; system pre-populates HRIS fields (Employee ID, cost center, legal entity, manager email)
- Fields required: trip purpose, destination(s), dates, estimated costs (flight, hotel, transport, meals, misc.)
- Business justification (min. 50 characters) required for estimated totals > $10,000
- Optional cash advance: max 80% of estimated total, capped at $5,000 per trip
- Policy rule **R-TR-01** applied: international trips > 7 days in business class require VP approval; domestic = economy only
- Status transitions: `DRAFT` → `PENDING_MANAGER_APPROVAL`

#### A2. Manager Review & Approval
- Manager receives email + in-app notification
- Displays: cost breakdown, policy compliance flags, remaining team travel budget (from ERP)
- Actions: **Approve** / **Reject** (with required reason) / **Request Changes**
- Approval triggers: cash advance → separate Finance Auditor queue
- Escalation: auto-escalate to manager's manager if no action within **48 hours**

---

### Module B — Real-Time Expense Tracking & Automated Capture

#### B1. Mobile OCR Receipt Capture
- Camera-based capture supporting thermal, digital, and printed receipts
- OCR engine (Google Document AI or Abbyy) extracts: merchant name, transaction date, total amount (incl. tax), currency (ISO 4217), line items (if per-diem)
- ML-based merchant taxonomy auto-maps to expense category
- Duplicate receipt detection via hash matching (same merchant + date + amount ± $0.50) → alert user

#### B2. Multi-Currency Expense Handling
- Expenses entered in local currency auto-converted to USD using Open Exchange Rates or corporate bank feed
- Exchange rate applied for the date of transaction
- If rate deviates > 10% from prior day → flagged for Finance Auditor review

#### B3. Corporate Card Auto-Import
- Corporate card transactions auto-imported via Amex/bank API
- `is_auto_imported_flag = TRUE` for auto-imported transactions
- Employees confirm or add business purpose

---

### Module C — Automated Audit, Policy Validation & Approval Routing

#### C1. P-01 Policy Engine
- Evaluates all line items at submission time
- Hard blocks: submission prevented (P-01-02, P-01-04, P-01-05)
- Soft flags: requires justification + next-level approver routing (P-01-01, P-01-03)
- Policy evaluation target: < 350 ms p99; asynchronous hold if > 2 seconds

#### C2. Dynamic Multi-Level Approval Routing

| Expense Amount | First Approver | Additional Approvers |
|---|---|---|
| $0 – $500 | Direct Manager | None |
| $501 – $2,500 | Direct Manager | Department Head |
| $2,501 – $10,000 | Direct Manager → Dept Head | Finance Controller |
| > $10,000 | Direct Manager → Dept Head → Finance Controller | VP of Finance |

- Parallel approval for cross-charged expenses (both cost center managers)
- 48-hour SLA per approver; daily reminder; auto-escalation on breach

#### C3. Exception Approval Flow
- Finance Auditor reviews out-of-policy and manual audit hold queues
- Override actions: approve with reason (dropdown), reject with notes, request additional documentation
- Exception approval cycle tracked: `exception_request.created_at` → `exception_approval.approved_or_denied_at`

---

### Module D — Settlement, Banking & Reimbursement Pipelines

#### D1. Reimbursement Batch Processing
- Approved reports batched every 2 hours (or on-demand)
- NetSuite REST API called to create AP entry with GL coding (from HRIS cost center + account mapping)
- Bank API (J.P. Morgan or Stripe Connect) triggers ACH/wire payment

#### D2. Employee Reimbursement & Notification
- Employee receives email: report ID, amount, transfer date, receipt archive link
- Corporate card expenses: card issuer reconciliation via Amex API (no cash to employee)
- Payment failure: marked as "Payment Exception" and assigned to Finance Auditor within **15 minutes**

---

### Module E — Reporting & Executive Dashboards

#### E1. CFO Dashboard
- Realized Leakage Recovery (MTD & QTD)
- Outstanding Liability Accrual (with 30-day forecast)
- Processing Cycle Time — p50/p90 trend
- Automation ROI gauge (`(Manual hours saved × hourly rate) / System cost`)
- Compliance Rate — stacked bar (compliant / overridden / rejected)
- Variance-to-Target table for all core KPIs

#### E2. Finance Audit Lead Dashboard
- Real-time Audit Queue Density (streaming, 10-second refresh)
- Manager SLA Violations — top 10 offenders, Slack/email nudge integration
- High-Frequency Policy Failure Alerts (≥3 violations from one user in 60 min; CRITICAL codes)
- OCR Correction Heatmap by field type and department
- API Sync Health (HRIS, NetSuite, Bank)
- Operational Throughput overlay with auditor headcount

---

## 4. Non-Functional Requirements

### 4.1 Scalability
| Requirement | Specification |
|---|---|
| Concurrent Users | 10,000 active employees; 3,000 concurrent sessions (peak Mon 9–11 AM) |
| Report Submission Throughput | ≥ 500 reports/minute |
| Database Architecture | Read replica for reporting; no write performance degradation |
| Infrastructure | Kubernetes: min 6 pods, auto-scale to 30; stateless API servers |
| Policy Engine Scale | Horizontal scaling to maintain > 5,000 line items/sec throughput |

### 4.2 Compliance & Privacy
| Standard | Requirement |
|---|---|
| **GDPR (EU)** | Right to erasure within 30 days for former employees; 7-year retention for tax audit logs |
| **SOC 2 Type II** | Immutable audit logs (all access, approvals, overrides, reimbursements) for 1 year |
| **Tax Audit (USA, Germany, Singapore)** | Receipt images in original resolution for 7 years; bulk export API for tax authorities |
| **Data Retention** | Raw event logs: 90 days; Aggregated KPIs: 7 years |

### 4.3 Security
| Control | Specification |
|---|---|
| **Authentication** | SSO via Azure AD / Okta with SAML 2.0; SCIM auto-deprovisioning on HRIS departure |
| **MFA** | Required for Finance Auditors, System Admins, and Travel Desk roles (TOTP or WebAuthn) |
| **Authorization** | RBAC enforced at API and UI layers; responses filtered by role + cost center |
| **Encryption** | AES-256 at rest; TLS 1.3 in transit |
| **PII Protection** | No PII or receipt images stored in logs; receipt access via `receipt_id` reference only |

### 4.4 Availability & Reliability
| Metric | Target |
|---|---|
| System Uptime | ≥ 99.9% (monthly) |
| Dead-Letter Queue Monitoring | Daily review for failed webhooks exceeding 3 retries |
| Idempotency | All ingestion pipelines (webhooks, OCR) use idempotency keys to prevent duplicate KPI increments |
| Timestamps | All stored in UTC (microsecond precision); timezone conversion at presentation layer only |

### 4.5 Data Architecture Standards
- **SCD Type 2** for policy rules and employee hierarchies (preserve historical compliance accuracy)
- **Sensitive fields** (receipt images, PII) not materialized in analytical tables — references only
- **All KPI dashboards** use dedicated read replicas; no analytical queries on transactional DB

---

## 5. In-Scope Development Portions

The following modules and components are **explicitly within scope** for the Q3 2026 delivery:

### 5.1 Core Application Modules

| Module | Component | Priority |
|---|---|---|
| **Module A** | Travel Request creation and HRIS pre-population | P0 |
| **Module A** | Multi-level pre-approval workflow engine | P0 |
| **Module A** | Cash advance request flow | P1 |
| **Module B** | Mobile OCR receipt capture (iOS & Android) | P0 |
| **Module B** | Multi-currency conversion with Open Exchange Rates | P1 |
| **Module B** | Corporate card auto-import and reconciliation | P1 |
| **Module C** | P-01 Policy Engine (5 rules, extensible ruleset) | P0 |
| **Module C** | Dynamic multi-level approval routing (4 tiers) | P0 |
| **Module C** | Exception approval and override workflow | P0 |
| **Module D** | Reimbursement batch processing (every 2 hours) | P0 |
| **Module D** | ACH/wire payment via corporate bank API | P0 |
| **Module D** | Payment failure handling (Finance Auditor alert within 15 min) | P1 |
| **Module E** | CFO Executive Dashboard | P1 |
| **Module E** | Finance Audit Lead Operational Dashboard | P1 |

### 5.2 Integrations

| Integration | Direction | Protocol | In-Scope |
|---|---|---|---|
| HRIS (Workday) | Inbound | REST + SCIM Webhooks | ✅ Yes |
| ERP (NetSuite) | Bi-directional | REST (OAuth 2.0) | ✅ Yes |
| Corporate Bank API (J.P. Morgan / Stripe Connect) | Inbound + Outbound | Webhook + REST | ✅ Yes |
| OCR Engine (Google Document AI or Abbyy) | Outbound | REST API | ✅ Yes |
| Open Exchange Rates | Outbound | REST API | ✅ Yes |
| Corporate Card (Amex API) | Inbound | REST API | ✅ Yes |
| SSO/MFA (Azure AD / Okta) | Inbound | SAML 2.0 + SCIM | ✅ Yes |
| Notification (Email + Slack nudges) | Outbound | SMTP + Slack API | ✅ Yes |

### 5.3 Data & Analytics Infrastructure

- Telemetry pipeline for all 5 KPIs (data ingestion, aggregation, and dashboard serving)
- KPI telemetry tables: `expense_report`, `reimbursement_transaction`, `finance_ledger`, `policy_violation`, `exception_approval`, `survey_response`, `audit_log`, `ocr_job_log`, `api_integration_log`
- SCD Type 2 implementation for `policy_rule` and employee hierarchy tables
- Dead-letter queue (DLQ) monitoring for all async integrations
- Immutable audit log implementation (SOC 2 compliance)

### 5.4 Mobile Application

- iOS and Android native applications
- Offline receipt capture with sync-on-connect capability
- Push notifications for approval status and reimbursement confirmation

### 5.5 Infrastructure & DevOps

- Kubernetes deployment: min 6 pods, auto-scale to 30
- Database read replica configuration for analytics isolation
- API gateway with rate limiting and RBAC enforcement
- Monitoring and alerting stack (all KPI alert thresholds wired to PagerDuty or equivalent)

---

## 6. Out-of-Scope Items

The following are **explicitly excluded** from the current project scope:

| Item | Reason / Notes |
|---|---|
| Corporate travel booking engine (flight/hotel search) | Travel Desk uses existing booking tools; system only receives booking confirmations |
| Payroll system integration | Reimbursements go via bank API, not through payroll |
| Legacy expense data migration (historical reports) | Historical data stays in existing systems; new system for net-new submissions only |
| Custom ERP connectors beyond NetSuite | Only NetSuite in scope; other ERPs deferred to Phase 2 |
| AI-based anomaly detection / fraud ML model | Rule-based P-01 engine in scope; ML fraud detection is Phase 2 |
| Self-serve policy configuration UI for admins | Initial policy rules hardcoded / config-driven; UI editor deferred |
| Supplier/vendor portal access | Internal employee-facing only |
| Cryptocurrency or digital wallet reimbursements | ACH/wire only; crypto deferred indefinitely |
| Tax computation / withholding engine | Tax coding provided to NetSuite; withholding computations remain in ERP |

---

## 7. Stopping Points / Definition of Done

Each stopping point below defines a clear acceptance condition. Work does **not** progress to the next phase until acceptance criteria are met and signed off by the relevant stakeholder.

---

### Stopping Point 1 — Architecture & Integration Sign-Off

**Condition:** Engineering architecture review approved; integration specs signed off by ERP, HRIS, and bank API vendors.

**Acceptance Criteria:**
- [ ] System architecture diagram reviewed and approved by Tech Lead + Security
- [ ] Workday SCIM webhook endpoint tested in sandbox (< 0.1% failure rate confirmed)
- [ ] NetSuite OAuth 2.0 connection established; test batch posting succeeds
- [ ] Bank API (J.P. Morgan/Stripe) test ACH transaction completed end-to-end
- [ ] SAML 2.0 SSO working with Azure AD/Okta (test user provisioned + deprovisioned via SCIM)

**Gate:** CTO + VP Engineering approval required to proceed to Module development.

---

### Stopping Point 2 — Core Module Completion (P0 Features)

**Condition:** All P0-priority modules fully developed, unit-tested, and code-reviewed.

**Acceptance Criteria:**
- [ ] Travel request submission flow functional with HRIS pre-population
- [ ] P-01 Policy Engine evaluates all 5 rules; hard blocks prevent submission; soft flags route to approvers
- [ ] Policy engine p99 latency < 350 ms under simulated load (500 submissions/minute)
- [ ] Multi-level approval routing correctly resolves all 4 expense tiers
- [ ] 48-hour escalation logic triggers correctly in test environment
- [ ] Reimbursement batch processes and calls NetSuite + bank API without error
- [ ] Mobile OCR: > 97% merchant accuracy, > 99% amount accuracy on test dataset of 500 receipts

**Gate:** QA sign-off on all P0 test cases with < 1% defect escape rate.

---

### Stopping Point 3 — UAT (User Acceptance Testing) Completion

**Condition:** UAT conducted with real users (Traveling Employees, Managers, Finance Auditors) in staging environment.

**Acceptance Criteria:**
- [ ] At least 50 real expense reports submitted, approved, and reimbursed in staging
- [ ] Finance Auditor confirms exception queue is operational and usable
- [ ] eNPS survey trigger fires within 3 days of mock reimbursement in test
- [ ] CFO and Audit Lead dashboards display accurate data (validated against source tables)
- [ ] Zero P0/P1 bugs open at UAT exit
- [ ] Security penetration test completed; all Critical/High findings resolved

**Gate:** Finance VP + CISO approval required to proceed to production deployment.

---

### Stopping Point 4 — Go-Live Readiness

**Condition:** Production environment deployed and smoke-tested; go-live checklist complete.

**Acceptance Criteria:**
- [ ] All integrations (Workday, NetSuite, Bank API, OCR, Amex) live and health-checked in production
- [ ] Kubernetes cluster running with min 6 pods; auto-scale tested to 30 pods under load
- [ ] Monitoring and alerting configured for all KPI thresholds (PagerDuty or equivalent)
- [ ] Dead-letter queue monitors active for all async integration paths
- [ ] Rollback plan documented and tested
- [ ] Data retention and GDPR erasure process verified
- [ ] Immutable audit logs (SOC 2) confirmed operational
- [ ] Employee communication and training materials distributed

**Gate:** CEO/CFO/CTO sign-off on go-live approval.

---

### Stopping Point 5 — KPI Target Validation (90 Days Post-Go-Live)

**Condition:** System has been live for 90 days; KPI targets measured against actuals.

**Acceptance Criteria (all must be met):**

| KPI | Target | Measurement |
|---|---|---|
| Expense Processing Cycle Time | ≤ 4 days (median) | System timestamp delta report |
| Policy Compliance Rate | ≥ 95% | Policy engine violation logs |
| Manual Audit Overhead | ≤ 20 hours/week | Timesheet + audit log |
| Employee eNPS (T&E domain) | ≥ +65 | Post-reimbursement survey |
| System Adoption Rate | ≥ 85% of eligible employees | HRIS vs. submission count |
| Duplicate Reimbursement Rate | < 0.5% | Finance reconciliation report |

**Gate:** If any KPI is not met, a remediation sprint is triggered (not a project stop). Full project closure requires all 6 KPIs validated.

---

## 8. Risks & Dependencies

### Key Dependencies

| Dependency | Owner | Risk if Delayed |
|---|---|---|
| Workday SCIM webhook configuration | HRIS team | Blocks user provisioning and HRIS sync |
| NetSuite OAuth credentials & sandbox access | Finance/IT | Blocks ERP posting and reimbursement pipeline |
| Bank API onboarding (J.P. Morgan/Stripe) | Finance/Treasury | Blocks all reimbursement flows |
| OCR vendor contract (Google/Abbyy) | Procurement | Blocks mobile receipt capture |
| Azure AD / Okta SSO configuration | IT Security | Blocks all user authentication |

### Key Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| OCR accuracy below 99% for complex receipts | Medium | High | Fallback to manual entry; OCR vendor SLA negotiation |
| Bank API latency > 45 min p95 under load | Low | High | Async polling with alerting; fallback to manual batch trigger |
| Policy engine scalability at Monday peak | Medium | High | Load test at 2,500 submissions/minute pre-launch |
| Low employee adoption (<85%) | Medium | High | Change management plan; manager champion program |
| GDPR erasure conflicts with 7-year audit retention | Low | High | Legal sign-off; pseudonymization strategy for erasure |
| NetSuite batch posting failures during peak | Low | Medium | DLQ with automatic retry; Finance Auditor alert on failure |

---

*This document is a living specification. Any change to business requirements, KPI definitions, or integration specs must be reflected here with a versioned update before engineering implementation proceeds.*

---

**Document Owner:** Product Management
**Approvers:** CFO, CTO, VP Engineering, CISO
**Review Cycle:** Bi-weekly during active development; monthly post-go-live
