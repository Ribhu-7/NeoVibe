# Enterprise Employee Travel & Expense Management System
## Frontend & Backend Test Specifications

**Version:** 1.0  
**Status:** Approved  
**Based on:** TravelExpensePRD v1.0, TravelExpenseKPI v1.0, and project_boundary.md  
**Date:** 2026-06-05  

---

## 1. Testing Strategy & Frameworks

We follow a test-driven approach separating client-side user experience testing from server-side transactional, integration, and performance validation.

```
                  +---------------------------------------+
                  |            Testing Pyramid            |
                  +---------------------------------------+
                  |      E2E & UI (Playwright, Detox)     |
                  |     API / Integration (Supertest, k6) |
                  |   Unit / Contract (Vitest, MSW, JUnit)|
                  +---------------------------------------+
```

| Domain | Target Application / Service | Key Tooling / Frameworks | Focus |
|---|---|---|---|
| **Frontend** | `apps/web-portal`<br>`apps/admin-dashboard` | Vitest, React Testing Library, Playwright, MSW (Mock Service Worker) | User interactions, form validations, routing, dynamic component states, visual layouts |
| **Mobile** | `apps/mobile-app` | Jest, React Native Testing Library, Detox, MSW | Camera permissions, offline SQLite caching, image compression, sync queue management |
| **Backend** | `services/*` | Node.js (Vitest, Supertest), Java/Go native test runner, Pact (Contract Testing) | Business logic validation, database schema migrations (Postgres, ClickHouse), API status |
| **Platform** | Entire system | k6, Postman/Newman, OWASP ZAP, SonarQube | Load performance (2,500 reqs/min), authentication flow, role checks (RBAC), code coverage |

---

## 2. Frontend Test Specifications (`apps/`)

### 2.1 Component & Unit Testing Specifications

All interactive components must undergo isolated rendering and behavior validation using **Vitest** and **React Testing Library**.

#### Spec FT-UT-01: Travel Request Form (Employee Portal)
* **Path:** `apps/web-portal/src/features/travel-requests/__tests__/TravelRequestForm.test.tsx`
* **Target:** `TravelRequestForm.tsx`
* **Test Configurations & Mocks:**
  - Mock `useAuth` hook to return a Traveling Employee role (`legal_entity = 'US_CORP'`, `cost_center = 'CC-US-5420'`).
  - Mock window alerts and console errors.
* **Test Cases:**
  1. Verify pre-population of employee metadata fields (ID, manager email) from the mock auth profile.
  2. Input an estimated flight cost of $10,500 without a business justification. Verify submission button is disabled and validation message appears.
  3. Input an estimated total cost of $4,000. Input a cash advance request of $3,500. Verify the form prints an error: "Cash advance request cannot exceed 80% of estimated costs".
  4. Toggle trip type between "Domestic" and "International". Verify that selecting "Business Class" flight is allowed for International, but blocked/disabled for Domestic (Rule R-TR-01).

#### Spec FT-UT-02: CFO KPI Metrics Grid (Admin Dashboard)
* **Path:** `apps/admin-dashboard/src/components/__tests__/MetricsGrid.test.tsx`
* **Target:** `MetricsGrid.tsx`
* **Test Configurations & Mocks:**
  - Mock API responses for metrics using MSW.
* **Test Cases:**
  1. Render component with mock KPI data: Cycle Time = 3.5 days, Compliance Rate = 96.2%, Audit Overhead = 18 hours/week.
  2. Verify that values are colored in standard green for achieving targets (Cycle Time Target ≤ 4 days, Compliance ≥ 95%, Overhead ≤ 20 hours/week).
  3. Simulate API updating with underperforming metrics (Cycle Time = 6 days, Compliance = 89%). Verify that color states change to warnings (amber/red).

---

### 2.2 Integration & E2E Testing Specifications

End-to-End testing uses **Playwright** (Web) and **Detox** (Mobile) to test complete flows against local sandbox backend configurations.

```mermaid
sequenceDiagram
    autonumber
    actor Employee
    participant UI as Web/Mobile UI
    participant API as API Gateway / Services
    participant DB as Transactional DB
    
    Employee->>UI: Logs in via SSO (Okta redirect mock)
    UI->>API: Fetch active traveler profile
    API-->>UI: Return profile details (synced from HRIS)
    Employee->>UI: Captures receipt image & clicks Submit
    UI->>API: POST /receipt/upload (Image binary)
    API->>API: Execute OCR parse & P-01 Policy Engine
    API-->>UI: Return extraction details & validation flag
    Employee->>UI: Confirms fields & submits expense report
    UI->>API: POST /expense-reports/submit
    API->>DB: Store record (Status = PENDING_MANAGER_APPROVAL)
    API-->>UI: Confirms successful submission
```

