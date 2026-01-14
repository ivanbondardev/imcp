# F1 Sea Import — Approval Gates (Human-in-the-Loop)

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [03_approval_pattern.md](../../core/03_approval_pattern.md) — патерн підтверджень  
- [02_core_data_model.md](../../core/02_core_data_model.md) — таблиця `approvals`

Approval Gates — це точки, де IMCP **не може рухатися далі без рішення людини**.
Вони забезпечують контроль, відповідальність і зменшують ризики.

> ⚠️ **Naming Convention:** Approval types використовують суфікс `_APPROVAL` згідно з Core.

---

## Gate List (PoC/MVP)

| approval_type | Назва | State, що тригерить | Ризик |
|---|---|---|---|
| `QUOTE_APPROVAL` | Підтвердити ціну | `QUOTE_APPROVAL_PENDING` | фінансовий |
| `ONEC_DEAL_CREATE_APPROVAL` | Створити угоду "імпорт" | `DEAL_IMPORT_PENDING` | системний/фінансовий |
| `DIMS_CHANGE_APPROVAL` | Підтвердити габарити | `DIMS_APPROVAL_PENDING` | документальний |
| `INSURANCE_APPROVAL` | Рішення про страхування | `INSURANCE_DECISION_PENDING` | фінансовий |
| `BL_DRAFT_APPROVAL` | Підтвердити BL draft | `BL_REVIEW_PENDING` | юридичний |
| `POA_SCAN_APPROVAL` | Підтвердити довіреність | `PRE_ARRIVAL_TASKS` | операційний |

---

## 1) QUOTE_APPROVAL

### Що затверджується
- фінальна ціна для клієнта
- умови та припущення
- коментар "ціна діє 7–9 днів"

### request_snapshot структура

```json
{
  "quote_version": 1,
  "cost_breakdown": {
    "sea_freight_usd": 1200,
    "warehouse_handling_usd": 200,
    "customs_usd": 150
  },
  "margin_usd": 250,
  "total_usd": 1800,
  "validity_days": 7,
  "assumptions": ["stackable cargo", "no dangerous goods"],
  "risks": ["dims not verified by warehouse yet"]
}
```

### Дії після рішення

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | `QUOTE_SENT` | система відправляє quote клієнту |
| APPROVED (edited) | `QUOTE_SENT` | система зберігає правки в decision_snapshot і відправляє |
| REJECTED | `QUOTE_READY` | кейс повертається на уточнення |

### Audit log (case_event)
- `event_type`: `APPROVAL_APPROVED` або `APPROVAL_REJECTED`
- `actor_type`: `HUMAN`
- `metadata`: `{ approval_id, has_edits: bool }`

---

## 2) ONEC_DEAL_CREATE_APPROVAL

### Що затверджується
- створення угоди "імпорт" в 1С на основі запиту

### request_snapshot структура

```json
{
  "onec_request_id": "REQ-123",
  "client_name": "Client LLC",
  "incoterms": "FOB",
  "route": {
    "origin": "Yiwu",
    "destination": "Kyiv"
  },
  "cargo_summary": {
    "packages": 10,
    "total_cbm": 3.2,
    "dangerous": false
  },
  "broker_owner": "OUR"
}
```

### Дії

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | `DEAL_IMPORT_CREATED` | n8n 1C node створює угоду → записує deal_number |
| REJECTED | `CONFIRMED` | зупинка + коментар |

---

## 3) DIMS_CHANGE_APPROVAL

### Коли виникає
- склад надав фактичні габарити, які відрізняються від заявлених клієнтом

### request_snapshot структура

```json
{
  "client_declared": {
    "packages": 10,
    "dimensions": [{"l": 60, "w": 40, "h": 50, "qty": 10}],
    "total_cbm": 1.2
  },
  "warehouse_measured": {
    "packages": 10,
    "dimensions": [{"l": 65, "w": 45, "h": 55, "qty": 10}],
    "total_cbm": 1.6
  },
  "difference_percent": 33,
  "price_impact_usd": 150,
  "document_impact": ["BL needs update", "Packing list needs update"]
}
```

### Дії

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | `DIMS_CONFIRMED` | `final_dims = warehouse_measured` |
| APPROVED (edited) | `DIMS_CONFIRMED` | менеджер може вручну скоригувати |
| REJECTED | `WAITING_WAREHOUSE_DIMS` | запит на уточнення складу/клієнта |

---

## 4) INSURANCE_APPROVAL

### Коли виникає
- менеджер узгоджує з клієнтом страхування вантажу

### request_snapshot структура

```json
{
  "insurance_required": true,
  "insurance_type": "STANDARD",
  "cargo_value_usd": 15000,
  "insurance_rate_percent": 0.5,
  "insurance_cost_usd": 75,
  "notes": "Client requested basic coverage"
}
```

### Дії

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | substate `INSURANCE_PENDING` removed | зафіксовано рішення |
| REJECTED | — | страхування не потрібне |

---

## 5) BL_DRAFT_APPROVAL

### Коли виникає
- ЗЕД надіслали драфт коносамента

### request_snapshot структура

```json
{
  "bl_document_id": "doc-456",
  "ai_verification": {
    "confidence": 0.85,
    "issues": [
      {"field": "buyer", "expected": "Client LLC", "actual": "Client LTD", "severity": "HIGH"}
    ]
  },
  "contract_reference": "doc-123",
  "invoice_reference": "doc-124",
  "final_dims": {
    "packages": 10,
    "total_cbm": 1.6
  }
}
```

### Дії

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | `BL_CONFIRMED` | повідомити ЗЕД/брокера "OK" |
| REJECTED | `BL_REVIEW_PENDING` (loop) | сформувати список правок і відправити |

---

## 6) POA_SCAN_APPROVAL

### Коли виникає
- клієнт заповнив довіреність і надіслав скан
- логісти просять підтвердити

### request_snapshot структура

```json
{
  "poa_document_id": "doc-789",
  "checklist": {
    "signature_present": true,
    "stamp_present": true,
    "company_details_correct": true,
    "dates_valid": true
  },
  "issues": []
}
```

### Дії

| Decision | Наступний state | Action |
|---|---|---|
| APPROVED | `POA_SCAN_VERIFIED` | перейти до підтвердження відправки оригіналу |
| REJECTED | `PRE_ARRIVAL_TASKS` (loop) | повернути клієнту на виправлення |

---

## Summary: Approval Types Mapping

| State | approval_type | on_approve | on_reject |
|---|---|---|---|
| `QUOTE_APPROVAL_PENDING` | `QUOTE_APPROVAL` | `QUOTE_SENT` | `QUOTE_READY` |
| `DEAL_IMPORT_PENDING` | `ONEC_DEAL_CREATE_APPROVAL` | `DEAL_IMPORT_CREATED` | `CONFIRMED` |
| `DIMS_APPROVAL_PENDING` | `DIMS_CHANGE_APPROVAL` | `DIMS_CONFIRMED` | `WAITING_WAREHOUSE_DIMS` |
| `INSURANCE_DECISION_PENDING` | `INSURANCE_APPROVAL` | (remove substate) | — |
| `BL_REVIEW_PENDING` | `BL_DRAFT_APPROVAL` | `BL_CONFIRMED` | `BL_REVIEW_PENDING` |
| `PRE_ARRIVAL_TASKS` | `POA_SCAN_APPROVAL` | `POA_SCAN_VERIFIED` | `PRE_ARRIVAL_TASKS` |
