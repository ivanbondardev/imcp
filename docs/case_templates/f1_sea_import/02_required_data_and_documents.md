# F1 Sea Import — Required Data & Documents

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [02_core_data_model.md](../../core/02_core_data_model.md) — JSONB Contracts (payload/computed)

Цей документ описує:
- які дані потрібні для кейса
- звідки вони беруться
- які документи є ключовими
- які правила валідації важливі

---

## 1) Client Intake Fields — Канонічний список

> ⚠️ **Контракт:** Це єдиний канонічний список полів intake. Усі інші документи посилаються на цю таблицю.

### 1.1 Обов'язкові поля (7 Core Questions)

Без цих полів **неможливий перехід** до `CLIENT_INFO_COLLECTED`:

| # | Field Code | Питання | Тип | Приклад | Payload Path |
|---|---|---|---|---|---|
| 1 | `origin_warehouse` | Склад Yiwu чи Shenzhen? | enum | `YIWU` | `payload.route.origin_warehouse` |
| 2 | `cargo_ready_date` | Дата готовності вантажу | date | `2026-02-10` | `payload.cargo.ready_date` |
| 3 | `packages_count` | Кількість місць | int | `10` | `payload.cargo.packages_count` |
| 4 | `dimensions` | Розміри місць | array | `[{l,w,h,qty}]` | `payload.cargo.dimensions[]` |
| 5 | `stackable` | Штабелюється? | boolean | `true` | `payload.cargo.stackable` |
| 6 | `destination_terminal` | Термінал у Києві | text | `Terminal X` | `payload.route.destination_terminal` |
| 7 | `broker_owner` | Брокер чий? | enum | `OUR` / `CLIENT` | `payload.broker.owner` |

### 1.2 Safety & Context поля (обов'язкові, але не блокують)

Ці поля **потрібні для повноти**, але їх відсутність не блокує перехід (додається risk flag):

| # | Field Code | Питання | Тип | Приклад | Payload Path |
|---|---|---|---|---|---|
| 8 | `dangerous_goods` | Небезпечний вантаж? | boolean | `false` | `payload.cargo.dangerous_goods` |
| 9 | `cargo_description` | Короткий опис вантажу | text | `Electronics` | `payload.cargo.description` |
| 10 | `incoterms` | Умови поставки | enum | `FOB` | `payload.route.incoterms` |

### 1.3 Опціональні поля

| Field Code | Питання | Payload Path |
|---|---|---|
| `unloading_point` | Точка вивантаження | `payload.route.unloading_point` |
| `dangerous_class` | Клас небезпеки (якщо DG) | `payload.cargo.dangerous_class` |
| `client_contact_*` | Контакти клієнта | `payload.client.contact_*` |

---

## 2) Cargo Dimensions Schema — Канонічні назви полів

> ⚠️ **Контракт:** Усі документи використовують ці canonical field names.

```typescript
interface CargoPayload {
  // === CLIENT-DECLARED (UI writes) ===
  ready_date: string;                    // ISO date
  packages_count: number;                // К-сть місць (заявлена клієнтом)
  dimensions: DimensionItem[];           // Заявлені габарити
  total_cbm?: number;                    // Розрахований об'єм
  total_weight_kg?: number;              // Заявлена вага
  stackable: boolean;
  dangerous_goods: boolean;
  dangerous_class?: string;              // Якщо DG=true
  description: string;

  // === WAREHOUSE-VERIFIED (n8n writes) ===
  warehouse_packages_count?: number;     // Фактична к-сть
  warehouse_dimensions?: DimensionItem[];// Фактичні габарити
  warehouse_cbm?: number;                // Фактичний об'єм
  warehouse_weight_kg?: number;          // Фактична вага
  warehouse_stackable?: boolean;         // Підтвердження
  warehouse_verified_at?: string;        // Timestamp верифікації

  // === FINAL (після approval, n8n writes) ===
  final_dimensions?: DimensionItem[];    // Затверджені габарити
  final_cbm?: number;                    // Затверджений об'єм
  dims_source?: 'CLIENT' | 'WAREHOUSE' | 'MANUAL'; // Звідки взято
}

interface DimensionItem {
  length_cm: number;
  width_cm: number;
  height_cm: number;
  quantity: number;
}
```

### Правила:
1. `dimensions` — завжди те, що **клієнт заявив** (UI)
2. `warehouse_dimensions` — завжди те, що **склад виміряв** (n8n)
3. `final_dimensions` — **затверджений результат** після approval (n8n)
4. Якщо mismatch немає → `final_dimensions = warehouse_dimensions`
5. Якщо mismatch є → `DIMS_CHANGE_APPROVAL` → менеджер вибирає

---

## 3) Derived / Computed Fields

> **Примітка:** Ці поля зберігаються в `cases.computed` і записуються n8n/AI (не UI).

| Field | Опис | Computed Path | Written By |
|---|---|---|---|
| Quote breakdown | Деталізація витрат | `computed.quote.breakdown` | n8n |
| Quote margin | Маржа | `computed.quote.margin_usd` | n8n |
| Quote total | Ціна для клієнта | `computed.quote.total_usd` | n8n |
| Quote validity | Дійсна до | `computed.quote.valid_until` | n8n |
| Risks | Виявлені ризики | `computed.risks[]` | n8n / AI |
| NBA | Next Best Action | `computed.nba` | n8n |
| Tasks | Pending tasks | `computed.tasks[]` | n8n |
| SLA | Deadline tracking | `computed.sla` | n8n |
| AI extractions | Витягнуті дані | `computed.ai_extractions.*` | AI |

