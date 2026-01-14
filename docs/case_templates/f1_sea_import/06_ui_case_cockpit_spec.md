# F1 Sea Import — UI Case Cockpit Spec (Next.js)

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [04_case_cockpit_ux.md](../../core/04_case_cockpit_ux.md) — UX/UI патерни  
- [01_architecture_overview.md](../../core/01_architecture_overview.md) — UI Contract

Цей документ описує, як має виглядати "кабіна менеджера" для кейса F1 Sea Import.

> ⚠️ **UI Contract:** UI має чітко обмежені права на зміну даних (див. Core 01_architecture_overview).

---

## 1) Page Layout (Single Pane of Glass)

Згідно з Core UX Pattern — "Case Cockpit":

```
┌─────────────────────────────────────────────────┐
│ CASE HEADER (ID, state, status, SLA, owner)     │
├─────────────────────────────────────────────────┤
│ STICKY NEXT BEST ACTION                         │
├───────────────────────────┬─────────────────────┤
│ MAIN WORK AREA (8/12)     │ CONTEXT SIDEBAR     │
│                           │ (4/12)              │
│ - Draft view              │ - Cargo summary     │
│ - Form view               │ - Route info        │
│ - Document review         │ - Risk flags        │
│ - Approval detail         │ - Quick links       │
├───────────────────────────┴─────────────────────┤
│ TIMELINE / EVENT LOG                            │
└─────────────────────────────────────────────────┘
```

### Header

| Елемент | Джерело | Приклад |
|---|---|---|
| Case ID | `cases.case_number` | `F1-SEA-2026-00123` |
| Case Type Badge | `cases.case_type` | `F1_SEA_IMPORT` |
| State Badge | `cases.state` | `QUOTE_APPROVAL_PENDING` |
| Status Indicator | `cases.status` | `OPEN` (зелений) / `BLOCKED` (жовтий) |
| SLA / Deadline | `cases.sla_deadline` | `2d 4h remaining` |
| Owner | `cases.owner_user_id` | Avatar + name |
| Priority | `cases.priority` | 🔴 (urgent) / 🟡 (high) / ⚪ (normal) |
| Key IDs | `payload.integration.*` | 1C request, deal number, marking code |

---

## 2) Main Blocks

### 2.1 Case Summary (Context Sidebar)

| Section | Дані з payload | Приклад |
|---|---|---|
| 📦 Cargo | `payload.cargo.*` | 10 pcs, 3.2 cbm, stackable: Yes |
| 🚚 Route | `payload.route.*` | Yiwu → Kyiv, Terminal X |
| 👤 Client | `payload.client.*` | Client LLC, john@client.com |
| 🏢 Broker | `payload.broker.*` | Our broker |
| ⚠️ Risk Flags | `computed.risks[]` | `DANGEROUS_GOODS`, `DIMS_MISMATCH` |
| 📅 Dates | `payload.cargo.ready_date` | Ready: 2026-02-10 |

### 2.2 Required Fields Checklist

Показати ключові поля + статус:

| Field | Status | Source |
|---|---|---|
| Client name | ✅ filled | `payload.client.name` |
| Warehouse | ✅ filled | `payload.route.origin_warehouse` |
| Cargo ready date | ✅ filled | `payload.cargo.ready_date` |
| Dimensions | ⚠️ needs verification | `payload.cargo.dimensions` |
| Broker | ✅ filled | `payload.broker.owner` |
| Dangerous goods | ✅ no | `payload.cargo.dangerous_goods` |

### 2.3 Next Best Action (NBA)

Згідно з Core — sticky component:

| Компонент | Джерело |
|---|---|
| Title | `computed.nba.title` |
| Description | `computed.nba.description` |
| Approval Required? | `computed.nba.approval_required` |
| Approval ID | `computed.nba.approval_id` |
| Primary CTA | Approve / Continue / Request Info |
| Secondary CTA | Edit / Postpone / View Details |

