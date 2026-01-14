# F1 Sea Import — States & Flow (State Machine)

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [02_core_data_model.md](../../core/02_core_data_model.md) — State vs Status термінологія  
- [03_approval_pattern.md](../../core/03_approval_pattern.md) — патерн підтверджень

> ⚠️ **Термінологія згідно Core:**  
> - `state` — бізнес-стан у state machine (напр. `WAITING_CLIENT_INFO`)  
> - `status` — технічний агрегат (`OPEN`, `BLOCKED`, `DONE`, `ARCHIVED`)

Цей документ описує процес як набір **станів (states) кейса** + переходів.
Мета — щоб IMCP міг:
- показувати менеджеру "де ми зараз"
- підказувати "що далі"
- автоматизувати рутину
- виносити критичні рішення на approval gates

---

## 1) State List (Primary States)

> **Примітка:** Це бізнес-стани (`cases.state`). Технічний статус (`cases.status`) розраховується автоматично як `OPEN`, `BLOCKED`, `DONE` або `ARCHIVED`.

| State Code | Назва стану | Що означає | Owner |
|---|---|---|---|
| NEW | Новий запит | Кейс створено, дані ще не зібрані | Manager |
| WAITING_CLIENT_INFO | Очікуємо дані від клієнта | Надіслали питання, чекаємо відповіді | Manager + System |
| CLIENT_INFO_COLLECTED | Дані зібрані | 7 Core Questions отримані | System |
| REQUEST_1C_CREATED | Запит у 1С створено | Є запит у 1С, можна рахувати | Manager |
| QUOTE_READY | Чернетка ціни готова | Є розрахунок + маржа | System |
| QUOTE_APPROVAL_PENDING | Очікує підтвердження ціни | Approval gate: `QUOTE_APPROVAL` | Manager |
| QUOTE_SENT | Ціну відправлено | Клієнту відправлено quote | System |
| WAITING_CLIENT_CONFIRMATION | Очікуємо підтвердження | Клієнт має сказати "так/ні" | Manager |
| CONFIRMED | Підтверджено | Клієнт підтвердив перевезення | Manager |
| DEAL_IMPORT_PENDING | Чернетка угоди "імпорт" | Approval gate: `ONEC_DEAL_CREATE_APPROVAL` | System |
| DEAL_IMPORT_CREATED | Угоду створено | В 1С створено угоду "імпорт" | System |
| WAREHOUSE_INSTRUCTIONS_SENT | Інструкції складу відправлені | Адреса + маркування ALEX-... | System |
| VP_TABLE_UPDATED | Таблицю заповнено | "Морські вантажі ВП" оновлено | System |
| ZED_NOTIFIED | ЗЕД повідомлено | Передано очікуваний вантаж + таблицю | System |
| WAITING_WAREHOUSE_DIMS | Очікуємо фактичні габарити | Чекаємо підтвердження від складу | System |
| WAREHOUSE_DIMS_RECEIVED | Габарити отримані | Фактичні дані зафіксовані | System |
| DIMS_APPROVAL_PENDING | Потрібне підтвердження змін | Approval gate: `DIMS_CHANGE_APPROVAL` | Manager |
| DIMS_CONFIRMED | Габарити підтверджені | Вони стануть базою для BL/PL | Manager |
| BL_DRAFT_RECEIVED | Драфт BL отримано | ЗЕД надіслали чернетку | System |
| BL_REVIEW_PENDING | BL на перевірці | Approval gate: `BL_DRAFT_APPROVAL` | Manager |
| BL_CONFIRMED | BL підтверджено | Дані звірені з контрактом/інвойсом | Manager |
| PRE_ARRIVAL_TASKS | Підготовка до прибуття | ETA-3 weeks tasks | System/Manager |
| POA_SCAN_VERIFIED | Скан довіреності перевірено | Approval gate: `POA_SCAN_APPROVAL` | Manager |
| POA_ORIGINAL_CONFIRMED | Оригінал підтверджено | Відправлення оригіналу підтверджено | Manager |
| BROKER_DOCS_REQUESTED | Запит брокеру | ПП/ПД та ЗДП | Manager |
| READY_FOR_CLEARANCE | Готово до оформлення | Пакет документів готовий | Manager |
| CLOSED | Закрито | Кейс завершено в межах шаблону | System |

---

## 2) Entry Data Requirements

