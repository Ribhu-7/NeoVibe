# Enterprise Employee Travel & Expense Management System – KPI Blueprint & Metrics Dictionary

## Version 1.0 – Production Implementation Ready

This document defines the technical mapping from business objectives to data engineering schemas, telemetry pipelines, and executive visualizations for the **Enterprise Employee Travel & Expense Management System**. All metrics are aligned with the target state: fully automated, exception-driven, integrated with Workday, NetSuite, and corporate banking APIs.

---

## 1. Core KPI Specification Sheets

Each KPI sheet below provides:

- **Business Definition & Organizational Impact** – what it measures and why it matters.
- **Exact Mathematical Formula** – in LaTeX notation, referencing field-level data.
- **Telemetry Triggers & Source Tables** – database entities, timestamps, and action records.
- **Data Aggregation Filters** – explicit rules to exclude edge cases and ensure clean baselines.

---

### KPI 1: Expense Processing Cycle Time

| Attribute | Specification |
|-----------|----------------|
| **Business Definition** | The median elapsed calendar time from an employee’s submission of an expense report to the final reimbursement posting in the corporate bank account (or accounting system liability offset). Impact: Directly correlates with employee satisfaction and working capital float. |
| **Mathematical Formula** | $$\text{Median Cycle Time} = \underset{i=1}^{N}{\text{median}} \left( T_{\text{reimbursed}, i} - T_{\text{submitted}, i} \right)$$ <br> Where $T_{\text{submitted}}$ = `expense_report.submitted_at` (UTC timestamp) and $T_{\text{reimbursed}}$ = first non-null value of either `reimbursement_transaction.settled_at` (bank confirmation) OR `finance_ledger.posted_at` (NetSuite liability cleared). |
| **Telemetry Triggers & Source Tables** | - **Trigger 1**: `expense_report` row state changes from `DRAFT` to `SUBMITTED` – capture `submitted_at`.<br>- **Trigger 2**: `reimbursement_transaction` status becomes `SETTLED` via bank API callback – capture `settled_at`.<br>- **Fallback**: `finance_ledger` entry with `transaction_type = 'REIMBURSEMENT'` and `ledger_status = 'POSTED'` – capture `posted_at`.<br>- **Join key**: `expense_report.report_id` → `reimbursement_transaction.report_id` → `finance_ledger.report_id`. |
| **Data Aggregation Filters** | **Include only**:<br> - Reports with `expense_report.is_test = FALSE`<br> - `expense_report.submission_channel IN ('WEB', 'MOBILE_APP', 'API')`<br> - `expense_report.currency = 'USD'` (or base functional currency) to avoid FX clearing delays<br> - `reimbursement_transaction.method = 'ACH'` (exclude manual checks)<br>**Exclude**:<br> - Reports with any line item `expense_line_item.advanced_cash_flag = TRUE` (cash advances settle on separate cycles)<br> - Reports flagged `expense_report.multi_month_travel = TRUE` (trip spanning >90 days)<br> - Reports with `expense_report.audit_hold_reason IN ('FRAUD_INVESTIGATION', 'TAX_CORRECTION')`<br> - Any report where `reimbursement_transaction.settled_at - expense_report.submitted_at > 180` days (systemic outlier) |

---

### KPI 2: Policy Compliance Rate

