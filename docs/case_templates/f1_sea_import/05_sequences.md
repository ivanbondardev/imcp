# F1 Sea Import — Sequence Diagrams (Mermaid)

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [01_architecture_overview.md](../../core/01_architecture_overview.md) — Sequence Diagrams, UI Contract  
- [02_core_data_model.md](../../core/02_core_data_model.md) — Event Taxonomy, State vs Status

Цей документ містить **Mermaid sequence diagrams** для ключових сценаріїв кейса `F1_SEA_IMPORT`.

Учасники (lifelines) спрощені відповідно до PoC-ландшафту:
- `Manager UI` (Next.js)
- `Supabase` (DB + Storage)
- `n8n Workflow` (orchestrator)
- спеціалізовані `n8n Nodes` (AI / 1C / Notification / Spreadsheet)
- зовнішні актори (Client / Warehouse / ZED / Broker)

> ⚠️ **Термінологія:**  
> - `state` — бізнес-стан (змінюється через `cases.state`)  
> - `status` — технічний агрегат (`OPEN`, `BLOCKED`, `DONE`, `ARCHIVED`)  
> - Events використовують канонічні типи з Core Event Taxonomy

---

## Sequence 1 — Intake: Request Client Info → Extract Answers → Missing Fields Loop

```mermaid
sequenceDiagram
autonumber
actor Client
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant AI as n8n AI Node
participant N as n8n Notification Node

UI->>DB: INSERT cases (state=NEW, status=OPEN)
DB-->>UI: case_id
UI->>DB: INSERT case_event (CASE_CREATED, actor_type=HUMAN)
UI->>DB: UPDATE cases.payload (trigger=request_client_info)
DB-->>WF: Webhook: CASE_UPDATED (payload changed)

WF->>AI: Generate intake questionnaire (7 questions)
AI-->>WF: Draft questionnaire text + structured checklist
WF->>DB: Store draft message
WF->>DB: UPDATE cases SET state='WAITING_CLIENT_INFO'
WF->>DB: INSERT case_event (STATE_CHANGED, from:NEW, to:WAITING_CLIENT_INFO)
WF->>N: Send message to Client (questionnaire)

Client-->>N: Reply with answers (free text)
N-->>WF: Webhook: incoming_client_message(caseId, rawText)

WF->>AI: Extract intake fields from rawText
AI-->>WF: Extracted fields + per-field confidence + missing_fields
WF->>DB: INSERT ai_runs (run_type=EXTRACT)
WF->>DB: INSERT case_event (AI_RUN_COMPLETED)
WF->>DB: UPDATE cases.payload with extracted fields
alt missing_fields not empty OR low confidence
  WF->>DB: UPDATE cases.computed.risks[] add LOW_CONFIDENCE/MISSING_DATA
  WF->>AI: Generate follow-up questions for missing fields
  AI-->>WF: Follow-up draft
  WF->>N: Send follow-up to Client
else all required fields complete
  WF->>DB: UPDATE cases SET state='CLIENT_INFO_COLLECTED'
  WF->>DB: INSERT case_event (STATE_CHANGED, from:WAITING_CLIENT_INFO, to:CLIENT_INFO_COLLECTED)
end

UI-->>DB: Read case (poll/realtime)
DB-->>UI: Render checklist: filled/missing/needs verification
```

---

## Sequence 2 — Quote: Create 1C Request → Calculate Quote → Approval Gate → Send Quote