#### Spec FT-E2E-01: Full Travel Request & Manager Approval Lifecycle
* **Path:** `apps/web-portal/e2e/travel-approval.spec.ts`
* **Steps:**
  1. Execute a mock login as Employee (Alex).
  2. Navigate to `/travel-requests/new` and submit a valid $3,000 domestic trip request.
  3. Capture the database-generated `request_id`. Verify request is listed in Employee’s dashboard under `PENDING_APPROVAL` status.
  4. Perform login swap as Manager (Marcus).
  5. Navigate to `/manager/approvals` and select the captured `request_id`.
  6. Verify the department budget gauge component is displayed and populated.
  7. Click "Approve".
  8. Swap credentials back to Employee. Verify travel request status has changed to `APPROVED_FOR_BOOKING`.

#### Spec FT-E2E-02: Offline Receipt Capture and Auto-Sync
* **Path:** `apps/mobile-app/e2e/offline-sync.spec.ts`
* **Steps:**
  1. Boot mobile application simulator under flight mode (simulated offline status).
  2. Click "Capture Receipt" and load test image (`receipt_stub.png`).
  3. Confirm extraction fields and click "Save Expense".
  4. Verify the application saves the transaction locally to the SQLite DB and displays "Pending Network Sync" badge.
  5. Restore internet connectivity (disable flight mode simulation).
  6. Verify background scheduler starts sync service automatically.
  7. Assert that the SQLite record is removed, and database checks confirm a successful `POST /receipt/upload` payload matching the local receipt details.

---

## 3. Backend Test Specifications (`services/`)

### 3.1 Unit & Contract Testing Specifications

Backend microservices must be tested in isolation, validating internal utilities, controller responses, and contract definitions.

#### Spec BE-UT-01: P-01 Policy Engine Rules Execution
* **Path:** `services/policy-engine/src/__tests__/policyEvaluator.test.ts`
* **Target:** `policyEvaluator.ts`
* **Test Cases:**
  1. Pass an expense report containing line items: Meal = $80 in New York (Tier 1 city). Assert that engine flags `P-01-01` but allows submission.
  2. Pass two line items containing identical receipt image hashes (`abcd1234hash`). Assert that evaluation fails with hard error block `P-01-02`.
  3. Pass a hotel expense line item dated `2026-06-10` while travel pre-approval dates are `2026-06-15` to `2026-06-18`. Assert that engine flags `P-01-03`.
  4. Validate execution latency. Assert that evaluation of a batch containing 50 line items completes in **less than 100 milliseconds**.

#### Spec BE-UT-02: Receipt Duplicate Detection Hash Generation
* **Path:** `services/expense-service/src/__tests__/hashGenerator.test.ts`
* **Target:** `hashGenerator.ts`
* **Test Cases:**
  1. Pass identical input values (Merchant: "Uber", Date: "2026-06-04", Amount: 24.50). Assert generated SHA-256 hashes match.
  2. Pass values with minimal amount difference (Merchant: "Uber", Date: "2026-06-04", Amount: 24.80). Assert generated hashes do not match.

---

### 3.2 Service & API Endpoint Testing Specifications

Endpoint and route validation are executed using **Supertest** (Node.js) or native service test utilities.

#### Spec BE-API-01: Authentication, Access Control, and SCIM Sync
* **Path:** `services/api-gateway/src/__tests__/authGateway.test.ts`
* **API Endpoints Tested:** `GET /api/v1/expense-reports`, `POST /api/v1/scim/users`
* **Test Cases:**
  1. Trigger `GET /api/v1/expense-reports` without authorization header. Assert return status is `401 Unauthorized`.
  2. Trigger `GET /api/v1/expense-reports` using an expired JWT token. Assert return status is `401 Unauthorized`.
  3. Trigger `GET /api/v1/expense-reports` with valid Employee role token. Assert return status is `200 OK` and payload is filtered to match only the employee's ID.
  4. Trigger `POST /api/v1/scim/users` with SCIM authorization credentials. Set payload:
     ```json
     {
       "userName": "test.user@company.com",
       "active": false
     }
     ```
     Assert database profile is updated to `is_active = FALSE`. Perform immediate route check using the user's active session token. Assert authorization is rejected with `403 Forbidden` status.

