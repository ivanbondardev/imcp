# F1 Sea Import — Automation & AI Rules

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [01_architecture_overview.md](../../core/01_architecture_overview.md) — UI Contract, n8n роль  
- [02_core_data_model.md](../../core/02_core_data_model.md) — Event Taxonomy  
- [03_approval_pattern.md](../../core/03_approval_pattern.md) — Draft-First Pattern

Цей документ описує:
- що система робить автоматично
- що робить AI (витяг/генерація/перевірка)
- які винятки підсвічуються як ризик

> ⚠️ **Термінологія:**  
> - `state` — бізнес-стан (змінюється n8n)  
> - `status` — технічний агрегат (`OPEN`, `BLOCKED`, `DONE`, `ARCHIVED`)

---

## 1) Automation Rules (без AI або з мінімальним AI)

### 1.1 Intake Request Automation

**Trigger:** менеджер натискає "Request Client Info"  
**Action:**
- створити шаблон запиту
- відправити клієнту питання (7 пунктів)
- оновити `cases.state = 'WAITING_CLIENT_INFO'`
- додати `case_event`: `STATE_CHANGED` з `{from_state: 'NEW', to_state: 'WAITING_CLIENT_INFO'}`

---

### 1.2 Quote Validity Timer

**Trigger:** quote відправлено (state = `QUOTE_SENT`)  
**Action:**
- записати `computed.quote.valid_until = now + 7 days` (з config `thresholds.quote_validity_days`)
- додати task до `computed.tasks[]`: `{code: 'QUOTE_FOLLOWUP', title: 'Перевірити актуальність ціни'}`
- якщо клієнт не відповів — nотифікація менеджеру

---

### 1.3 After Confirmation Automation

**Trigger:** клієнт підтвердив перевезення (state = `CONFIRMED`)  
**Action:**
1. Оновити state → `DEAL_IMPORT_PENDING`
2. Створити approval `ONEC_DEAL_CREATE_APPROVAL`
3. Після затвердження та створення угоди:
   - відправити клієнту адресу складу Yiwu/Shenzhen
   - відправити маркування `ALEX-<deal_number>`
   - заповнити таблицю "Морські вантажі ВП"
   - повідомити ЗЕД про очікуваний вантаж + таблицю

**Events (згідно Core Event Taxonomy):**
- `STATE_CHANGED`: `{from_state: 'CONFIRMED', to_state: 'DEAL_IMPORT_PENDING'}`
- `APPROVAL_CREATED`: `{approval_id, approval_type: 'ONEC_DEAL_CREATE_APPROVAL'}`
- `INTEGRATION_SUCCESS`: `{integration_id, external_id: deal_number}`
- `EMAIL_SENT`: `{to: client, subject: 'Warehouse instructions'}`

---

### 1.4 Warehouse Dims Notification

**Trigger:** склад надіслав фактичні габарити (webhook)  
**Action:**
- записати dims в `payload.cargo.warehouse_*`
- порівняти з `payload.cargo.dimensions`
- якщо mismatch → створити approval `DIMS_CHANGE_APPROVAL`
- після підтвердження → повідомити клієнта фактичні dims

**Events:**
- `DOC_UPLOADED` (якщо прикріплено документ)
- `APPROVAL_CREATED`: `{approval_type: 'DIMS_CHANGE_APPROVAL'}` (якщо mismatch)

---

### 1.5 ETA-3 Weeks Pre-Arrival Tasks

**Trigger:** estimated arrival date - 21 день (або вручну)  
**Action:**
- оновити state → `PRE_ARRIVAL_TASKS`
- додати tasks до `computed.tasks[]`:
  ```json
  [
    {"code": "CLEARANCE_INSTRUCTIONS", "title": "Надати інструкцію з оформлення", "status": "PENDING"},
    {"code": "POA_REQUEST", "title": "Отримати довіреність", "status": "PENDING"}
  ]
  ```
- оновити `computed.nba` згідно з `nba_by_state.PRE_ARRIVAL_TASKS`
- нагадування менеджеру

**Events:**
- `STATE_CHANGED`: `{from_state: 'BL_CONFIRMED', to_state: 'PRE_ARRIVAL_TASKS'}`
- `NOTIFICATION_SENT`: `{channel: 'internal', type: 'task_reminder'}`

---

### 1.6 Broker Docs Request

**Trigger:** POA original confirmed (state = `POA_ORIGINAL_CONFIRMED`)  
**Action:**
- автоматично сформувати запит брокеру: підготувати ПП/ПД та ЗДП
- оновити state → `BROKER_DOCS_REQUESTED`

**Events:**
- `EMAIL_SENT`: `{to: broker, subject: 'Prepare customs docs'}`
- `STATE_CHANGED`

---

## 2) AI Rules (Jarvis functions)

### 2.1 Extract Client Answers → Structured Fields

**Input:** лист/чат від клієнта  
**Output:**
- incoterms
- warehouse (Yiwu/Shenzhen)
- ready date
- dims + packages
- stackable
- terminal + unloading point
- broker owner
- dangerous flag + cargo description