```mermaid
sequenceDiagram
autonumber
actor Client
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant C1 as n8n 1C Node
participant AI as n8n AI Node
participant N as n8n Notification Node

UI->>DB: UPDATE cases.payload (trigger=create_1c_request)
UI->>DB: INSERT case_event (COMMENT_ADDED, action=create_1c_request)
DB-->>WF: Webhook: CASE_UPDATED (action requested)

WF->>C1: Create 1C Request (payload from case.payload)
C1-->>WF: 1c_request_id
WF->>DB: Save 1c_request_id to payload.integration
WF->>DB: UPDATE cases SET state='REQUEST_1C_CREATED'
WF->>DB: INSERT case_event (STATE_CHANGED)
WF->>DB: INSERT case_event (INTEGRATION_SUCCESS, external_id=1c_request_id)

WF->>DB: Read case + payload fields
WF->>WF: Calculate pricing (costs + margin) [PoC calc node]
WF->>AI: Generate quote email draft (price + assumptions + note)
AI-->>WF: Quote draft text + structured quote data
WF->>DB: Save quote to computed.quote
WF->>DB: UPDATE cases SET state='QUOTE_READY'
WF->>DB: INSERT case_event (STATE_CHANGED)

WF->>DB: INSERT approvals (approval_type=QUOTE_APPROVAL, status=PENDING, request_snapshot)
WF->>DB: UPDATE cases SET state='QUOTE_APPROVAL_PENDING'
WF->>DB: INSERT case_event (APPROVAL_CREATED)
DB-->>UI: Show approval in inbox (PENDING)

UI->>DB: Manager decision: approve/edit/reject (QUOTE_APPROVAL)
DB-->>WF: Event: approval_decided(QUOTE_APPROVAL)

alt approved or edited
  WF->>DB: INSERT case_event (APPROVAL_APPROVED, has_edits:bool)
  WF->>N: Send quote to Client + validity note
  WF->>DB: UPDATE computed.quote.valid_until
  WF->>DB: INSERT case_event (EMAIL_SENT)
  WF->>DB: UPDATE cases SET state='QUOTE_SENT'
  WF->>DB: INSERT case_event (STATE_CHANGED)
else rejected
  WF->>DB: INSERT case_event (APPROVAL_REJECTED)
  WF->>DB: UPDATE cases SET state='QUOTE_READY'
  WF->>DB: INSERT case_event (STATE_CHANGED)
end

Client-->>N: (optional) questions / negotiation
N-->>WF: Webhook: client_reply(caseId)
WF->>DB: INSERT case_event (COMMENT_ADDED, source:CLIENT)
```

---

## Sequence 3 — Client Confirmed: Create Import Deal in 1C → Warehouse Instructions → VP Table → Notify ZED

```mermaid
sequenceDiagram
autonumber
actor Client
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant C1 as n8n 1C Node
participant AI as n8n AI Node
participant N as n8n Notification Node
participant S as n8n Spreadsheet Node
actor ZED as ZED Department

Client-->>N: Client confirms shipment ("YES")
N-->>WF: Webhook: client_confirmed(caseId)
WF->>DB: UPDATE cases SET state='CONFIRMED'
WF->>DB: INSERT case_event (STATE_CHANGED)

WF->>DB: INSERT approvals (approval_type=ONEC_DEAL_CREATE_APPROVAL, status=PENDING)
WF->>DB: UPDATE cases SET state='DEAL_IMPORT_PENDING'
WF->>DB: INSERT case_event (APPROVAL_CREATED)
DB-->>UI: Show approval "Create Import Deal" (PENDING)

UI->>DB: Manager decision approve/reject (ONEC_DEAL_CREATE_APPROVAL)
DB-->>WF: Event: approval_decided(ONEC_DEAL_CREATE_APPROVAL)

alt approved
  WF->>DB: INSERT case_event (APPROVAL_APPROVED)
  WF->>C1: Create 1C Deal type="import" from 1C request + case data
  C1-->>WF: deal_number
  WF->>DB: Save deal_number to payload.integration.onec_deal_number
  WF->>DB: Save marking_code="ALEX-"+deal_number to payload.integration
  WF->>DB: INSERT case_event (INTEGRATION_SUCCESS, external_id=deal_number)
  WF->>DB: UPDATE cases SET state='DEAL_IMPORT_CREATED'
  WF->>DB: INSERT case_event (STATE_CHANGED)

  WF->>AI: Generate message: warehouse address + marking instructions
  AI-->>WF: Draft instructions
  WF->>N: Send to Client (warehouse instructions + ALEX-marking)
  WF->>DB: INSERT case_event (EMAIL_SENT)
  WF->>DB: UPDATE cases SET state='WAREHOUSE_INSTRUCTIONS_SENT'
  WF->>DB: INSERT case_event (STATE_CHANGED)

  WF->>S: Append/Update row in "Морські вантажі ВП"
  S-->>WF: row_id (or ok)
  WF->>DB: INSERT case_event (INTEGRATION_SUCCESS, type:SPREADSHEET)
  WF->>DB: UPDATE cases SET state='VP_TABLE_UPDATED'
  WF->>DB: INSERT case_event (STATE_CHANGED)

  WF->>N: Notify ZED (expected cargo + link/table reference)
  N-->>ZED: Message delivered
  WF->>DB: INSERT case_event (NOTIFICATION_SENT, channel:ZED)
  WF->>DB: UPDATE cases SET state='ZED_NOTIFIED'
  WF->>DB: INSERT case_event (STATE_CHANGED)
else rejected
  WF->>DB: INSERT case_event (APPROVAL_REJECTED)
  WF->>DB: UPDATE cases SET state='CONFIRMED'
  WF->>DB: INSERT case_event (STATE_CHANGED)
end
```