| Attribute | Specification |
|-----------|----------------|
| **Business Definition** | Percentage of total expense report line items (or total submitted amount) that fully adhere to all configured travel & expense policies before any manual override or exception approval. Impact: Measures leakage reduction target (from 12-15% to <5%). |
| **Mathematical Formula** | Two variants. **Primary (line item count)**:<br> $$\text{ComplianceRate}_{items} = \frac{\sum \mathbf{1}_{\text{policy\_violation\_flag = FALSE}}}{\text{Total submitted line items}} \times 100$$<br>**Secondary (amount weighted)**:<br> $$\text{ComplianceRate}_{amount} = \frac{\sum (\text{line\_amount} \times \mathbf{1}_{\text{violation=FALSE}})}{\sum \text{line\_amount}} \times 100$$<br>Where `policy_violation_flag` is computed by the **P-01 Policy Engine** at submission time. |
| **Telemetry Triggers & Source Tables** | - **Trigger**: Upon `expense_report.submitted_at`, the P-01 engine writes one row per line item to `policy_violation` table.<br>- **Source tables**:<br>  - `expense_line_item` (line_id, report_id, amount, category)<br>  - `policy_violation` (line_id, violation_flag, violation_code, is_overridden, override_reason)<br>  - `exception_approval` (line_id, approver_id, approved_at, override_ttl)<br>- **Join**: `expense_line_item.line_id = policy_violation.line_id LEFT JOIN exception_approval.line_id` |
| **Data Aggregation Filters** | **Include**:<br> - All submitted reports with `submitted_at >= CURRENT_DATE - 30 days` (rolling monthly) or as defined by reporting period.<br> - All line items regardless of `is_test` in lower environments – but production only.<br> - Line items with `status != 'CANCELLED'`.<br>**Exclude**:<br> - Line items where `exception_approval.override_reason = 'EXECUTIVE_APPROVAL'`? **No** – compliance rate without overrides: use `violation_flag = TRUE AND exception_approval.approved_at IS NULL` as true non-compliance.<br> - Specifically, **override does not fix compliance** – we count original violation unless policy says “override = compliant”. Business rule: compliance rate = % of line items that never triggered a violation. Therefore filter: ignore rows where `exception_approval.approved_at IS NOT NULL` for the purpose of counting compliant? Actually to measure **policy adherence**, count line items with `violation_flag = FALSE` OR (`violation_flag = TRUE` AND `exception_approval.approved_at IS NOT NULL`)? Standard practice: compliance rate excludes overridden violations. We define: `compliant_final = violation_flag = FALSE OR (violation_flag = TRUE AND overridden = TRUE AND override_policy_code = 'ALLOWED_BY_MANAGER')`. Explicit filter: exclude overrides that are not allowed by policy (override_reason = 'SYSTEM_ERROR' should not count as compliant). We'll implement as:<br> ```sql<br> CASE <br>   WHEN pv.violation_flag = FALSE THEN 'COMPLIANT'<br>   WHEN pv.violation_flag = TRUE AND ea.approved_at IS NOT NULL AND ea.override_type IN ('POLICY_WAIVER', 'CORRECTED_RECEIPT') THEN 'COMPLIANT'<br>   ELSE 'NON_COMPLIANT'<br> END<br>```<br> - Additionally exclude line items where `expense_report.is_sample_data = TRUE`. |

---

### KPI 3: Manual Auditing Overhead

| Attribute | Specification |
|-----------|----------------|
| **Business Definition** | Total weekly finance team hours spent on manual review, validation, correction, and follow-up of expense reports, normalized per 1,000 submitted reports. Target reduction from 120 hours/week to <25 hours/week baseline. |
| **Mathematical Formula** | $$\text{Overhead}_{weekly} = \sum_{u \in \text{FinanceAuditors}} \left( \text{hours\_logged}_{u, \text{ExpenseTasks}} \right)$$<br>Where `hours_logged` comes from time-tracking system with task tags `['EXPENSE_AUDIT', 'MANUAL_RECEIPT_OCR_CORRECTION', 'POLICY_ESCALATION_HANDLE', 'APPROVER_CHASE']`.<br><br>**Normalized**: $\text{Overhead}_{per1k} = \frac{\text{Overhead}_{weekly}}{\text{SubmittedReports}_{weekly}} \times 1000$ |
| **Telemetry Triggers & Source Tables** | - **Trigger 1**: Time entry system (e.g., Jira Tempo, or internal `finance_time_log`) with `task_category = 'EXPENSE_MANUAL'` – capture `user_id`, `duration_minutes`, `report_id` (if applicable).<br>- **Trigger 2**: Application audit log events: `audit_log.action IN ('MANUAL_REVIEW_START', 'MANUAL_REVIEW_END', 'RECEIPT_DATA_CORRECTION', 'MANUAL_APPROVAL_OVERRIDE')` – each event has `performed_by` (Finance role). Infer duration from `start/end` pairs.<br>- **Source tables**: `time_tracking.entries`, `audit_log`, `expense_report.audit_trail`.<br>- **Join**: Optional `report_id` to attribute hours to specific report characteristics. |
| **Data Aggregation Filters** | **Include**:<br> - Time entries where `date` falls within Monday 00:00 to Sunday 23:59 of the reporting week.<br> - Only users with `role = 'FINANCE_AUDITOR'` OR `role = 'EXPENSE_ADMIN'`.<br> - For inferred audit log durations: minimum 30 seconds, maximum 60 minutes per report (clamp outliers).<br>**Exclude**:<br> - Time spent on system maintenance, policy configuration, or user training (task tag `'PROJECT_OVERHEAD'`).<br> - Automated actions (e.g., `system` user in audit log).<br> - Time entries lacking any expense-related tag even if user is auditor. |