> **Канонічний список:** див. [02_required_data_and_documents.md](./02_required_data_and_documents.md#1-client-intake-fields--канонічний-список)

Щоб перейти до `CLIENT_INFO_COLLECTED`, потрібно мати **7 Core Questions**:

| # | Field | Required |
|---|---|---|
| 1 | `origin_warehouse` | ✅ |
| 2 | `cargo_ready_date` | ✅ |
| 3 | `packages_count` | ✅ |
| 4 | `dimensions[]` | ✅ |
| 5 | `stackable` | ✅ |
| 6 | `destination_terminal` | ✅ |
| 7 | `broker_owner` | ✅ |

Safety fields (`dangerous_goods`, `cargo_description`, `incoterms`) не блокують перехід, але додають risk flags якщо відсутні.

---

## 3) Transitions (основні переходи)

| From | To | Trigger/Event | Notes |
|---|---|---|---|
| NEW | WAITING_CLIENT_INFO | Manager clicks "Request Client Info" | система генерує/відправляє питання |
| WAITING_CLIENT_INFO | CLIENT_INFO_COLLECTED | All 7 Core Questions extracted | може бути ручне підтвердження |
| CLIENT_INFO_COLLECTED | REQUEST_1C_CREATED | Manager confirms & creates 1C request | або n8n node |
| REQUEST_1C_CREATED | QUOTE_READY | Pricing calculation finished | включає маржу |
| QUOTE_READY | QUOTE_APPROVAL_PENDING | Create approval: `QUOTE_APPROVAL` | ворота |
| QUOTE_APPROVAL_PENDING | QUOTE_SENT | Approval approved | відправка клієнту |
| QUOTE_SENT | WAITING_CLIENT_CONFIRMATION | Auto | очікуємо відповідь |
| WAITING_CLIENT_CONFIRMATION | CONFIRMED | Client says YES | фіксуємо |
| CONFIRMED | DEAL_IMPORT_PENDING | System prepares payload | чернетка |
| DEAL_IMPORT_PENDING | DEAL_IMPORT_CREATED | Approval `ONEC_DEAL_CREATE_APPROVAL` + 1C node success | створено угоду |
| DEAL_IMPORT_CREATED | WAREHOUSE_INSTRUCTIONS_SENT | Auto | адреса + маркування |
| WAREHOUSE_INSTRUCTIONS_SENT | VP_TABLE_UPDATED | Auto | таблиця "Морські вантажі ВП" |
| VP_TABLE_UPDATED | ZED_NOTIFIED | Auto | повідомити ЗЕД |
| ZED_NOTIFIED | WAITING_WAREHOUSE_DIMS | Auto | чекаємо склад |
| WAITING_WAREHOUSE_DIMS | WAREHOUSE_DIMS_RECEIVED | Warehouse dims webhook | факт |
| WAREHOUSE_DIMS_RECEIVED | DIMS_APPROVAL_PENDING | If mismatch > 10% | ворота `DIMS_CHANGE_APPROVAL` |
| WAREHOUSE_DIMS_RECEIVED | DIMS_CONFIRMED | If no mismatch | одразу |
| DIMS_APPROVAL_PENDING | DIMS_CONFIRMED | Manager approves dims | зафіксовано |
| DIMS_CONFIRMED | BL_DRAFT_RECEIVED | ZED sends BL draft | документ |
| BL_DRAFT_RECEIVED | BL_REVIEW_PENDING | Auto create approval `BL_DRAFT_APPROVAL` | ворота |
| BL_REVIEW_PENDING | BL_CONFIRMED | Approved | підтверджено |
| BL_CONFIRMED | PRE_ARRIVAL_TASKS | ETA-3 weeks trigger | follow-up |
| PRE_ARRIVAL_TASKS | POA_SCAN_VERIFIED | Approval `POA_SCAN_APPROVAL` approved | |
| POA_SCAN_VERIFIED | POA_ORIGINAL_CONFIRMED | Original sent confirmed | |
| POA_ORIGINAL_CONFIRMED | BROKER_DOCS_REQUESTED | Request broker docs | ПП/ПД, ЗДП |
| BROKER_DOCS_REQUESTED | READY_FOR_CLEARANCE | Docs prepared | |
| READY_FOR_CLEARANCE | CLOSED | Manager closes case | |

---

## 4) Parallel Processes (PoC: via Tasks)

> ⚠️ **PoC Note:** Substates (`cases.substates[]`) заплановані для P2. У PoC паралельні процеси моделюються через `computed.tasks[]`.

### Паралельні задачі, що можуть виконуватись разом з primary state:

| Task Code | Опис | Паралельно з |
|---|---|---|
| `INSURANCE_DECISION` | Рішення про страхування | Будь-який state після CONFIRMED |
| `TELEX_RELEASE_REQUEST` | Запит телекс-релізу | BL_CONFIRMED, PRE_ARRIVAL_TASKS |
| `EXPORT_DECLARATION_CHECK` | Перевірка експортної декларації | WAITING_WAREHOUSE_DIMS+ |
| `BROKER_DOCS_FOLLOWUP` | Follow-up до брокера | BROKER_DOCS_REQUESTED |

### Task object schema:

```typescript
interface Task {
  code: string;           // 'INSURANCE_DECISION'
  title: string;          // 'Визначити страхування'
  status: 'PENDING' | 'DONE' | 'CANCELLED';
  created_at: string;
  completed_at?: string;
  related_approval_id?: string;  // Якщо task → approval
}
```

---

## 5) Case Type Config (JSON для Supabase)

> ⚠️ **Контракт:** Ця конфігурація використовується n8n для валідації переходів.

```json
{
  "case_type": "F1_SEA_IMPORT",
  "display_name": "Морський імпорт (Китай → Україна)",
  "version": 1,
  
  "initial_state": "NEW",
  "final_states": ["CLOSED", "CANCELLED"],
  
  "required_fields_by_state": {
    "CLIENT_INFO_COLLECTED": [
      "payload.route.origin_warehouse",
      "payload.cargo.ready_date",
      "payload.cargo.packages_count",
      "payload.cargo.dimensions",
      "payload.cargo.stackable",
      "payload.route.destination_terminal",
      "payload.broker.owner"
    ],
    "QUOTE_READY": [
      "computed.quote.total_usd"
    ],
    "DIMS_CONFIRMED": [
      "payload.cargo.final_dimensions"
    ]
  },
  
  "approval_map": {
    "QUOTE_APPROVAL_PENDING": {
      "approval_type": "QUOTE_APPROVAL",
      "on_approve": "QUOTE_SENT",
      "on_reject": "QUOTE_READY"
    },
    "DEAL_IMPORT_PENDING": {
      "approval_type": "ONEC_DEAL_CREATE_APPROVAL",
      "on_approve": "DEAL_IMPORT_CREATED",
      "on_reject": "CONFIRMED"
    },
    "DIMS_APPROVAL_PENDING": {
      "approval_type": "DIMS_CHANGE_APPROVAL",
      "on_approve": "DIMS_CONFIRMED",
      "on_reject": "WAITING_WAREHOUSE_DIMS"
    },
    "BL_REVIEW_PENDING": {
      "approval_type": "BL_DRAFT_APPROVAL",
      "on_approve": "BL_CONFIRMED",
      "on_reject": "BL_DRAFT_RECEIVED"
    },
    "PRE_ARRIVAL_TASKS": {
      "approval_type": "POA_SCAN_APPROVAL",
      "on_approve": "POA_SCAN_VERIFIED",
      "on_reject": "PRE_ARRIVAL_TASKS"
    }
  },
  
  "nba_by_state": {
    "NEW": {
      "action_code": "REQUEST_CLIENT_INFO",
      "title": "Запросити дані клієнта",
      "approval_required": false
    },
    "WAITING_CLIENT_INFO": {
      "action_code": "WAIT_OR_FOLLOWUP",
      "title": "Очікуємо відповідь клієнта",
      "approval_required": false
    },
    "CLIENT_INFO_COLLECTED": {
      "action_code": "CREATE_1C_REQUEST",
      "title": "Створити запит у 1С",
      "approval_required": false
    },
    "QUOTE_APPROVAL_PENDING": {
      "action_code": "APPROVE_QUOTE",
      "title": "Підтвердити калькуляцію",
      "approval_required": true
    },
    "DEAL_IMPORT_PENDING": {
      "action_code": "APPROVE_DEAL",
      "title": "Створити угоду в 1С",
      "approval_required": true
    },
    "DIMS_APPROVAL_PENDING": {
      "action_code": "APPROVE_DIMS",
      "title": "Підтвердити габарити",
      "approval_required": true
    },
    "BL_REVIEW_PENDING": {
      "action_code": "APPROVE_BL",
      "title": "Перевірити BL draft",
      "approval_required": true
    }
  },
  
  "thresholds": {
    "dims_mismatch_cbm_percent": 10,
    "dims_mismatch_packages_triggers": true,
    "dims_stackable_change_triggers": true,
    "ai_auto_accept_confidence": 0.85,
    "ai_needs_review_confidence": 0.70,
    "quote_validity_days": 7
  }
}
```