---

## Sequence 4 — Warehouse Dims: Compare → Dims Approval Gate → Notify Client → Lock Final Dims

```mermaid
sequenceDiagram
autonumber
actor Warehouse
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant AI as n8n AI Node
participant N as n8n Notification Node

Warehouse-->>N: Warehouse reports measured dims (Yiwu/Shenzhen)
N-->>WF: Webhook: warehouse_dims(caseId, dims, packages)

WF->>DB: Save warehouse_measured_dims to payload.cargo.warehouse_*
WF->>DB: INSERT case_event (DOC_UPLOADED, doc_type:WAREHOUSE_DIMS, source:INTEGRATION)
WF->>DB: UPDATE cases SET state='WAREHOUSE_DIMS_RECEIVED'
WF->>DB: INSERT case_event (STATE_CHANGED)

WF->>DB: Read payload.cargo.dimensions (client declared)
WF->>WF: Compare dims (diff %, mismatch threshold)

alt mismatch > threshold OR stackable changed
  WF->>AI: Generate diff summary for manager (impact on docs/price)
  AI-->>WF: Diff summary
  WF->>DB: INSERT approvals (approval_type=DIMS_CHANGE_APPROVAL, request_snapshot=diff)
  WF->>DB: UPDATE cases SET state='DIMS_APPROVAL_PENDING'
  WF->>DB: INSERT case_event (APPROVAL_CREATED)
  DB-->>UI: Show approval "Confirm Dims Change" (PENDING)

  UI->>DB: Manager decision approve/edit/reject (DIMS_CHANGE_APPROVAL)
  DB-->>WF: Event: approval_decided(DIMS_CHANGE_APPROVAL)

  alt approved
    WF->>DB: INSERT case_event (APPROVAL_APPROVED)
    WF->>DB: Set payload.cargo.final_dimensions = warehouse_measured
    WF->>DB: UPDATE cases SET state='DIMS_CONFIRMED'
    WF->>DB: INSERT case_event (STATE_CHANGED)
  else edited
    WF->>DB: INSERT case_event (APPROVAL_APPROVED, has_edits:true)
    WF->>DB: Set payload.cargo.final_dimensions = manager_edited
    WF->>DB: UPDATE cases SET state='DIMS_CONFIRMED'
    WF->>DB: INSERT case_event (STATE_CHANGED)
  else rejected
    WF->>DB: INSERT case_event (APPROVAL_REJECTED)
    WF->>DB: UPDATE cases SET state='WAITING_WAREHOUSE_DIMS'
    WF->>DB: INSERT case_event (STATE_CHANGED)
  end
else no mismatch
  WF->>DB: Set payload.cargo.final_dimensions = warehouse_measured
  WF->>DB: UPDATE cases SET state='DIMS_CONFIRMED'
  WF->>DB: INSERT case_event (STATE_CHANGED)
end

WF->>AI: Generate message to client with confirmed dims (for BL/PL)
AI-->>WF: Draft message
WF->>N: Send confirmed dims to Client
WF->>DB: INSERT case_event (EMAIL_SENT)
```

