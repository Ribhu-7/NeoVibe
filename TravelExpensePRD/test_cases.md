# Enterprise Employee Travel & Expense Management System
## Test Cases Specification Document

**Version:** 1.0  
**Status:** Ready for QA Review  
**Based on:** TravelExpensePRD v1.0, TravelExpenseKPI v1.0, and project_scope.md  
**Date:** 2026-06-05  

---

## 1. Functional Test Cases

### Module A: Travel Request & Pre-Approval Workflow

| Test Case ID | Priority | Module / Feature | Objective | Preconditions | Test Steps | Expected Result |
|---|---|---|---|---|---|---|
| **TC-A1-01** | P0 | Travel Request | Submit valid travel request (Economy class, Domestic, under budget) | User is logged in as traveling employee. Profile synced from HRIS. | 1. Navigate to "New Travel Request".<br>2. Fill in dates (within next week), destination (domestic), purpose: "Client meeting".<br>3. Enter estimated costs: flight $300, hotel $150, transport $50, meals $100.<br>4. Click "Submit". | 1. Status changes to `PENDING_MANAGER_APPROVAL`.<br>2. Policy engine returns zero violations.<br>3. Manager receives email & in-app notifications. |
| **TC-A1-02** | P0 | Travel Request | Enforce VP approval for Business Class flights on international routes >7 days (Rule R-TR-01) | User profile is configured with VP escalation path. | 1. Create Travel Request.<br>2. Destination: London (International). Duration: 8 days.<br>3. Flight Class: Business Class.<br>4. Click "Submit". | 1. Policy warning triggers indicating VP approval is required.<br>2. Status updates to `PENDING_VP_APPROVAL` (escalated routing). |
| **TC-A1-03** | P1 | Travel Request | Validate free-text justification requirement for estimated totals > $10,000 | User creates travel request. | 1. Enter estimated costs totaling $10,500.<br>2. Leave justification empty.<br>3. Click "Submit".<br>4. Fill in justification (<50 chars). Click "Submit".<br>5. Fill in justification (55 chars). Click "Submit". | 1. Steps 3 & 4 block submission with validation error: "Justification must be at least 50 characters."<br>2. Step 5 submits successfully. |
| **TC-A1-04** | P1 | Travel Request | Request cash advance within limits (max 80%, capped at $5,000) | User has active travel profile. | 1. Total estimated cost: $4,000.<br>2. Request cash advance: $3,500 (87.5%). Submit.<br>3. Change cash advance request: $3,200 (80%). Submit.<br>4. Total estimated cost: $8,000. Request cash advance: $6,000 (75%, but > $5,000 cap). Submit. | 1. Step 2 fails (exceeds 80% limit).<br>2. Step 3 succeeds (routes to Manager).<br>3. Step 4 fails (exceeds $5,000 cap). |
| **TC-A2-01** | P0 | Pre-Approval | Manager actions: Approve, Reject, Request Changes | Travel request is pending manager review. | 1. Log in as Manager. Open pending request.<br>2. Verify visibility of cost details, remaining department budget, and compliance flags.<br>3. Action: Approve.<br>4. Repeat for different request with Action: Reject (without reason).<br>5. Repeat with Action: Reject (with reason). | 1. Step 3 status changes to `APPROVED_FOR_BOOKING`. Travel Desk is notified.<br>2. Step 4 is blocked (rejection reason required).<br>3. Step 5 status changes to `REJECTED`, employee receives notification. |
| **TC-A2-02** | P1 | Pre-Approval | Escalation of manager SLA violation (48-hour SLA breach) | Travel request status is `PENDING_MANAGER_APPROVAL`. | 1. Set travel request creation time to `CURRENT_TIME - 49 hours` in test environment.<br>2. Trigger background scheduler daemon. | 1. Request routes to manager's manager.<br>2. Original manager and new approver receive escalation alerts. |

---

### Module B: Real-Time Expense Tracking & Automated Capture