#### Spec BE-API-02: NetSuite Ledger Batch Posting Flow
* **Path:** `services/settlement-service/src/__tests__/ledgerBridge.test.ts`
* **API Endpoints Tested:** `POST /api/v1/settlements/batch-post`
* **Test Cases:**
  1. Mock NetSuite REST client API callback to return connection failure. Trigger batch post API call. Assert response status is `503 Service Unavailable`, batch status is marked `PENDING_RETRY`, and payload is stored in the dead-letter queue (DLQ).
  2. Mock NetSuite API callback to return success code. Trigger batch post API call. Assert status is updated to `ERP_POSTED` in database, and payment instructions trigger to JP Morgan API.

---

### 3.3 Database & Integration Testing Specifications

These test specifications target database schema validations, history logging, and replication consistencies.

#### Spec BE-DB-01: SCD Type 2 Hierarchy Preservation
* **Path:** `database/tests/scd_hierarchy.test.sql`
* **Test Steps:**
  1. Set Employee ID `E123` manager to `M456` in employee dimension tables.
  2. Submit expense report `R-001` matching Employee ID `E123`.
  3. Update Employee ID `E123` manager hierarchy to `M789` (simulated manager change sync from Workday).
  4. Query database for report `R-001` approval configuration.
  5. Assert that historical approval record for `R-001` correctly points to the original manager `M456`, while new reports point to `M789`.

#### Spec BE-DB-02: Double Ingestion Prevention (Idempotency)
* **Path:** `services/expense-service/src/__tests__/idempotency.test.ts`
* **Test Steps:**
  1. Send `POST /api/v1/receipts/ocr-ingest` with request body containing receipt details and `Idempotency-Key: idemp-key-999`. Assert response is `200 OK`.
  2. Immediately send identical request body containing the same header key (`idemp-key-999`).
  3. Assert return status is `200 OK` (cached response return) and database has recorded only 1 new transaction row.

---

## 4. Performance & SLA Test Specifications

Performance validations ensure system endpoints maintain stability and meet target response times during concurrent workflows.

| Spec ID | Target System | Test Type | Execution Scenario | Performance Assertions (Pass/Fail) |
|---|---|---|---|---|
| **BE-PERF-01** | `services/policy-engine` | Load / Stress | Inject 250,000 policy validation payload tasks over a 10-minute window (2,500/min scale). | 1. p95 execution latency remains under **200 ms**.<br>2. p99 execution latency remains under **350 ms**.<br>3. Zero payload losses or connection abort errors. |
| **BE-PERF-02** | `services/expense-service` | API Latency | Trigger 5,000 OCR extraction processing endpoints using mock images. | 1. p95 OCR endpoint process time is **under 1,200 ms**.<br>2. Amount accuracy parser matches **99.0%** precision.<br>3. Merchant name classification matches **97.0%** precision. |
| **BE-PERF-03** | `apps/admin-dashboard` | Read replica isolation | Load dashboard visualizations containing 1,000,000 historical record aggregates. | 1. Page render completes in **less than 1.0 second**.<br>2. Dashboard reads run exclusively on ClickHouse/Read Replica.<br>3. Primary transactional database CPU utilization remains below **20%**. |

---

## 5. Security & Vulnerability Audits

### Spec BE-SEC-01: Automated Security Pipeline (SAST & DAST)
* **Execution:** Configured inside GitHub Actions CI/CD step.
* **Scan Rules:**
  - **SonarQube Static Analysis:** Minimum code coverage requirements: **85.0%**. Zero security hotspots allowed.
  - **OWASP ZAP Dynamic Scanning:** Perform automated vulnerability scans against exposed APIs (`/scim/*`, `/expense-reports/*`). Assert zero SQL injection (SQLi) or Cross-Site Scripting (XSS) warnings are returned.

### Spec BE-SEC-02: JWT Token Compromise Defense
* **Target:** `api-gateway`
* **Test Scenarios:**
  1. Attempt authorization using JWT signed with a custom key. Assert return status is `401 Unauthorized`.
  2. Inject SQL syntax payloads into the JWT header payload keys. Assert gateway handles parsing safely and rejects request.

---

## 6. Test Environment Configuration

Testing configurations must define isolated system environment parameters to prevent test logs from writing to production systems.

```yaml
# configuration.test.yml
env: test
gateway:
  jwt_secret: test_jwt_secret_key_12345678
  port: 8080
database:
  transactional_url: postgresql://test_user:test_pass@localhost:5432/te_transactional_test
  analytical_url: clickhouse://test_user:test_pass@localhost:8123/te_analytical_test
services:
  ocr_api_endpoint: http://localhost:9091/mock-ocr
  open_exchange_endpoint: http://localhost:9092/mock-exchange
  netsuite_endpoint: http://localhost:9093/mock-netsuite
  jpm_endpoint: http://localhost:9094/mock-jpmorgan
```
