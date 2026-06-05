# Enterprise Employee Travel & Expense Management System
## User Personas Specification Document

**Version:** 1.0  
**Status:** Approved  
**Based on:** TravelExpensePRD v1.0, TravelExpenseKPI v1.0, and project_scope.md  
**Date:** 2026-06-05  

---

## 1. Traveling Employee: "Alex, the Frequent Flyer"
*“I just want to submit my receipts quickly so I can get reimbursed and focus on my actual job.”*

```
+-------------------------------------------------------------+
| Archetype: Traveling Employee                               |
| Focus: Fast entry, mobile responsiveness, clear status      |
| Core KPIs Influenced: KPI 4 (eNPS), KPI 5 (Adoption Rate)   |
+-------------------------------------------------------------+
```

### Profile & Context
Alex is a Senior Sales Consultant who travels domestic and international routes twice a month to meet clients. He is tech-savvy, heavily dependent on his smartphone, and does not have time to sit at a desktop copy-pasting numbers from paper receipts.

### Goals & Objectives
* **Fast Reimbursements:** Get personal money back quickly (ACH settlements) for out-of-pocket expenses.
* **Low Administration:** Photograph receipts on the go immediately after buying meals, renting cars, or checking out of hotels.
* **Policy Guidance:** Know immediately if an expense violates policy *before* submitting it to avoid delays.
* **Cash Advances:** Request cash advances for long international trips easily.

### Current Pain Points (Manual System)
* Wait times of 18–22 days for reimbursement.
* Tracking physical receipts while traveling and losing them.
* Rejections received days or weeks later because a manager didn't approve an exception.
* Manually typing in exchange rates and calculating conversions for foreign trips.

### Key Actions in System
1. **Submit Travel Request:** Pre-populate details from HRIS, request optional cash advance.
2. **Mobile OCR Upload:** Snap a photo of a receipt, verify parsed data (amount, merchant, date, category), and submit.
3. **Multi-Currency Entries:** Log expenses in local currency (e.g., EUR) and see automatic conversion to base currency (USD).
4. **Track Status:** Monitor approval process and reimbursement progress on the dashboard.

### System Permissions
* Own data access only. Read-only access to company travel policies. No permission to edit reports once submitted or approved.

---

## 2. Approving Manager: "Marcus, the Department Lead"
*“I need to make sure my team is staying within budget and following policies, without being bogged down by notifications.”*

```
+-------------------------------------------------------------+
| Archetype: Approving Manager                                |
| Focus: Quick reviews, budget checks, task delegation        |
| Core KPIs Influenced: KPI 1 (Cycle Time), SLA compliance    |
+-------------------------------------------------------------+
```

### Profile & Context
Marcus manages a team of 15 sales and technical staff. He is busy with meetings, projects, and strategy. He views expense approval as a necessary administrative task that should take as little time as possible, but he is financially accountable for his department's budget.

### Goals & Objectives
* **Clear Context:** See cost breakdowns, policies violated, and current budget statuses at a single glance.
* **Efficiency:** Approve standard, compliant team expenses in bulk or with single-click actions.
* **Budget Tracking:** Track real-time travel accruals against remaining department budgets before signing off.
* **Notification Control:** Avoid email fatigue but receive immediate alerts for high-value or out-of-policy requests.

### Current Pain Points (Manual System)
* Approving travel request emails without knowing how much travel budget is left in the ERP.
* Reviewing Excel sheets manually line-by-line to match receipts.
* Chasing team members for missing receipts or justifications.
* No system to delegate approval duties when on leave.

### Key Actions in System
1. **Review Travel Requests:** Approve, reject, or request changes with context.
2. **Review Expense Reports:** Inspect receipts and policy exceptions.
3. **Budget Validation:** Compare report costs against departmental budget graphs loaded from the ERP.
4. **Delegate Authority:** Configure temporary approval routing delegates when traveling or on leave.

### System Permissions
* Full access to reports and travel requests of direct reports. Cross-charge approvals for specific project codes. No permission to approve own expenses.

---

## 3. Corporate Travel Desk: "Sarah, the Booking Coordinator"
*“My priority is booking flights and hotels quickly while ensuring we maximize corporate vendor discounts.”*

```
+-------------------------------------------------------------+
| Archetype: Corporate Travel Desk                            |
| Focus: Booking matching, vendor compliance, itinerary changes|
| Core KPIs Influenced: Policy compliance, corporate discounts |
+-------------------------------------------------------------+
```

### Profile & Context
Sarah manages corporate travel bookings for the company. She works closely with preferred airlines, hotel chains, and car rental agencies. She is details-oriented and relies on pre-approved travel itineraries to coordinate flights and rooms.