**NBA Variants за state:**

| State | NBA Title | CTA |
|---|---|---|
| NEW | Запросити дані клієнта | [Request Client Info] |
| CLIENT_INFO_COLLECTED | Створити запит у 1С | [Create 1C Request] |
| QUOTE_READY | Створити approval на ціну | Auto (approval created) |
| QUOTE_APPROVAL_PENDING | Підтвердити калькуляцію | [Approve] [Edit] [Reject] |
| CONFIRMED | Створити угоду в 1С | [Approve Deal Creation] |
| DIMS_APPROVAL_PENDING | Підтвердити габарити | [Approve] [Edit] [Reject] |
| BL_REVIEW_PENDING | Перевірити BL draft | [Approve] [Reject] |

### 2.4 Approvals Inbox

Список pending approvals (з таблиці `approvals` де `status='PENDING'`):

| approval_type | request_snapshot | Actions |
|---|---|---|
| `QUOTE_APPROVAL` | Quote breakdown, risks | [Approve] [Edit] [Reject] |
| `ONEC_DEAL_CREATE_APPROVAL` | Deal payload preview | [Approve] [Reject] |
| `DIMS_CHANGE_APPROVAL` | Client vs Warehouse diff | [Approve] [Edit] [Reject] |
| `BL_DRAFT_APPROVAL` | Diff report, AI issues | [Approve] [Reject] |
| `POA_SCAN_APPROVAL` | Document preview, checklist | [Approve] [Reject] |

### 2.5 Documents

Список документів (з таблиці `documents`):

| doc_type | Status | Source | Actions |
|---|---|---|---|
| `CONTRACT` | VERIFIED | CLIENT | [View] [Replace] |
| `INVOICE` | UPLOADED | ZED | [View] [Verify] |
| `BL_DRAFT` | PROCESSING | ZED | [View] (in approval) |
| `POA` | VERIFIED | CLIENT | [View] |

### 2.6 Timeline / Events

Потік подій (з таблиці `case_events`):

| Timestamp | Actor | Event | Details |
|---|---|---|---|
| 12:10 | SYSTEM | `STATE_CHANGED` | NEW → WAITING_CLIENT_INFO |
| 11:42 | HUMAN | `APPROVAL_APPROVED` | QUOTE_APPROVAL |
| 10:30 | AI | `DOC_EXTRACTED` | BL_DRAFT, confidence: 85% |
| 09:55 | HUMAN | `CASE_CREATED` | — |

---

## 3) UX Rules (згідно Core)

### Що UI показує завжди:

| # | Питання | Відповідь UI |
|---|---|---|
| 1 | Де ми зараз? | State badge + Status indicator |
| 2 | Що робити далі? | NBA component |
| 3 | Що зробила система? | Timeline / Events |
| 4 | Що я можу зробити? | CTAs based on state |
| 5 | Що блокує перехід? | Risk flags + Missing fields |

### UI Contract (що UI може/не може):

| ✅ UI МОЖЕ | ❌ UI НЕ МОЖЕ |
|---|---|
| UPDATE `cases.payload.*` | UPDATE `cases.state` |
| INSERT `approvals` (status=PENDING) | UPDATE `cases.status` |
| UPDATE `approvals.decision_*` (якщо PENDING) | UPDATE `cases.computed` |
| INSERT `documents` | DELETE `case_events` |
| INSERT `case_events` (actor_type=HUMAN) | Запускати workflow напряму |

---

## 4) Context Sidebar Panels

### Cargo Summary Panel

```
📦 Cargo Summary
├── Packages: 10
├── Declared dims: 3.2 cbm
├── Warehouse dims: 3.6 cbm (⚠️ mismatch)
├── Stackable: Yes
├── Dangerous: No
└── Description: Consumer goods
```

### Route Panel