---

## Sequence 5 — BL Draft Review: Receive Draft → AI Verify → Approval Gate → Confirm/Rework Loop

```mermaid
sequenceDiagram
autonumber
actor ZED as ZED Department
actor Broker
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant AI as n8n AI Node
participant N as n8n Notification Node

ZED-->>N: Sends BL draft (document)
N-->>WF: Webhook: document_received(caseId, docType=BL_DRAFT, fileRef)

WF->>DB: Store doc in Supabase Storage (BL draft)
WF->>DB: INSERT documents (doc_type=BL_DRAFT, source=ZED, status=UPLOADED)
WF->>DB: INSERT case_event (DOC_UPLOADED, doc_type:BL_DRAFT, source:ZED)
WF->>DB: UPDATE cases SET state='BL_DRAFT_RECEIVED'
WF->>DB: INSERT case_event (STATE_CHANGED)

WF->>DB: Fetch contract + invoice + final_dims references
WF->>AI: Verify BL draft vs contract/invoice/final_dims (diff report)
AI-->>WF: Diff report + issues list + confidence
WF->>DB: INSERT ai_runs (run_type=VERIFY)
WF->>DB: UPDATE documents SET extracted_data, extraction_confidence
WF->>DB: INSERT case_event (DOC_EXTRACTED, ai_run_id)
WF->>DB: INSERT approvals (approval_type=BL_DRAFT_APPROVAL, request_snapshot=diff+refs)
WF->>DB: UPDATE cases SET state='BL_REVIEW_PENDING'
WF->>DB: INSERT case_event (APPROVAL_CREATED)
DB-->>UI: Show approval "Approve BL Draft" (PENDING)

UI->>DB: Manager decision approve/reject/edit (BL_DRAFT_APPROVAL)
DB-->>WF: Event: approval_decided(BL_DRAFT_APPROVAL)

alt approved
  WF->>DB: INSERT case_event (APPROVAL_APPROVED)
  WF->>DB: UPDATE documents SET status=VERIFIED
  WF->>DB: INSERT case_event (DOC_VERIFIED)
  WF->>N: Notify ZED: BL approved
  N-->>ZED: BL approved message
  WF->>DB: INSERT case_event (NOTIFICATION_SENT)
  WF->>DB: UPDATE cases SET state='BL_CONFIRMED'
  WF->>DB: INSERT case_event (STATE_CHANGED)
else rejected
  WF->>DB: INSERT case_event (APPROVAL_REJECTED)
  WF->>AI: Generate rework checklist (what to fix)
  AI-->>WF: Fix list
  WF->>N: Send fix list to ZED + Broker for review
  N-->>ZED: Fix request sent
  N-->>Broker: Fix request sent
  WF->>DB: INSERT case_event (NOTIFICATION_SENT)
  Note over WF,DB: State remains BL_REVIEW_PENDING (loop for new draft)
end
```

---

## Sequence 6 — Pre-Arrival: ETA-3 Weeks Tasks → POA Scan Verify → Original Confirm → Request Broker Docs

