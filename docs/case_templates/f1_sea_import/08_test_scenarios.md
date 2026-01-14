# F1 Sea Import — Test Scenarios (PoC)

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [02_core_data_model.md](../../core/02_core_data_model.md) — State vs Status, Event Taxonomy  
- [03_approval_pattern.md](../../core/03_approval_pattern.md) — Approval Flow

Ціль: перевірити не лише happy path, а типові винятки.

> ⚠️ **Термінологія:**  
> - `state` — бізнес-стан (напр. `WAITING_CLIENT_INFO`)  
> - `status` — технічний агрегат (`OPEN`, `BLOCKED`, `DONE`)

---

## A) Happy Path

### Test Case: HP-001 — Full Flow Success

**Preconditions:**
- Case created, state = `NEW`, status = `OPEN`

**Steps:**
1. Створити кейс → state = `NEW`
2. Запросити дані клієнта (7 питань) → state = `WAITING_CLIENT_INFO`
3. Отримати відповіді → AI extraction → state = `CLIENT_INFO_COLLECTED`
4. Створити запит у 1С → state = `REQUEST_1C_CREATED`
5. Порахувати quote → state = `QUOTE_READY`
6. Approval `QUOTE_APPROVAL` → APPROVED → state = `QUOTE_SENT`
7. Клієнт підтвердив → state = `CONFIRMED`
8. Approval `ONEC_DEAL_CREATE_APPROVAL` → APPROVED
9. Створити угоду "імпорт" → deal_number → state = `DEAL_IMPORT_CREATED`
10. ALEX marking → таблиця → ЗЕД → state = `ZED_NOTIFIED`
11. Отримати dims зі складу → no mismatch → state = `DIMS_CONFIRMED`
12. Отримати BL draft → AI verify → approval `BL_DRAFT_APPROVAL` → APPROVED → state = `BL_CONFIRMED`
13. ETA-3 weeks → state = `PRE_ARRIVAL_TASKS`
14. POA scan → approval `POA_SCAN_APPROVAL` → APPROVED → state = `POA_SCAN_VERIFIED`
15. Original confirmed → state = `POA_ORIGINAL_CONFIRMED`
16. Broker docs request → state = `BROKER_DOCS_REQUESTED`
17. Docs prepared → state = `READY_FOR_CLEARANCE`
18. Close case → state = `CLOSED`, status = `DONE`

**Expected Events (sample):**
- `CASE_CREATED`
- `STATE_CHANGED` (multiple)
- `APPROVAL_CREATED` / `APPROVAL_APPROVED`
- `DOC_UPLOADED` / `DOC_VERIFIED`
- `INTEGRATION_SUCCESS`
- `EMAIL_SENT`

---

## B) Missing Client Fields

### Test Case: MF-001 — Incomplete Intake

**Preconditions:**
- Case created, state = `WAITING_CLIENT_INFO`
- Client sends partial data (missing dimensions)

**Steps:**
1. Client reply arrives (missing cargo_packages)
2. AI extraction runs → confidence OK for other fields
3. n8n detects missing_fields = ['cargo_packages']

**Expected:**
- state remains `WAITING_CLIENT_INFO`
- `computed.risks[]` contains `{code: 'MISSING_DATA', field: 'cargo_packages'}`
- AI generates follow-up question
- `case_event`: `AI_RUN_COMPLETED`
- `case_event`: `EMAIL_SENT` (follow-up)

**NOT Expected:**
- state change to `CLIENT_INFO_COLLECTED`

---

## C) Warehouse Dims Mismatch

### Test Case: DM-001 — Dims Differ by >10%

**Preconditions:**
- Case state = `WAITING_WAREHOUSE_DIMS`
- Client declared: 10 pcs, 3.2 cbm
- Warehouse reports: 10 pcs, 4.1 cbm (28% diff)

**Steps:**
1. Warehouse webhook arrives with measured dims
2. n8n compares with `payload.cargo.dimensions`
3. Mismatch detected (28% > threshold)

**Expected:**
- state → `DIMS_APPROVAL_PENDING`
- `approvals` record created: `approval_type = 'DIMS_CHANGE_APPROVAL'`
- `request_snapshot` contains diff summary
- `case_event`: `APPROVAL_CREATED`
- UI shows approval in inbox

**After Manager Approves:**
- `payload.cargo.final_dimensions` = warehouse_measured
- state → `DIMS_CONFIRMED`
- `case_event`: `APPROVAL_APPROVED`
- `case_event`: `STATE_CHANGED`
- Client notified about final dims

---

## D) Dangerous Goods

### Test Case: DG-001 — Dangerous Goods Declared

**Preconditions:**
- Client intake includes `dangerous_goods = true`

**Steps:**
1. AI extracts field `dangerous_goods = true`
2. n8n processes intake

**Expected:**
- `computed.risks[]` contains `{code: 'DANGEROUS_GOODS', severity: 'HIGH'}`
- `cases.priority` elevated (optional)
- Additional task created for manager: "Request dangerous goods details"
- Automatic state transitions blocked until additional info collected