```
🚚 Route & Logistics
├── Origin: Yiwu Warehouse
├── Port: Shenzhen
├── Destination: Kyiv, Terminal X
├── Unloading: Client warehouse
└── Broker: Our broker
```

### Risk Flags Panel

```
⚠️ Risk Flags
├── 🔴 DIMS_MISMATCH: Warehouse dims differ by 33%
├── 🟡 LOW_CONFIDENCE: AI extraction 76% on warehouse field
└── 🟢 No dangerous goods declared
```

---

## 5) PoC UI (мінімальна версія)

Для PoC достатньо:

| Component | Priority | Notes |
|---|---|---|
| Case page | P0 | Single page with all sections |
| Intake form (7 fields) | P0 | Edit payload.client/cargo/route |
| Approvals list | P0 | Show pending, allow decision |
| Documents list | P0 | Upload, view, status |
| Timeline | P1 | Read-only event log |
| Risk flags | P1 | Display computed.risks |
| Context sidebar | P2 | Summary cards |

---

## 6) ASCII Wireframe (згідно Core)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Case #F1-SEA-02451        STATE: QUOTE_APPROVAL_PENDING   SLA: 2d 4h  │
│ Client: ACME LTD     Route: CN → UA (SEA)        Owner: Ivan      Priority: 🔴│
│ Status: OPEN                                                                  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ 🔴 NEXT BEST ACTION                                                          │
│ ─────────────────────────────────────────────────────────────────────────────│
│ Approve Quote (QUOTE_APPROVAL)                                               │
│ • System prepared price calculation                                          │
│ • Price validity: 7–9 days                                                   │
│ • Risk: client-declared dimensions not yet verified                          │
│                                                                              │
│ [ Approve ]   [ Edit & Approve ]   [ Reject ]                                │
└──────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────┬──────────────────────────────────────────────┐
│ MAIN WORK AREA                │ CONTEXT SIDEBAR                              │
│                               │                                              │
│ ┌───────────────────────────┐ │ ┌──────────────────────────────────────────┐ │
│ │ 💰 Quote Draft            │ │ │ 📦 Cargo Summary                         │ │
│ │                           │ │ │ • Packages: 12                           │ │
│ │ Base cost:      $2,450    │ │ │ • Declared dims: 3.2 cbm                 │ │
│ │ Margin:           $350    │ │ │ • Stackable: Yes                         │ │
│ │ Total:          $2,800    │ │ │ • Dangerous: No                          │ │
│ │                           │ │ └──────────────────────────────────────────┘ │
│ │ Assumptions:              │ │                                              │
│ │ • Stackable cargo         │ │ ┌──────────────────────────────────────────┐ │
│ │ • No dangerous goods      │ │ │ 🚚 Route                                 │ │
│ │                           │ │ │ • Origin: Yiwu Warehouse                 │ │
│ │ Validity: 7–9 days        │ │ │ • Destination: Kyiv, Terminal X          │ │
│ └───────────────────────────┘ │ │ • Broker: Our                            │ │
│                               │ └──────────────────────────────────────────┘ │
│                               │                                              │
│                               │ ┌──────────────────────────────────────────┐ │
│                               │ │ ⚠️ Risk Flags                            │ │
│                               │ │ • Dimensions not verified                │ │
│                               │ └──────────────────────────────────────────┘ │
└───────────────────────────────┴──────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ 🕓 TIMELINE                                                                  │
│ ─────────────────────────────────────────────────────────────────────────────│
│ 12:10  SYSTEM   STATE_CHANGED: QUOTE_READY → QUOTE_APPROVAL_PENDING          │
│ 12:10  SYSTEM   APPROVAL_CREATED: QUOTE_APPROVAL                             │
│ 11:42  HUMAN    STATE_CHANGED: CLIENT_INFO_COLLECTED → QUOTE_READY           │
│ 10:30  AI       AI_RUN_COMPLETED: EXTRACT, confidence: 92%                   │
│ 09:55  HUMAN    CASE_CREATED                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```