### Goals & Objectives
* **Pre-Approved Access:** View all travel requests marked "Approved for Booking" instantly.
* **Preferred Vendors:** Enforce corporate vendor booking to maximize company-negotiated discounts.
* **Reconciliation:** Match travel requests to final itineraries to ensure zero unauthorized bookings.

### Current Pain Points (Manual System)
* Booking travel based on verbal or email approvals, leading to post-booking audits showing lack of approval documentation.
* Handling manual changes to flights or hotels via telephone threads.
* Manually keying itineraries back to finance for reconciliations.

### Key Actions in System
1. **Access Booking Queue:** Retrieve approved travel requests (`APPROVED_FOR_BOOKING`).
2. **Input Booking Details:** Record booking details, flight carriers, hotel names, and reservation numbers back into the travel request log.
3. **Modify Itineraries:** Update travel details in cases of cancellations or rescheduling within policy constraints.

### System Permissions
* Read-only access to travel requests. Write access on booking details/reservations. No access to financial expense reports.

---

## 4. Finance Auditor: "Fiona, the Compliance Guard"
*“I only want to spend time reviewing reports that have actual issues or policy violations.”*

```
+-------------------------------------------------------------+
| Archetype: Finance Auditor                                  |
| Focus: Exception management, GL coding, fraud detection     |
| Core KPIs Influenced: KPI 2 (Compliance), KPI 3 (Overhead)  |
+-------------------------------------------------------------+
```

### Profile & Context
Fiona is part of the corporate finance team. She manages tax compliance, corporate credit card reconciliations, general ledger integrations, and internal audit compliance. She is highly analytical, risk-aware, and values structured workflows.

### Goals & Objectives
* **Exception-Only Workflow:** Focus only on expense items flagged by the policy engine.
* **Audit Trail Security:** Ensure all overrides have clear reasons and timestamps for tax authorities.
* **ERP Accuracy:** Verify cost centers, tax codes, and general ledger coding are correct before ERP ingestion.
* **Prevent Fraud:** Prevent duplicate receipt submissions or claims for personal expenses.

### Current Pain Points (Manual System)
* Spending 120 hours/week manually matching paper receipts and checking every line.
* Finding duplicate receipt entries after reimbursement occurred.
* Chasing managers for missing override approvals, leading to audit deficiencies.

### Key Actions in System
1. **Access Exception Queue:** Review flagged items (out-of-policy, duplicate hashes, FX deviations).
2. **Process Overrides:** Select standard override reasons from dropdowns or request clarification from the traveler.
3. **Execute Batches:** Approve and push cleared batches to NetSuite ERP.
4. **Manage DLQs / Payment Exceptions:** Identify and correct payment errors (e.g., banking failure notifications).

### System Permissions
* Access to all expense reports for assigned cost centers. Policy override and manual freeze capability. Read-only on receipts (cannot modify images). Full audit log creation.

---

## 5. System Administrator: "Adam, the Platform Configurator"
*“I ensure the system integrates cleanly with HRIS and ERP, matches current policies, and scales during peak hours.”*

```
+-------------------------------------------------------------+
| Archetype: System Administrator                             |
| Focus: Integrations, rule configuration, access controls    |
| Core KPIs Influenced: System availability, API sync health  |
+-------------------------------------------------------------+
```

### Profile & Context
Adam is a Senior IT Systems Administrator. He manages corporate SaaS platforms, handles system integrations, manages roles and directory syncs, and ensures data security controls are active.

### Goals & Objectives
* **Seamless Integrations:** Ensure HRIS (Workday), ERP (NetSuite), and Okta sync APIs run error-free.
* **Rules Up-Time:** Maintain the policy engine rules database, making sure updates roll out without disrupting active evaluations.
* **Scale Integrity:** Monitor latency, database locks, and performance metrics, scaling up pods to handle Monday morning spikes.

### Current Pain Points (Manual System)
* Manually creating and deleting user accounts as employees join or leave.
* Hardcoded policy changes in legacy spreadsheets, needing constant adjustments.
* No system health logs or webhook queue tracking, leading to invisible connection breaks.

### Key Actions in System
1. **Configure Approval Matrices:** Maintain multi-level hierarchy structures.
2. **Update Policy Rules:** Modify limits (e.g., Tier 1 city meal caps) in the database rule tables.
3. **Integration Monitoring:** Check API sync status logs, dead-letter queues, and response times.
4. **Provision Roles:** Oversee SCIM configurations and resolve identity access tickets.

### System Permissions
* System-wide configuration access. Access control rules management. No authority to create, approve, or modify financial expense transactions.

---

*All UI layouts, feature flags, and notifications should be designed around the roles and goals of these five personas.*