| Test Case ID | Priority | Module / Feature | Objective | Preconditions | Test Steps | Expected Result |
|---|---|---|---|---|---|---|
| **TC-B1-01** | P0 | Mobile OCR | Extract receipt details with zero manual input | Mobile app logged in; camera operational. | 1. Photograph a standard thermal restaurant receipt ($85.50, USD, McDonald's, Date: 2026-06-04).<br>2. Trigger OCR engine execution. | 1. OCR extracts: Merchant = McDonald's, Date = 2026-06-04, Amount = $85.50, Currency = USD.<br>2. Category maps to "Meals".<br>3. Receipt image attaches to expense item. |
| **TC-B1-02** | P0 | Expense Engine | Duplicate receipt hash detection | A receipt is already submitted in database. | 1. Upload a receipt identical to the previously submitted receipt (same merchant, date, and amount ± $0.50).<br>2. Attempt to submit. | 1. System blocks submission.<br>2. Error displayed: "Duplicate receipt detected." |
| **TC-B2-01** | P1 | Multi-Currency | Conversion using daily exchange rates | Network access to currency exchange feed. | 1. Create an expense entry for EUR 150.<br>2. Transaction Date: 2026-06-04.<br>3. Daily Rate on 2026-06-04 was 1.08 USD/EUR. | 1. System calculates and stores USD equivalent: $162.00 USD.<br>2. Both values (`original_amount/currency` and `usd_amount`) saved in database. |
| **TC-B2-02** | P1 | Multi-Currency | Flag >10% exchange rate fluctuation | Daily exchange rate feed mock configured. | 1. Input an exchange rate that is 12% higher than prior day’s exchange rate.<br>2. Submit expense. | 1. Expense is accepted but flagged `EXCHANGE_RATE_EXCEPTION` for Auditor review. |

---

### Module C: Policy Validation & Approval Routing

| Test Case ID | Priority | Module / Feature | Objective | Preconditions | Test Steps | Expected Result |
|---|---|---|---|---|---|---|
| **TC-C1-01** | P0 | Policy Engine | Enforce P-01 policy rules at submission | P-01 Policy engine active. | 1. Submit meal expense of $90 in Tier 1 city (Rule P-01-01 limit is $75).<br>2. Submit domestic flight booked in Business Class (Rule P-01-04).<br>3. Submit expense with transaction date 65 days past trip end date (Rule P-01-05). | 1. Case 1 triggers soft flag: requires manager justification.<br>2. Case 2 triggers auto-rejection: submission blocked.<br>3. Case 3 triggers hard block: submission blocked (expired). |
| **TC-C2-01** | P0 | Approval Routing | Tier-based manager routing validation | Workflow matrix configured as per PRD. | 1. Submit expense of $450 (Tier 1).<br>2. Submit expense of $1,200 (Tier 2).<br>3. Submit expense of $5,000 (Tier 3).<br>4. Submit expense of $12,000 (Tier 4). | 1. $450 routes to: Direct Manager only.<br>2. $1,200 routes to: Direct Manager → Dept Head.<br>3. $5,000 routes to: Direct Manager → Dept Head → Finance Controller.<br>4. $12,000 routes to: Direct Manager → Dept Head → Finance Controller → VP of Finance. |
| **TC-C2-02** | P1 | Approval Routing | Parallel routing for cross-charged expenses | User profile mapped to project codes spanning different cost centers. | 1. Submit expense report cross-charged to Cost Center A and Cost Center B.<br>2. Assign to workflow. | 1. Parallel approval tasks generated for both Cost Center A and B managers.<br>2. Both approvals required to proceed. |
| **TC-C3-01** | P0 | Exception Auditing | Finance Auditor review and policy override | Expense report flagged with P-01 policy violations. | 1. Log in as Finance Auditor.<br>2. Open flagged report. Review comments.<br>3. Approve with override reason "Client requirement".<br>4. Verify workflow update. | 1. Action allowed only if override reason is selected from dropdown.<br>2. Report moves to settlement batch queue.<br>3. Audit trail records auditor ID, action, timestamp, and justification. |

---

### Module D: Settlement, Banking & Reimbursement Pipelines

| Test Case ID | Priority | Module / Feature | Objective | Preconditions | Test Steps | Expected Result |
|---|---|---|---|---|---|---|
| **TC-D1-01** | P0 | ERP Integration | Automated export to ERP NetSuite | Report status is `APPROVED` (passed audit/manager). | 1. Trigger automated 2-hour integration batch scheduler.<br>2. Verify payload payload mapping. | 1. REST client makes connection using OAuth 2.0.<br>2. NetSuite returns posting confirmation ID.<br>3. Report status in T&E system updates to `ERP_POSTED`. |
| **TC-D2-01** | P0 | Bank Integration | Direct payment execution via bank API | ERP AP liability successfully posted. | 1. Trigger reimbursement instructions to JP Morgan/Stripe Connect API.<br>2. Simulate bank success webhook payload callback. | 1. Payment settled.<br>2. Status updates to `REIMBURSED`. |
| **TC-D2-02** | P1 | Bank Integration | Handle transaction settlement failure | Bank settlement API returns failure status. | 1. Trigger mock payment callback indicating "Invalid routing/bank account". | 1. Status marked as `PAYMENT_EXCEPTION`.<br>2. Incident flagged in Finance Auditor queue within 15 minutes. |

---

## 2. KPI & Telemetry Validation Test Cases

These test cases ensure the underlying data schema, aggregations, filters, and pipeline rules measure KPIs accurately.

| Test Case ID | Priority | Target KPI | Verification Objective | Steps & Input Data | Expected Analytical Result |
|---|---|---|---|---|---|
| **TC-KPI-01** | P1 | **KPI 1**: Expense Processing Cycle Time | Validate data aggregation filters and exclusions | Insert mock database logs:<br>- Report A: `submitted_at`: 2026-06-01, `settled_at`: 2026-06-03, `is_test` = FALSE, ACH (Valid)<br>- Report B: `submitted_at`: 2026-06-01, `settled_at`: 2026-06-03, `is_test` = TRUE (Test data)<br>- Report C: `submitted_at`: 2026-06-01, `settled_at`: 2026-06-03, `advanced_cash_flag` = TRUE<br>- Report D: `submitted_at`: 2026-06-01, `settled_at`: 2026-06-03, method: "Manual Check" | 1. Query analytical view for Median Cycle Time.<br>2. Expected: Report A is **included** (Cycle Time = 2 days).<br>3. Reports B, C, and D are **excluded** from the aggregation. |
| **TC-KPI-02** | P1 | **KPI 2**: Policy Compliance Rate | Ensure correct classification of overridden vs non-compliant | Create line items:<br>- Line 1: `violation_flag` = FALSE<br>- Line 2: `violation_flag` = TRUE, override allowed by policy (`POLICY_WAIVER`) & approved<br>- Line 3: `violation_flag` = TRUE, override reason = `SYSTEM_ERROR` (Not allowed policy code) | 1. Execute SQL metrics engine.<br>2. Expected compliance classification:<br>  - Line 1: COMPLIANT<br>  - Line 2: COMPLIANT<br>  - Line 3: NON_COMPLIANT<br>3. Overall compliance calculation reflects exact rules. |
| **TC-KPI-03** | P2 | **KPI 3**: Manual Auditing Overhead | Clamp duration outliers in inferred logs | Insert audit logs into `audit_log` table:<br>- Log A: duration 15 seconds (Under threshold)<br>- Log B: duration 20 minutes (Standard)<br>- Log C: duration 120 minutes (Outlier) | 1. Run time evaluation parser script.<br>2. Expected values output:<br>  - Log A is clamped to **30 seconds minimum**.<br>  - Log B remains **20 minutes**.<br>  - Log C is clamped to **60 minutes maximum**. |
| **TC-KPI-04** | P2 | **KPI 4**: employee NPS | Invalidate survey responses on memory decay | Insert survey responses:<br>- Survey A: submitted 2 days after reimbursement settled.<br>- Survey B: submitted 8 days after reimbursement settled. | 1. Query KPI 4 metric generator.<br>2. Survey A is **included** in calculation.<br>3. Survey B is **excluded** (due to memory decay limit > 7 days). |
| **TC-KPI-05** | P2 | **KPI 5**: System Adoption | Validate active traveled employee denominator logic | Setup users:<br>- User 1: Eligible, Travel Freq = HIGH, login = true, submitted = true<br>- User 2: On leave of absence (`leave_of_absence_flag` = TRUE)<br>- User 3: Eligible, Freq = LOW, no corporate card charges. | 1. Run adoption processor batch job.<br>2. User 1 is included in both numerator and denominator.<br>3. User 2 is **excluded** entirely (on leave).<br>4. User 3 is **excluded** from denominator (no expense need). |

---

## 3. Non-Functional & System Performance Test Cases

### Performance & Latency

| Test Case ID | Priority | Target SLA / System | Objective / Test Scenario | Verification Method & Action | Expected Result |
|---|---|---|---|---|---|
| **TC-NFR-01** | P0 | Policy Engine Latency | Verify p99 evaluation latency under peak load | Generate mock traffic of 2,500 submissions per minute (peak Monday load). Measure telemetry database. | 1. Average evaluation time < 150 ms.<br>2. p99 evaluation latency remains below **350 ms**.<br>3. No API request drops. |
| **TC-NFR-02** | P1 | System Latency Guardrail | Asynchronous demotion of slow reports | Execute policy execution rule payload containing complex recursive validations (forces processing >2 seconds). | 1. Gateway demotes request to asynchronous hold within 2 seconds.<br>2. User receives success receipt with "Processing in background" status.<br>3. Operational flow not blocked. |
| **TC-NFR-03** | P1 | OCR Performance | Validate extraction limits and accuracies | Upload a batch of 1,000 scanned receipt images of varying quality (low resolution, folds, faded ink). | 1. p95 OCR conversion time < 1,200 ms.<br>2. Amount field accuracy matches or exceeds **99.0%**.<br>3. Merchant field accuracy matches or exceeds **97.0%**. |
| **TC-NFR-04** | P1 | Scalability / Load | Kubernetes Horizontal Pod Autoscaler (HPA) validation | Ramp up concurrent user sessions from 100 to 3,000 within 5 minutes. | 1. CPU/Memory usage triggers pod creation.<br>2. Kubernetes cluster scales smoothly from 6 pods up to 30 pods.<br>3. Zero downtime or request timeout errors observed by client devices. |

### Security & Compliance

| Test Case ID | Priority | Compliance Domain | Objective / Test Scenario | Verification Method & Action | Expected Result |
|---|---|---|---|---|---|
| **TC-SEC-01** | P0 | Access Control | Role-Based Access Control (RBAC) enforcement | Attempt to call Auditor API endpoint (`GET /audit/exceptions`) using Traveling Employee JWT token. | 1. Gateway blocks request with HTTP 403 Forbidden.<br>2. Action is blocked at both router layer and service interface. |
| **TC-SEC-02** | P1 | Authentication | SCIM Auto-provisioning & SSO enforcement | 1. Deprovision user in Okta directory simulation.<br>2. Attempt to login using the deactivated account via SAML. | 1. SCIM webhook triggers immediate termination of user's active session.<br>2. Authentication is rejected; user profile marked `inactive` in db. |
| **TC-SEC-03** | P1 | Privacy (GDPR) | Right to Erasure enforcement | Trigger "Right to Erasure" request for deactivated Employee ID. | 1. All PII and personal documents deleted from database tables within 30 days.<br>2. Financial transactions and audit logs are retained but fully anonymized (7-year tax audit retention). |
| **TC-SEC-04** | P0 | Security | SOC 2 Immutable Audit Trail validation | Attempt to modify or delete a row in the `ApprovalAudit` table using a high-privilege DB administrator credential. | 1. Database triggers block UPDATE or DELETE operations on transactional log table.<br>2. Any update bypass attempt triggers alert to security monitoring system. |

---

*This test specification document should be configured in the CI/CD automated test frameworks (e.g. Cypress, Playwright, Jest, Postman API runner) and manual execution matrices.*