---

### KPI 4: Employee Net Promoter Score (eNPS) – Expense System Domain

| Attribute | Specification |
|-----------|----------------|
| **Business Definition** | A survey-based measure of employee willingness to recommend the expense management process to colleagues. Captured post-reimbursement. Scale 0-10, classified as Detractors (0-6), Passives (7-8), Promoters (9-10). Impact: Drives adoption and reduces shadow finance processes. |
| **Mathematical Formula** | $$\text{eNPS} = \frac{\#\text{Promoters} - \#\text{Detractors}}{\#\text{Total Respondents}} \times 100$$<br>Respondents are employees who submitted at least one report in the trailing 30 days and received a survey. |
| **Telemetry Triggers & Source Tables** | - **Trigger**: After `reimbursement_transaction.settled_at` event, an async job inserts a row into `survey_request` with `expires_at = settled_at + 3 days`.<br>- **Source tables**:<br>  - `user_profile` (user_id, department, tenure)<br>  - `survey_response` (response_id, user_id, score, comment, submitted_at)<br>  - `expense_report` (report_id, user_id, submitted_at)<br>- **Join**: `survey_response.user_id` → `expense_report.user_id` → ensure user has report in last 30d. |
| **Data Aggregation Filters** | **Include**:<br> - Responses where `survey_response.submitted_at <= survey_request.expires_at`.<br> - Only first response per `user_id` per rolling 90 days (avoid polling bias).<br> - Surveys sent for reports with `expense_report.total_amount >= $10` (micro-expenses excluded).<br>**Exclude**:<br> - Test users (`user_profile.is_employee = FALSE`).<br> - Contractors with less than 3 months tenure (low engagement).<br> - Responses with `survey_response.submitted_at` more than 7 days after reimbursement (memory decay). |

---

### KPI 5: System Adoption Rate

| Attribute | Specification |
|-----------|----------------|
| **Business Definition** | Percentage of active employees (with any travel or expense need in a given month) who submit at least one expense report through the automated system, compared to those still using offline/manual or exception channels. |
| **Mathematical Formula** | $$\text{Adoption} = \frac{\left|\{u \in \text{ActiveUsers} : \exists r \in \text{ExpenseReport} \text{ with } r.user\_id = u \text{ and } r.submitted\_at \in \text{month}\}\right|}{\left|\{u \in \text{ActiveUsers} : \text{had expense need in month}\}\right|} \times 100$$<br>“Expense need” determined via HRIS job role travel frequency or corporate card swipe activity outside system. |
| **Telemetry Triggers & Source Tables** | - **Trigger**: Monthly batch job on day 1 of next month.<br>- **Source tables**:<br>  - `user_profile` (user_id, is_active, employment_status, travel_frequency_code)<br>  - `expense_report` (user_id, submitted_at, source_channel)<br>  - `corporate_card_transaction` (user_id, transaction_date, is_auto_imported_flag) – for identifying off-system spend.<br>  - `hris_import_log` (hris_user_id, last_sync_at, expense_eligible_flag)<br>- **Denominator logic**: Users with `user_profile.expense_eligible = TRUE` AND (`travel_frequency_code IN ('HIGH','MEDIUM')` OR `corporate_card_transaction` exists with `transaction_date` in month AND `is_auto_imported_flag = FALSE` indicating manual card feed fallback). |
| **Data Aggregation Filters** | **Include**:<br> - Users with `employment_status = 'ACTIVE'` and `hire_date <= first_day_of_month`.<br> - Expense reports with `source_channel IN ('WEB', 'MOBILE_APP', 'EMAIL_PARSER')` – exclude batch imports from legacy systems.<br>**Exclude**:<br> - Users on long-term leave (`leave_of_absence_flag = TRUE`).<br> - C-suite executives with administrative assistants submitting on their behalf (unless assistant uses system – count executive as adopted if any report tied to their ID).<br> - One-time project-based contractors with no repeat need. |

---

## 2. System Performance & Tech Stack Telemetry

Engineering-level metrics to ensure scalability for 10,000+ concurrent sessions (peak: Monday 9–11 AM local across global regions).

### 2.1 OCR Parsing Latency & Accuracy Engine

| Metric | Definition | Telemetry Source | Target & Alerting |
|--------|------------|------------------|-------------------|
| **p95 OCR Latency** | Time from receipt of receipt image (API POST `/receipt/upload`) to extraction of structured fields (merchant, date, amount, tax). Measured in milliseconds. | `ocr_job_log` table: `received_at` (from load balancer) → `extracted_at` (OCR engine completion). Exclude queue wait time. | < 1,200 ms p95. Alert > 2,000 ms for 5% of requests. |
| **OCR Field Accuracy** | For each extracted field, True Positive rate compared to manual correction within 24 hours. Formula: $$ \text{Accuracy}_{field} = \frac{\text{TP}}{\text{TP + FP}} $$ where TP = automated extraction matches final corrected value, FP = automated value differs and human overrides. | Join `ocr_extraction` (job_id, field_name, extracted_value) with `receipt_correction_log` (job_id, corrected_value, corrected_by). Correction log written only when user manually edits. | **Amount accuracy** ≥ 99.0%; **Merchant** ≥ 97%; **Date** ≥ 98%. Alert if falls below 95% over 1 hour. |
| **Manual Adjustment Rate** | Percentage of OCR-processed receipts that receive at least one human correction before submission. $$\frac{\#\text{receipts with any correction}}{\#\text{receipts processed by OCR}} \times 100$$ | `receipt_correction_log` grouped by `receipt_id`. | Target < 5%. Alert > 12% per rolling hour. |

**Source Tables:** `ocr_job_log`, `receipt_image_metadata`, `receipt_correction_log`, `expense_line_item` (final amount validation).

---

### 2.2 API Sync Health & Error Rates

| Integration | Metric | Calculation | Thresholds |
|-------------|--------|-------------|-------------|
| **HRIS (Workday) Webhook** | Webhook delivery failure rate | $\frac{\#\text{webhook delivery failures (HTTP 5xx or timeout)}}{\#\text{total webhook events from Workday}}$ over 15 min window. | < 0.1%. Alert at 2%. |
| **NetSuite Ledger Posting** | Batch posting error rate | $\frac{\#\text{expense reports where `ledger_post_status = 'FAILED'`}}{\#\text{reports sent to NetSuite in batch}}$ per hour. | < 0.5%. Alert at 3%. |
| **Corporate Card Banking API** | Token refresh failure & transaction import latency | - Token failures: count of HTTP 401/403 in /token endpoint.<br>- Import latency: p95 time from card swipe (bank timestamp) to visibility in system (bank_api.import_finished_at). | Token failures: 0 per day.<br>Latency p95 < 45 minutes. Alert > 2 hours. |

**Source Tables:** `api_integration_log` (endpoint, http_status, response_time_ms, retry_count), `ledger_posting_queue` (posting_id, status, error_message), `card_transaction_import` (transaction_id, bank_event_time, import_complete_time).

**Reconciliation:** Daily dead-letter queue (DLQ) monitor for any failed webhooks that exceed 3 retries.

---

### 2.3 P-01 Policy Engine Execution Latency

| Metric | Definition | Measurement | Peak Load Handling |
|--------|------------|-------------|--------------------|
| **p99 Policy Evaluation Latency** | Time from `expense_report.submitted_at` to completion of all policy rules against all line items. Includes rule fetch, condition evaluation, violation persistence. | `policy_engine_audit` table: `evaluation_start` (triggered by submission event) → `evaluation_end`. | Target < 350 ms p99. Under Monday morning peak (estimated 2,500 submissions/minute), scale horizontally. Alert > 800 ms. |
| **Rule Execution Throughput** | Number of line items evaluated per second aggregated across engine workers. | Sum(`line_items_evaluated`) / (max(`evaluation_end`) - min(`evaluation_start`)) per minute. | Maintain > 5,000 items/sec. |

**Source Tables:** `policy_engine_audit`, `policy_rule` (active rule count), `expense_line_item`.

**Derived Alert:** If policy evaluation latency exceeds 2 seconds for any report, automatically demote that report to “synchronous hold” and process asynchronously to not block user.

---

## 3. Executive Dashboard Design Schema

Modern corporate analytics layer (PowerBI / Tableau / custom Next.js + Highcharts). Data refresh: near real-time (latency ≤ 5 minutes) for operational views; daily aggregated for CFO view.

### 3.1 Chief Financial Officer (CFO) View

**Purpose:** Macro financial tracking, liability forecasting, realized savings from leakage reduction.

| Visual Component | Data Source (Table / Aggregation) | Refresh | Interactive Filters |
|------------------|-----------------------------------|---------|----------------------|
| **1. Realized Leakage Recovery (MTD & QTD)** | KPI: Policy non-compliance amount **before overrides** that was **rejected or recouped**. Calculation: `SUM(line_amount WHERE violation_flag=TRUE AND (rejected=TRUE OR employee_paid=TRUE))` over period. Line chart with prior period comparison. | Daily | Department, Region |
| **2. Outstanding Liability (Accrual)** | Sum of `expense_report.total_amount` where `reimbursement_transaction.settled_at IS NULL` and `expense_report.submitted_at <= current_date` AND `expense_report.payment_status != 'PAID'`. Show trend last 30 days + forecast next 30 using average cycle time. | Near real-time (5 min) | Currency (USD/EUR/GBP) |
| **3. Processing Cycle Time – p50 / p90** | Box plot or trend line using formula from KPI 1, broken by `expense_report.department_id` and `report_type` (travel vs. out-of-pocket). | Daily | Time range (last 3 months) |
| **4. Automation ROI** | Metric: $\frac{\text{Manual hours saved} \times \text{Avg finance hourly rate}}{\text{System operating cost}}$ per month. Manual hours saved = baseline (120h/week) – current overhead (KPI 3). Show as gauge chart. | Weekly (Thursdays) | None |
| **5. Compliance Rate – Amount Weighted** | KPI 2 secondary formula, presented as stacked bar: compliant, overridden, rejected (non-compliant and not overridden). Drill-down to top violating policy codes (e.g., `MEAL_EXCESSIVE`, `FARE_CLASS_NON_COMPLIANT`). | Daily | Cost center hierarchy |

**Annotations:** CFO dashboard includes a “Variance to Target” table for each core metric (Cycle Time target 7 days, Compliance target 95%, Overhead target 25h/week).

---

### 3.2 Operational / Finance Audit Lead View

**Purpose:** Queue management, bottleneck identification, manager SLA violations, real-time policy failure alerts.

| Visual Component | Data Source & Logic | Refresh | Drill-Down Action |
|------------------|----------------------|---------|-------------------|
| **1. Audit Queue Density** | Real-time count of `expense_report` where `status = 'AUDIT_REQUIRED'` AND `assigned_auditor_id IS NULL`. Histogram by `wait_time_minutes` since `audit_required_flag_set_at`. | Streaming (10s) | Click on bar to see list of report IDs and amounts. |
| **2. Manager SLA Violations** | % of reports where `approval_step.approver_role = 'MANAGER'` and `approval_step.completed_at - approval_step.assigned_at > 48 hours`. Bar chart by manager name (top 10 offenders). | Hourly | Send nudge button (integration with Slack/email). |
| **3. High-Frequency Policy Failure Alerts** | Real-time alert tile: any single `user_id` with ≥3 policy violations within 60 minutes, or any violation of `CRITICAL` policy code (e.g., `FRAUD_DUPLICATE_RECEIPT`). Display as scrolling list. | Streaming (5s) | Link to user profile & audit log. |
| **4. OCR Correction Heatmap** | Field-specific correction rate: `correction_count` per field type (`MERCHANT`, `AMOUNT`, `DATE`) per department. Use table heatmap coloring (red = high correction). | Hourly | Export to CSV for vendor feedback. |
| **5. API Sync Health Dashboard** | Three small line charts: HRIS webhook success %, NetSuite posting failure queue length, bank API latency p95. Show red line when above threshold. | 1 minute | Logs viewer modal for each integration. |
| **6. Operational Throughput** | Moving average of reports fully processed (submitted → reimbursed) per hour, overlaid with audit team online count. | 15 min | None – used for capacity planning. |

**Additional Audit Lead Component:** “Exception Approval Cycle Time” – median time from `exception_request.created_at` to `exception_approval.approved_or_denied_at`, grouped by approver role (Manager, Finance, Compliance Officer).

---

## 4. Data Engineering Implementation Notes

- **All timestamps** stored in UTC with microsecond precision. Timezone conversion applied only at presentation layer.
- **Sensitive data** (receipt images, personal identifiable information) not materialized in analytical tables; references via `receipt_id` with access controls.
- **Slowly Changing Dimensions (SCD) Type 2** for policy rules and employee hierarchies to ensure historical compliance calculations remain accurate.
- **Idempotency keys** on all ingestion pipelines (webhooks, OCR results) to prevent duplicate KPI increments.
- **Retention policy**: raw event logs 90 days; aggregated KPIs 7 years.

---

*This document is a living specification. Any change to business logic (e.g., policy rule definition) must trigger a versioned update to the relevant KPI formula and telemetry mapping.*