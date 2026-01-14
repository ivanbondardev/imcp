# F1 Sea Import — n8n Workflows Map

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [01_architecture_overview.md](../../core/01_architecture_overview.md) — n8n як Orchestrator  
- [02_core_data_model.md](../../core/02_core_data_model.md) — Event Taxonomy

Мета: описати n8n як оркестратор (executor), а Supabase як source of truth.

> ⚠️ **Принцип:** n8n не зберігає стан. Все записується в Supabase.

---

## Workflow List

| Workflow Name | Trigger | Input | Output (Supabase) |
|---|---|---|---|
| `f1_intake_request` | UI action event | caseId | state → `WAITING_CLIENT_INFO`, message draft |
| `f1_intake_extract` | incoming message webhook | raw text | extracted fields, state → `CLIENT_INFO_COLLECTED` |
| `f1_create_1c_request` | UI action event | caseId | 1c_request_id, state → `REQUEST_1C_CREATED` |
| `f1_calculate_quote` | state = CLIENT_INFO_COLLECTED | case payload | computed.quote, state → `QUOTE_READY` |
| `f1_quote_approval_flow` | approval created/decided | approval payload | state → `QUOTE_SENT` or `QUOTE_READY` |
| `f1_client_confirmed` | client confirm webhook | caseId | approval `ONEC_DEAL_CREATE_APPROVAL` |
| `f1_create_import_deal` | approval decided | caseId | deal_number, state → `DEAL_IMPORT_CREATED` |
| `f1_post_deal_actions` | state = DEAL_IMPORT_CREATED | deal_number | marking, VP table, ZED notify |
| `f1_warehouse_dims` | warehouse dims webhook | dims data | state → `WAREHOUSE_DIMS_RECEIVED`, approval if mismatch |
| `f1_bl_draft_review` | BL draft uploaded event | docs | diff report, approval `BL_DRAFT_APPROVAL` |
| `f1_pre_arrival_tasks` | cron/ETA event | caseId | state → `PRE_ARRIVAL_TASKS`, tasks |
| `f1_poa_approval_flow` | POA uploaded event | caseId | approval `POA_SCAN_APPROVAL` |
| `f1_broker_docs_request` | POA confirmed event | caseId | state → `BROKER_DOCS_REQUESTED` |

---

## Workflow Details

### f1_intake_request

```
Trigger: case_action = 'request_client_info'
Steps:
1. Read case from Supabase
2. AI Node: Generate intake questionnaire
3. Notification Node: Send to client
4. Update cases.state = 'WAITING_CLIENT_INFO'
5. Insert case_event: STATE_CHANGED
```

### f1_intake_extract

```
Trigger: webhook incoming_client_message
Steps:
1. AI Node: Extract fields from raw text
2. Insert ai_runs record
3. Update cases.payload with extracted fields
4. If all fields complete:
   - Update cases.state = 'CLIENT_INFO_COLLECTED'
   - Insert case_event: STATE_CHANGED
5. Else:
   - Update computed.risks with LOW_CONFIDENCE/MISSING_DATA
   - AI Node: Generate follow-up questions
   - Notification Node: Send follow-up
```

### f1_calculate_quote

```
Trigger: state = 'CLIENT_INFO_COLLECTED' (or manual)
Steps:
1. Read case payload
2. Calculation Node: Compute costs + margin
3. AI Node: Generate quote email draft
4. Update computed.quote
5. Update cases.state = 'QUOTE_READY'
6. Insert case_event: STATE_CHANGED
7. Create approval: QUOTE_APPROVAL
8. Update cases.state = 'QUOTE_APPROVAL_PENDING'
9. Insert case_event: APPROVAL_CREATED
```

### f1_create_import_deal

```
Trigger: approval_decided (ONEC_DEAL_CREATE_APPROVAL, status=APPROVED)
Steps:
1. Insert case_event: APPROVAL_APPROVED
2. 1C Node: Create deal (type=import)
3. Update payload.integration.onec_deal_number
4. Update payload.integration.marking_code = 'ALEX-' + deal_number
5. Insert case_event: INTEGRATION_SUCCESS
6. Update cases.state = 'DEAL_IMPORT_CREATED'
7. Insert case_event: STATE_CHANGED
8. Trigger: f1_post_deal_actions
```

### f1_post_deal_actions

```
Trigger: state = 'DEAL_IMPORT_CREATED'
Steps:
1. AI Node: Generate warehouse instructions
2. Notification Node: Send to client
3. Insert case_event: EMAIL_SENT
4. Update cases.state = 'WAREHOUSE_INSTRUCTIONS_SENT'
5. Insert case_event: STATE_CHANGED
6. Spreadsheet Node: Update "Морські вантажі ВП"
7. Insert case_event: INTEGRATION_SUCCESS (type: SPREADSHEET)
8. Update cases.state = 'VP_TABLE_UPDATED'
9. Notification Node: Notify ZED
10. Insert case_event: NOTIFICATION_SENT
11. Update cases.state = 'ZED_NOTIFIED'
```

---

## Error Handling (згідно Core)

### Retry Policy

| Error Type | Strategy | Max Retries | Backoff |
|---|---|---|---|
| Transient (network) | Retry with backoff | 3 | 1s, 4s, 16s |
| Integration (1C error) | Log + create incident | 1 | — |
| Data (validation) | Log + notify manager | 0 | — |
| Critical (DB down) | Alert on-call | — | Pause workflow |

### Idempotency Keys

| Action | Idempotency Key Pattern |
|---|---|
| Create 1C request | `CREATE_1C_REQUEST:<caseId>` |
| Create 1C deal | `CREATE_1C_DEAL:<caseId>` |
| Send quote | `SEND_QUOTE:<caseId>:<quoteVersion>` |
| Update spreadsheet | `UPDATE_VP:<caseId>:<dealNumber>` |
| Notify ZED | `NOTIFY_ZED:<caseId>:<dealNumber>` |

### Failure Events

При помилці записуємо:
- `case_event`: `INTEGRATION_FAILED` з `{error, retry_count, will_retry}`
- `integrations` table: `status=FAILED`, `last_error`

---

## Event Types Used (згідно Core Event Taxonomy)

| Category | Events |
|---|---|
| Core | `CASE_CREATED`, `STATE_CHANGED`, `STATUS_CHANGED` |
| Approvals | `APPROVAL_CREATED`, `APPROVAL_APPROVED`, `APPROVAL_REJECTED` |
| Documents | `DOC_UPLOADED`, `DOC_EXTRACTED`, `DOC_VERIFIED` |
| Integration | `INTEGRATION_STARTED`, `INTEGRATION_SUCCESS`, `INTEGRATION_FAILED` |
| AI | `AI_RUN_COMPLETED` |
| Communication | `EMAIL_SENT`, `NOTIFICATION_SENT` |

---

## Workflow Naming Convention

```
f1_{action}_{detail}

Examples:
- f1_intake_request
- f1_quote_approval_flow
- f1_bl_draft_review
```

Prefix `f1_` вказує на case type `F1_SEA_IMPORT`.