```mermaid
sequenceDiagram
autonumber
actor Client
actor Broker
participant UI as Manager UI (Next.js)
participant DB as Supabase (DB+Storage)
participant WF as n8n Workflow
participant AI as n8n AI Node
participant N as n8n Notification Node

Note over WF,DB: Trigger when ETA is known and (ETA - 21 days) reached
WF->>DB: UPDATE cases SET state='PRE_ARRIVAL_TASKS'
WF->>DB: INSERT case_event (STATE_CHANGED)
WF->>DB: Create tasks in computed.nba: clearance_instructions, POA_request

WF->>AI: Generate message to manager/client: request POA + clearance instructions
AI-->>WF: Draft tasks messages
WF->>N: Send POA request to Client (if required)
WF->>N: Notify internal delivery team (if applicable)
WF->>DB: INSERT case_event (NOTIFICATION_SENT)

Client-->>N: Sends POA scan
N-->>WF: Webhook: document_received(caseId, docType=POA, fileRef)
WF->>DB: Store POA scan in Storage
WF->>DB: INSERT documents (doc_type=POA, source=CLIENT, status=UPLOADED)
WF->>DB: INSERT case_event (DOC_UPLOADED, doc_type:POA, source:CLIENT)
WF->>DB: INSERT approvals (approval_type=POA_SCAN_APPROVAL)
WF->>DB: INSERT case_event (APPROVAL_CREATED)

DB-->>UI: Approval inbox shows "Verify POA scan"
UI->>DB: Manager approves/rejects POA scan
DB-->>WF: Event: approval_decided(POA_SCAN_APPROVAL)

alt approved
  WF->>DB: INSERT case_event (APPROVAL_APPROVED)
  WF->>DB: UPDATE documents SET status=VERIFIED
  WF->>DB: INSERT case_event (DOC_VERIFIED)
  WF->>DB: UPDATE cases SET state='POA_SCAN_VERIFIED'
  WF->>DB: INSERT case_event (STATE_CHANGED)
  WF->>AI: Generate message: confirm original POA shipment + requested docs list
  AI-->>WF: Draft confirmation message
  WF->>N: Send to Client: please send original POA + docs
  WF->>DB: INSERT case_event (EMAIL_SENT)
else rejected
  WF->>DB: INSERT case_event (APPROVAL_REJECTED)
  WF->>N: Ask Client to fix POA scan
  WF->>DB: INSERT case_event (EMAIL_SENT)
  Note over WF,DB: State remains PRE_ARRIVAL_TASKS (loop)
end

Client-->>N: Confirms original POA shipped / provides tracking
N-->>WF: Webhook: client_message(caseId, "original sent")
WF->>DB: UPDATE cases SET state='POA_ORIGINAL_CONFIRMED'
WF->>DB: INSERT case_event (STATE_CHANGED)

WF->>AI: Generate broker request for customs docs (ПП/ПД, ЗДП)
AI-->>WF: Broker request draft
WF->>N: Send to Broker: prepare PP/PD and ZDP
N-->>Broker: Request delivered
WF->>DB: INSERT case_event (EMAIL_SENT)
WF->>DB: UPDATE cases SET state='BROKER_DOCS_REQUESTED'
WF->>DB: INSERT case_event (STATE_CHANGED)
```

---

## Notes / Conventions

### Idempotency (важливо для PoC)

Для дій, які можуть повторитися через retry/webhook duplicates, використовуємо `idempotency_key`:

| Action | Idempotency Key Pattern |
|---|---|
| Send quote | `SEND_QUOTE:<caseId>:<quoteVersion>` |
| Create 1C deal | `CREATE_1C_DEAL:<caseId>` |
| Update spreadsheet | `UPDATE_SPREADSHEET:<caseId>:<rowId>` |
| Notify ZED | `NOTIFY_ZED:<caseId>:<dealNumber>` |

### Source of Truth

- Усі стани (`state`), approvals, документи та історія рішень — у **Supabase**.
- n8n не "тримає" стан, а виконує кроки та пише події/оновлення.
- UI тільки читає state та записує в дозволені поля (`payload.*`, approval decisions).

### Event Logging Convention

Кожен `case_event` має:
- `event_type` — канонічний тип з Core Event Taxonomy
- `actor_type` — `HUMAN` | `SYSTEM` | `AI` | `INTEGRATION`
- `metadata` — JSON зі специфічними даними
- `idempotency_key` — для дедуплікації (опціонально)