**Behavior:**
- якщо confidence low → додати `computed.risks[]` з `{code: 'LOW_CONFIDENCE', field: '...'}`
- якщо missing fields → авто-запит уточнення

**Events:**
- `AI_RUN_COMPLETED`: `{ai_run_id, run_type: 'EXTRACT', confidence}`

---

### 2.2 Generate Draft Messages

AI генерує чернетки (run_type = `GENERATE`):
- запит даних у клієнта (7 питань)
- комерційна пропозиція (quote email)
- повідомлення складу (маркування)
- повідомлення ЗЕД
- запит телекс-релізу
- запит брокеру

**Правило:** AI генерує → людина підтверджує відправку (draft-first)

---

### 2.3 Verify BL Draft vs Contract/Invoice

**Input:** BL draft + контракт + інвойс  
**Output:** diff-report:
- seller/buyer mismatch
- cargo description mismatch
- missing fields
- suspicious values

**Events:**
- `AI_RUN_COMPLETED`: `{run_type: 'VERIFY', confidence}`
- `AI_SUGGESTION_CREATED`: `{field, suggested_value, confidence}`

**Rule:** завжди створюється approval `BL_DRAFT_APPROVAL`

---

## 3) Exception / Red Flags (Risk Taxonomy)

> **Примітка:** Ризики зберігаються в `computed.risks[]` згідно з Core schema.

| Flag Code | Умова | Severity | Дія системи |
|---|---|---|---|
| `DANGEROUS_GOODS` | `payload.cargo.dangerous_goods=true` | HIGH | підняти пріоритет, додаткові питання |
| `DIMS_MISMATCH` | dims diff > threshold | MEDIUM | approval gate `DIMS_CHANGE_APPROVAL` |
| `BROKER_UNKNOWN` | `payload.broker.owner` missing | HIGH | блок переходів далі |
| `EXPORT_DECLARATION_REQUIRED` | брокер каже "потрібна" | MEDIUM | позначити, додати tasks |
| `BL_DIFF_FOUND` | AI/manager знайшов mismatch | HIGH | reject BL + список правок |
| `TELEX_NOT_REQUESTED` | BL confirmed, але немає telex-release | MEDIUM | нагадування/задача |
| `LOW_CONFIDENCE` | AI extraction confidence < 0.7 | LOW | потребує верифікації людиною |
| `MISSING_DATA` | required field is null | MEDIUM | блок автоматичних переходів |

### Risk object schema (згідно Core)

```json
{
  "code": "DIMS_MISMATCH",
  "severity": "MEDIUM",
  "message": "Warehouse dims differ from client declared by 33%",
  "detected_at": "2026-01-14T10:00:00Z",
  "detected_by": "SYSTEM"
}
```

---

## 4) Non-negotiable Rules

> ⚠️ **Критичний контракт:** Порушення цих правил — це архітектурний дефект.

ШІ/Система **не має права**:
- ❌ підтвердити BL самостійно
- ❌ створити фінансове зобов'язання без менеджера
- ❌ "відправити" критичний документ без approval
- ❌ змінювати `cases.state` напряму з UI (тільки n8n)
- ❌ записувати в `cases.computed` з UI

---

## 5) Tasks vs NBA (розмежування)

> ⚠️ **Контракт:** NBA та Tasks — різні концепції.

| Концепт | Призначення | Зберігається в |
|---|---|---|
| **NBA** | Одна головна дія для поточного state | `computed.nba` |
| **Tasks** | Список паралельних задач | `computed.tasks[]` |

### NBA (Next Best Action)
- **Завжди одна** дія для state
- Розраховується n8n при кожній зміні state
- Джерело: `case_type_config.nba_by_state`

### Tasks
- **Може бути декілька** одночасно
- Можуть існувати паралельно з різними states
- Кожна task має `status`: `PENDING` → `DONE` / `CANCELLED`

---

## 6) Event Types Summary (згідно Core Event Taxonomy)

| Категорія | Типи подій для F1_SEA_IMPORT |
|---|---|
| **Core** | `CASE_CREATED`, `STATE_CHANGED` |
| **Approvals** | `APPROVAL_CREATED`, `APPROVAL_APPROVED`, `APPROVAL_REJECTED`, `APPROVAL_CANCELLED` |
| **Documents** | `DOC_UPLOADED`, `DOC_EXTRACTED`, `DOC_VERIFIED`, `DOC_REPLACED` |
| **Integration** | `INTEGRATION_STARTED`, `INTEGRATION_SUCCESS`, `INTEGRATION_FAILED` |
| **AI** | `AI_RUN_COMPLETED`, `AI_SUGGESTION_CREATED` |
| **Communication** | `EMAIL_SENT`, `NOTIFICATION_SENT`, `COMMENT_ADDED` |

> **Note:** `SUBSTATE_ADDED` / `SUBSTATE_REMOVED` заплановані для P2 (parallel substates).