**NOT Expected:**
- System proceeds to quote without verification

---

## E) BL Draft Mismatch

### Test Case: BL-001 — Seller/Buyer Mismatch

**Preconditions:**
- Case state = `BL_DRAFT_RECEIVED`
- Contract: seller = "China Supplier Ltd", buyer = "Client LLC"
- BL draft: seller = "China Supplier Ltd", buyer = "Client LTD" (typo)

**Steps:**
1. BL draft uploaded by ZED
2. AI verification runs
3. AI detects buyer mismatch

**Expected:**
- `ai_runs` record: `run_type = 'VERIFY'`, confidence < 1.0
- `approvals` record: `approval_type = 'BL_DRAFT_APPROVAL'`
- `request_snapshot.ai_verification.issues[]` contains:
  ```json
  {"field": "buyer", "expected": "Client LLC", "actual": "Client LTD", "severity": "HIGH"}
  ```
- state → `BL_REVIEW_PENDING`
- `case_event`: `AI_RUN_COMPLETED`
- `case_event`: `APPROVAL_CREATED`

**After Manager Rejects:**
- `case_event`: `APPROVAL_REJECTED`
- AI generates fix list
- ZED and Broker notified
- state remains `BL_REVIEW_PENDING` (loop for new draft)

---

## F) Duplicate Webhook / Retry

### Test Case: ID-001 — Idempotent Deal Creation

**Preconditions:**
- Approval `ONEC_DEAL_CREATE_APPROVAL` approved
- 1C deal creation in progress

**Steps:**
1. n8n sends request to 1C → success → deal_number = "IMP-456"
2. n8n crashes before updating Supabase
3. n8n restarts and retries
4. Request sent to 1C again (same idempotency_key)

**Expected:**
- 1C returns existing deal_number (or error "already exists")
- n8n uses idempotency_key to detect duplicate
- NO second deal created in 1C
- Only one `case_event`: `INTEGRATION_SUCCESS` (deduplicated)
- Only one deal_number in `payload.integration`

---

## G) Approval Timeout

### Test Case: AT-001 — Stale Approval

**Preconditions:**
- Approval `QUOTE_APPROVAL` created
- 48 hours passed without decision

**Steps:**
1. Cron workflow checks stale approvals
2. Finds approval older than threshold

**Expected:**
- Notification sent to manager: "Pending approval requires attention"
- `case_event`: `NOTIFICATION_SENT`
- (Optional) Escalation to supervisor

**NOT Expected:**
- Automatic approval/rejection

---

## H) Quote Rejected

### Test Case: QR-001 — Manager Rejects Quote

**Preconditions:**
- state = `QUOTE_APPROVAL_PENDING`
- Approval `QUOTE_APPROVAL` pending

**Steps:**
1. Manager clicks "Reject" with comment "Margin too low"
2. Decision saved to Supabase

**Expected:**
- `approvals.status` = `REJECTED`
- `approvals.decision_comment` = "Margin too low"
- state → `QUOTE_READY` (back to recalculation)
- `case_event`: `APPROVAL_REJECTED`
- `case_event`: `STATE_CHANGED`

---

## I) Integration Failure

### Test Case: IF-001 — 1C Node Error

**Preconditions:**
- Approval `ONEC_DEAL_CREATE_APPROVAL` approved
- 1C system temporarily unavailable

**Steps:**
1. n8n attempts to create deal in 1C
2. 1C returns error 503

**Expected:**
- Retry 1: wait 1s, retry
- Retry 2: wait 4s, retry
- Retry 3: wait 16s, retry
- After 3 retries: fail
- `integrations.status` = `FAILED`
- `integrations.last_error` = "503 Service Unavailable"
- `case_event`: `INTEGRATION_FAILED`
- Incident created for manual review
- Manager notified

**NOT Expected:**
- state changes without successful integration

---

## J) Concurrent Approval Decision

### Test Case: CA-001 — Two Managers Try to Approve

**Preconditions:**
- Approval `QUOTE_APPROVAL` pending
- Manager A and Manager B both view approval

**Steps:**
1. Manager A clicks "Approve" at T+0
2. Manager B clicks "Approve" at T+0.5s

**Expected:**
- Manager A's decision saved (status = APPROVED)
- Manager B's request fails (optimistic lock: `WHERE status='PENDING'` returns 0 rows)
- Manager B sees error: "Approval already decided"
- Only one `case_event`: `APPROVAL_APPROVED`

---

## Verification Checklist

For each test scenario, verify:

| # | Check | Method |
|---|---|---|
| 1 | Correct state after action | Query `cases.state` |
| 2 | Correct status | Query `cases.status` |
| 3 | Event logged | Query `case_events` |
| 4 | Approval created/updated | Query `approvals` |
| 5 | Document stored | Query `documents` |
| 6 | Risks updated | Query `cases.computed.risks` |
| 7 | Notification sent | Check logs / mock |
| 8 | Integration recorded | Query `integrations` |
| 9 | Idempotency works | Repeat action, check no duplicates |