---

## 4) Integration Fields

Поля, що записуються інтеграційними нодами:

| Field | Опис | Payload Path | Written By |
|---|---|---|---|
| 1C Request ID | ID запиту | `payload.integration.onec_request_id` | n8n (1C Node) |
| 1C Deal Number | Номер угоди | `payload.integration.onec_deal_number` | n8n (1C Node) |
| Marking Code | `ALEX-<deal>` | `payload.integration.marking_code` | n8n |
| VP Table Row | ID рядка таблиці | `payload.integration.vp_table_row_id` | n8n (Spreadsheet) |

---

## 5) Key Documents (Docs)

> **Примітка:** Документи зберігаються в таблиці `documents` з посиланням на `case_id`.

| Document | doc_type | source | Коли потрібен | Навіщо |
|---|---|---|---|---|
| Контракт | `CONTRACT` | CLIENT/ZED | перед BL review | seller/buyer, умови |
| Інвойс | `INVOICE` | CLIENT/ZED | перед BL review | суми, опис |
| Пакувальний лист | `PACKING_LIST` | CLIENT | після warehouse dims | має збігатися з габаритами |
| Драфт BL | `BL_DRAFT` | ZED | після підготовки | ключовий документ |
| Довіреність | `POA` | CLIENT | ETA-3 weeks | експедирування |
| Інструкція | `CLEARANCE_INSTRUCTIONS` | SYSTEM | ETA-3 weeks | для логістів |
| Митні документи | `CUSTOMS_DOCS` | BROKER | перед оформленням | ПП/ПД, ЗДП |

---

## 6) Validation Rules & Thresholds

> ⚠️ **Контракт:** Конкретні числові пороги для автоматичних рішень.

### 6.1 Dims Mismatch

```yaml
dims_mismatch:
  threshold_cbm_percent: 10        # Якщо різниця > 10% → approval
  threshold_packages_count: 0      # Будь-яка різниця в к-сті → approval
  stackable_change_triggers: true  # Зміна stackable → завжди approval
```

**Логіка:**
- Обчислити `diff_percent = abs(warehouse_cbm - declared_cbm) / declared_cbm * 100`
- Якщо `diff_percent > 10` АБО `packages_count` змінився АБО `stackable` змінився:
  - Створити `DIMS_CHANGE_APPROVAL`
  - State → `DIMS_APPROVAL_PENDING`

### 6.2 Dangerous Goods

```yaml
dangerous_goods:
  requires_class: true             # Потрібен dangerous_class
  requires_msds: false             # MSDS опціональний для PoC
  blocks_auto_quote: true          # Quote потребує ручної перевірки
```

### 6.3 AI Confidence

```yaml
ai_confidence:
  auto_accept_threshold: 0.85      # Якщо >= 85% → auto-accept field
  needs_review_threshold: 0.70     # Якщо < 70% → add NEEDS_REVIEW risk
  low_threshold: 0.50              # Якщо < 50% → request clarification
```

---

## 7) Data Ownership (хто "власник істини")

| Дані | Власник | Хто пише в payload |
|---|---|---|
| Intake fields (1-10) | Клієнт | UI |
| Warehouse dims | Склад | n8n (webhook) |
| Final dims | Менеджер (approval) | n8n |
| 1C deal number | 1С | n8n (1C Node) |
| BL draft | ЗЕД | n8n (doc upload) |
| Export declaration | Брокер | UI (через менеджера) |
| Quote calculation | Система | n8n |
| Risks & NBA | Система/AI | n8n / AI |

---

## 8) JSONB Ownership Contract

| JSONB Field | UI writes | n8n writes | AI writes |
|---|---|---|---|
| `payload.client.*` | ✅ | ❌ | ❌ |
| `payload.cargo.dimensions` | ✅ | ❌ | ❌ |
| `payload.cargo.warehouse_*` | ❌ | ✅ | ❌ |
| `payload.cargo.final_*` | ❌ | ✅ | ❌ |
| `payload.route.*` | ✅ | ❌ | ❌ |
| `payload.broker.*` | ✅ | ❌ | ❌ |
| `payload.integration.*` | ❌ | ✅ | ❌ |
| `computed.quote.*` | ❌ | ✅ | ❌ |
| `computed.risks[]` | ❌ | ✅ | ✅ |
| `computed.nba` | ❌ | ✅ | ❌ |
| `computed.tasks[]` | ❌ | ✅ | ❌ |
| `computed.ai_extractions.*` | ❌ | ❌ | ✅ |

---

## 9) Glossary (F1 Sea Import специфічні терміни)

| Термін | Пояснення |
|---|---|
| **ЗЕД** | Відділ зовнішньоекономічної діяльності — готує BL, супроводжує документи |
| **BL (Bill of Lading)** | Коносамент — ключовий транспортний документ морського перевезення |
| **ПП/ПД** | Попередній перелік / Попередня декларація — митні документи |
| **ЗДП** | Заява декларанта платника — митний документ |
| **POA** | Power of Attorney / Довіреність на експедирування |
| **Telex Release** | Електронний реліз коносамента — дозволяє отримати вантаж без оригіналу BL |
| **VP Table** | Таблиця "Морські вантажі ВП" — внутрішній трекінг spreadsheet |
| **ALEX-<N>** | Формат маркування вантажу: ALEX- + номер угоди в 1С |
| **CBM** | Cubic Meter — кубічний метр (одиниця об'єму) |
| **DG** | Dangerous Goods — небезпечний вантаж |
| **MSDS** | Material Safety Data Sheet — паспорт безпеки для DG |
