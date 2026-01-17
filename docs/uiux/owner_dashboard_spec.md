# Owner Dashboard Specification
## IMCP — Business Owner / Operations Dashboard

**Версія:** 1.1  
**Статус:** Spec (PoC → MVP)  
**Тип документа:** Spec  
**Аудиторія:** Business Owner, Ops Lead, Product, UX/UI, Frontend, Architecture  
**Changelog:** v1.1 — додано Metric Definitions, виправлено state/status термінологію, empty states, sorting  

**Попередні документи:**  
- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](../core/01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](../core/02_core_data_model.md) — модель даних  
- [03_approval_pattern.md](../core/03_approval_pattern.md) — патерн підтверджень  
- [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) — UX/UI Case Cockpit  

**Пов'язані документи:**  
- [UI Style Reference](./ui_style_reference.md) — дизайн-токени  
- [System/Engineering Dashboard Spec](./system_engineering_dashboard_spec.md) — технічні інсайти (AI quality loop, інтеграції)  

---

## 1. Purpose — Навіщо потрібен Owner Dashboard

Owner Dashboard в IMCP — це **панель управління потоком кейсів, ризиками та якістю рішень** для бізнес-овнерів та операційних керівників.

### 1.0 Scope / Boundary (важливо)

Owner Dashboard відповідає на питання **"де буксує потік кейсів і що робити зараз"**.

- **Owner/Ops**: бачать *операційні* bottlenecks, SLA, approvals health, at-risk чергу і виконують *дії* (escalations/reassign/policy changes).
- **Engineering/System insights** (якість AI, quality loop, інтеграційні помилки, дебаг): **не є частиною Owner Dashboard v0/v1**, щоб не перевантажувати інтерфейс і не змішувати аудиторії.

> Пропозиція: винести технічні інсайти в окремий **System/Engineering Dashboard** (див. `docs/uiux/system_engineering_dashboard_spec.md`).

### 1.1 Основні задачі бізнес-овнера

| Задача | Опис |
|--------|------|
| **Здоров'я операцій** | Throughput, bottlenecks, SLA compliance |
| **Контроль ризиків** | At-risk кейси, ескалації, blocked зовнішніми залежностями |
| **Ефект від IMCP** | ROI, зниження ручної рутини, якість AI drafts |
| **Owner-level рішення** | Approvals високого рівня (escalations) |

### 1.2 Owner Dashboard ≠ Case Cockpit

| Аспект | Case Cockpit | Owner Dashboard |
|--------|--------------|-----------------|
| **Фокус** | Один кейс | Весь потік кейсів |
| **Аудиторія** | Менеджер/оператор | Business Owner, Ops Lead |
| **Ключове питання** | "Що робити далі в цьому кейсі?" | "Де буксує система і як вплинути?" |
| **Дії** | Approve/Edit/Reject drafts | Escalations, reassign, policy changes |

> **Owner Dashboard** — це "контрольна вежа", а не "кабіна пілота".

---

## 2. Guiding Principles (відповідність IMCP Mental Model)

Owner Dashboard дотримується принципів [Shared Mental Model](../core/00_shared_mental_model.md):

| Принцип | Реалізація в Owner Dashboard |
|---------|------------------------------|
| **Human-in-the-Loop** | Owner Approvals для escalations |
| **Accountability** | Хто затримує кейс, хто прийняв рішення |
| **Interfaces over Chatbots** | Single Pane of Glass, не чат |
| **Actionable Metrics** | KPI з drill-down до кейса |
| **Fatigue-Aware** | Показувати critical path, не все підряд |

### 2.1 Термінологія: State vs Status vs Block Reason

> ⚠️ **Контракт:** відповідно до [02_core_data_model.md](../core/02_core_data_model.md)

| Термін | Тип | Значення | Приклади |
|--------|-----|----------|----------|
| **`status`** | Технічний агрегат | Загальний стан кейса | `OPEN`, `BLOCKED`, `DONE`, `ARCHIVED` |
| **`state`** | Бізнес-стан | Позиція в state machine | `WAITING_CLIENT_INFO`, `QUOTE_READY`, `BL_REVIEW_PENDING` |
| **`substates`** | Паралельні підстани | Незалежні підпроцеси | `["INSURANCE_PENDING", "BROKER_DOCS_PENDING"]` |
| **`block_reason`** | Причина блокування | Чому кейс заблоковано | `WAITING_CLIENT`, `WAITING_BROKER`, `WAITING_ZED` |

**Для Owner Dashboard (контракт):**
- **Active cases** = `status IN ('OPEN', 'BLOCKED')` (обидва є "в роботі")
- **Open cases** = `status = 'OPEN'`
- **Blocked cases** = `status = 'BLOCKED'` (підмножина active; часто також at-risk)
- **At Risk** визначається комбінацією: SLA, pending approvals, `computed.risks`, `status = 'BLOCKED'`
- **Bottlenecks** аналізуються по `state` (бізнес-стани)

> **Примітка (substates):** У v0 Owner Dashboard не показує substates окремо. У v1+ substates враховуються при bottleneck-аналізі та можуть відображатись як "паралельні процеси" в At Risk Queue.

### 2.3 Що Owner Dashboard показує

- **Критичне** — те, що потребує уваги зараз
- **Операційне** — можливість швидко діяти
- **Людську відповідальність** — approvals, затримки, ескалації

### 2.4 Що Owner Dashboard НЕ показує

- Детальну роботу менеджера (це в Timeline кейса)
- Складні BI-звіти без можливості діяти
- Фінансові звіти бухгалтерії
- ML-прогнози (v2+)

---

## 3. Information Architecture

### 3.1 Навігація (PoC → MVP)

| Секція | PoC | MVP | Призначення |
|--------|-----|-----|-------------|
| **Overview** | ✅ | ✅ | Головна сторінка з KPI |
| **At Risk** | ✅ | ✅ | Черга кейсів під ризиком |
| **Approvals** | — | ✅ | Pending approvals health |
| **Escalations** | — | ✅ | Owner-level рішення |
| **Insights** | — | v2 | Бізнес/операційні інсайти (без дебагу) |
| **System / Engineering** | — | v2 | Окремий дашборд технічної якості/надійності (див. `system_engineering_dashboard_spec.md`) |

> **PoC:** може мати лише 1 сторінку Overview з усіма блоками.

### 3.2 Drill-down модель

```
Overview (KPI) → At Risk Queue → Case Cockpit
     ↓
Approvals Health → Pending List → Approval Detail → Case Cockpit
```

**Правило:** Owner ніколи не "застрягає" в аналітиці. Завжди можна перейти в кейс.

---

## 4. Data Sources (Supabase)

Owner Dashboard будується з core таблиць ([02_core_data_model.md](../core/02_core_data_model.md)):

| Таблиця | Дані для Dashboard |
|---------|-------------------|
| `cases` | Active count, status distribution, SLA, owner |
| `approvals` | Pending count, latency by type, overdue |
| `case_events` | Time-in-state metrics, bottlenecks |
| `ai_runs` | Acceptance rate, confidence flags |

### 4.1 Ключові queries

```sql
-- Active cases count (з org_id фільтром)
SELECT COUNT(*) 
FROM cases 
WHERE org_id = $org_id 
  AND status = 'OPEN';

-- At risk cases (SLA < 24h)
SELECT * FROM cases 
WHERE org_id = $org_id
  AND status IN ('OPEN', 'BLOCKED')
  AND sla_deadline < now() + interval '24 hours';

-- Pending approvals з percentiles
SELECT 
  approval_type, 
  COUNT(*) as count,
  AVG(EXTRACT(EPOCH FROM (now() - requested_at))/60)::int as avg_minutes,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - requested_at))/60)::int as p50_minutes,
  percentile_cont(0.9) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - requested_at))/60)::int as p90_minutes
FROM approvals 
WHERE status = 'PENDING'
  AND case_id IN (SELECT id FROM cases WHERE org_id = $org_id)
GROUP BY approval_type;

-- Top bottleneck states (only canonical states from case_type_configs)
SELECT state, AVG(EXTRACT(EPOCH FROM (next_event - created_at))/3600) as avg_hours
FROM (
  SELECT e.case_id, e.metadata->>'to_state' as state, e.created_at,
         LEAD(e.created_at) OVER (PARTITION BY e.case_id ORDER BY e.created_at) as next_event
  FROM case_events e
  JOIN cases c ON c.id = e.case_id
  WHERE e.event_type = 'STATE_CHANGED'
    AND c.org_id = $org_id
) t
WHERE state IS NOT NULL
GROUP BY state
ORDER BY avg_hours DESC
LIMIT 5;
```

---

## 5. Metric Definitions

> ⚠️ **Контракт:** формальні визначення метрик для імплементації.

| Metric | Definition | Source | Formula |
|--------|------------|--------|---------|
| **Active cases** | Кейси в роботі | `cases.status IN ('OPEN','BLOCKED')` | `COUNT(*)` |
| **At risk** | Кейси з ризиком SLA/блокування | View `v_cases_at_risk` | `COUNT(*)` |
| **Time-to-quote** | Час від створення до відправки ціни | `case_events` | `QUOTE_SENT.created_at - CASE_CREATED.created_at` |
| **Approval latency P50** | Медіана часу до рішення | `approvals` | `percentile_cont(0.5) OVER (decided_at - requested_at)` |
| **Approval latency P90** | 90-й перцентиль часу до рішення | `approvals` | `percentile_cont(0.9) OVER (decided_at - requested_at)` |
| **No-edit rate (AI)** | % AI approvals без правок | `case_events` (APPROVAL_APPROVED) | `COUNT(where ai_generated=true AND has_edits=false) / COUNT(where ai_generated=true)` |
| **No-edit rate (overall)** | % approvals без правок (усі) | `case_events` (APPROVAL_APPROVED) | `COUNT(where has_edits=false) / COUNT(*)` |
| **AI acceptance rate** | % AI drafts прийнятих (з edits або без) | `case_events` (APPROVAL_*) | `COUNT(APPROVED where ai_generated=true) / COUNT(CREATED where ai_generated=true)` |

> **Примітка:** `ai_generated` та `has_edits` беруться з `case_events.metadata` (подія `APPROVAL_APPROVED`). Див. [02_core_data_model.md](../core/02_core_data_model.md) Sec 4.2.

### 5.1 Event Types для метрик

| Metric | Event Type | Metadata Fields |
|--------|------------|-----------------|
| **Time-to-quote** | `QUOTE_SENT` | `{quote_id, sent_to}` |
| **Quote created** | `STATE_CHANGED` | `{to_state: 'QUOTE_READY'}` |
| **Approval decision** | `APPROVAL_APPROVED` / `APPROVAL_REJECTED` | `{approval_id, has_edits: boolean}` |

### 5.2 Визначення "без edits"

Approval вважається **без edits**, якщо:

```sql
-- Варіант A: порівняння snapshots (точний)
decision_snapshot IS NOT DISTINCT FROM request_snapshot

-- Варіант B: флаг в event metadata (рекомендований)
-- event_type = 'APPROVAL_APPROVED' AND metadata->>'has_edits' = 'false'
```

> **Рекомендація:** використовувати `has_edits` флаг в `case_events.metadata` при `APPROVAL_APPROVED`.

> **Owner Dashboard KPI:** використовуємо **No-edit rate (AI)** як основну метрику якості AI-чернеток.

---

## 6. Dashboard Blocks Specification

### 6.1 Operational Overview (KPI Tiles)

Верхній рядок з ключовими метриками.

| KPI | Опис | Поріг (At Risk) | Див. Metric Definition |
|-----|------|-----------------|------------------------|
| **Active cases** | Кейси зі `status IN ('OPEN', 'BLOCKED')` | — | Sec. 5 |
| **At risk** | Кейси під ризиком SLA/зриву | > 10% від active | Sec. 5 |
| **Pending approvals** | Approvals зі `status = 'PENDING'` | > threshold для типу | Sec. 5 |
| **Time-to-quote (avg)** | Середній час до відправки ціни | > 4h | Sec. 5 |
| **Approval latency P90** | 90-й перцентиль часу до рішення | > 2h | Sec. 5 |
| **No-edit rate (AI)** | % AI approvals прийнятих без правок | < 50% | Sec. 5 |
| **AI acceptance rate** | % AI drafts/чернеток прийнятих (з/без правок) | < policy threshold | Sec. 5 |

> **Примітка:** Для Owner Dashboard ключова метрика якості — **No-edit rate (AI)**. Якщо потрібна загальна якість процесу — додаємо окремо **No-edit rate (overall)** (не обовʼязково для v0).

**UX:**
- Кожна плитка клікабельна → drill-down
- Колір індикатора: 🟢 OK / 🟡 Warning / 🔴 Critical
- Період вибирається в header (Last 7 days / 30 days / Custom)
- "Last updated: Xs ago" indicator (див. Sec. 8.6)

---

### 6.2 At Risk Queue

Операційна черга кейсів, які потребують уваги.

#### Критерії At Risk

| Критерій | Опис | Джерело | Як визначається |
|----------|------|---------|-----------------|
| **SLA < 24h** | Дедлайн наближається | `cases.sla_deadline` | `sla_deadline < now() + interval '24h'` |
| **Pending approval > threshold** | Approval зависає | `approvals` | `requested_at < now() - threshold` |
| **Missing critical data** | Немає обов'язкових полів | `cases.payload` vs `case_type_configs.required_fields[state]` | Порівняння payload з required_fields для поточного state |
| **Conflict detected** | Розбіжність даних | `cases.computed.risks[]` | `risks[].code = 'DIMS_MISMATCH'` або інші |
| **Blocked external** | Чекаємо broker/ZED/client | `cases.status = 'BLOCKED'` | `status = 'BLOCKED'` (окремий статус, не state) |

> **Примітка:** `BLOCKED` — це `status`, а не `state`. Кейс може бути `status = 'BLOCKED'` у будь-якому `state` (наприклад, `state = 'WAITING_CLIENT_INFO'` + `status = 'BLOCKED'` означає, що чекаємо клієнта і процес заблоковано).

#### Колонки таблиці

| Колонка | Тип | Опис |
|---------|-----|------|
| Case ID | Link | Клікабельний ID кейса |
| Case type | Badge | Тип кейса (F1_SEA_IMPORT) |
| Risk reason | Text | 1–2 слова чому at risk |
| State | Badge | Поточний бізнес-стан |
| Owner | Avatar + name | Відповідальний менеджер |
| SLA / deadline | Relative time | "2h 30m left" / "Overdue 4h" |
| Action | Button | "View case" |

**UX-правило:** Owner має за 5 секунд зрозуміти "чому цей кейс тут".

---

### 6.3 Approvals Health

Блок показує, де система "стоїть на воротах" ([03_approval_pattern.md](../core/03_approval_pattern.md)).

| Показник | Опис |
|----------|------|
| **Pending approvals total** | Загальна кількість PENDING |
| **By type** | Розбивка: QUOTE / BL / INSURANCE / 1C |
| **Avg time by type** | Середній час до рішення |
| **Overdue approvals** | Понад threshold для типу |

#### Thresholds за типом approval

| approval_type | Warning | Critical |
|---------------|---------|----------|
| `QUOTE_APPROVAL` | > 30m | > 2h |
| `BL_DRAFT_APPROVAL` | > 1h | > 4h |
| `DIMS_CHANGE_APPROVAL` | > 1h | > 4h |
| `ONEC_DEAL_CREATE_APPROVAL` | > 15m | > 1h |

> **Важливо:** у всіх місцях UI/alerting використовуємо **per-approval_type thresholds**, а не “один поріг на все”.

---

### 6.4 Top Bottlenecks

Відповідає на питання: **"На якому етапі ми найчастіше зависаємо?"**

| Показник | Візуалізація |
|----------|--------------|
| Top 3 states з найбільшим avg time-in-state | Horizontal bar chart |
| Top 3 причини блокувань | List з counts |

**Приклад причин блокувань:**
- `MISSING_DATA` — немає даних від клієнта
- `WAITING_CLIENT` — чекаємо відповідь клієнта
- `WAITING_BROKER` — чекаємо брокера
- `DOC_REVIEW` — документ на перевірці

---

### 6.5 Escalations (Owner Decisions) — MVP

Escalation — це approval, який потребує рішення business owner, а не менеджера.

#### Типові причини ескалації

| Причина | Опис |
|---------|------|
| **High value shipment** | Вартість понад threshold |
| **Non-standard margin** | Маржа поза межами policy |
| **Risky contractor** | Підрядник з low rating |
| **Dangerous cargo** | Небезпечний вантаж |
| **Force majeure** | Перенаправлення, зміна маршруту |
| **Policy exception** | Виключення з бізнес-правил |

#### UX

- Окрема секція "Owner Decisions"
- Кнопки: **Approve** / **Reject** + Comment
- Посилання на кейс для контексту
- Показувати `request_snapshot` (що система пропонує)

---

## 7. Wireframe — Owner Dashboard (PoC v0)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Owner Dashboard                              Period: [Last 7 days ▾]  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ 📊 OPERATIONAL OVERVIEW                                                  │ │
│ ├──────────┬──────────┬──────────┬──────────┬──────────┬──────────────────┤ │
│ │ Active   │ At Risk  │ Pending  │ Time-to  │ Approval │ AI Drafts        │ │
│ │ Cases    │          │ Approvals│ Quote    │ Latency  │ Accepted         │ │
│ │          │          │          │          │          │                  │ │
│ │   48     │    6     │    9     │   2h     │ P50: 12m │    63%           │ │
│ │          │   🔴     │   🟡     │   🟢     │ P90: 1h40│   🟡             │ │
│ └──────────┴──────────┴──────────┴──────────┴──────────┴──────────────────┘ │
│                                                                              │
│ ┌─────────────────────────────────────┬────────────────────────────────────┐ │
│ │ 🔥 AT RISK QUEUE                    │ ⏳ APPROVALS HEALTH                │ │
│ ├─────────────────────────────────────┼────────────────────────────────────┤ │
│ │                                     │                                    │ │
│ │ Case ID          Risk       State   │ Pending total: 9                   │ │
│ │ ─────────────────────────────────── │ ──────────────────────────────     │ │
│ │ #F1-SEA-02451    SLA<24h    WAITING │ QUOTE:   4  (avg 18m)    🟢        │ │
│ │ #F1-SEA-02477    CONFLICT   DOCS_RE │ BL:      3  (avg 55m)    🟡        │ │
│ │ #F1-SEA-02480    BLOCKED    WAITING │ DIMS:    1  (avg 2h)     🔴        │ │
│ │ #F1-SEA-02485    MISSING    CLIENT_ │ 1C:      1  (avg 10m)    🟢        │ │
│ │                                     │                                    │ │
│ │ [View all 6 at risk →]              │ [View pending approvals →]         │ │
│ └─────────────────────────────────────┴────────────────────────────────────┘ │
│                                                                              │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ 📌 TOP BOTTLENECKS (Avg time in state)                                   │ │
│ ├──────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                          │ │
│ │ DOCS_REVIEW           ████████████████████████████  1d 4h                │ │
│ │ WAITING_CLIENT_INFO   ████████████████              10h                  │ │
│ │ WAITING_APPROVAL      ████████                      2h                   │ │
│ │                                                                          │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ 🧨 ESCALATIONS (Owner Decisions)                        [ 2 pending ]    │ │
│ ├──────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                          │ │
│ │ #F1-SEA-02488  High value shipment ($45,000)                             │ │
│ │ Approve non-standard insurance?                                          │ │
│ │                                      [View Case]  [Approve]  [Reject]    │ │
│ │ ────────────────────────────────────────────────────────────────────     │ │
│ │ #F1-AIR-01203  Margin below threshold (8%)                               │ │
│ │ Approve reduced margin quote?                                            │ │
│ │                                      [View Case]  [Approve]  [Reject]    │ │
│ │                                                                          │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. UX Behavior & Interactions

### 8.1 Drill-down Flow

| З блоку | Дія | Результат |
|---------|-----|-----------|
| KPI tile "At Risk" | Click | → At Risk Queue (full list) |
| KPI tile "Pending Approvals" | Click | → Approvals list |
| At Risk row | Click | → Case Cockpit |
| Escalation "View Case" | Click | → Case Cockpit |
| Escalation "Approve/Reject" | Click | → Inline decision + Comment modal |

### 8.2 Пороги та алерти

Пороги задаються як policy (можуть бути в `case_type_configs` або окремій таблиці):

| Поріг | Default | Customizable |
|-------|---------|--------------|
| SLA warning | < 24h | ✅ |
| Pending approval warning | ✅ per type | ✅ per type |
| Conflict detected | auto | — |
| At risk % threshold | > 10% | ✅ |

### 8.3 Owner Actions

| Дія | Де доступна | Результат | Audit Event |
|-----|-------------|-----------|-------------|
| **View case** | Усюди | → Case Cockpit | — |
| **Approve escalation** | Escalations block | → Decision recorded + workflow triggered | `APPROVAL_APPROVED` |
| **Reject escalation** | Escalations block | → Decision recorded + fallback workflow | `APPROVAL_REJECTED` |
| **Reassign owner** | At Risk Queue (MVP) | → Owner changed + notification | `OWNER_REASSIGNED` |

#### Правила для Owner Actions

| Правило | PoC | MVP |
|---------|-----|-----|
| **Коментар при Reject** | Опціональний | **Обов'язковий** |
| **Коментар при Reassign** | — | Опціональний |
| **Bulk actions** | ❌ Не підтримується | ❌ Не підтримується (v2) |
| **Audit trail** | Записується в `case_events` | Записується в `case_events` |

---

### 8.4 Empty & Error States

| Стан | Що показуємо | UX |
|------|--------------|-----|
| **No at-risk cases** | "All cases healthy" + green indicator | Позитивне підтвердження |
| **No pending approvals** | "No approvals pending" | Нейтральний стан |
| **No data for period** | "No data available for selected period" | Підказка змінити період |
| **Loading** | Skeleton loaders для кожного блоку | Не блокувати весь dashboard |
| **Error fetching data** | Error message + Retry button per block | Ізольовані помилки |
| **Partial data** | Показувати доступні дані + warning badge | Graceful degradation |

### 8.5 Sorting & Filters (At Risk Queue)

| Параметр | PoC | MVP |
|----------|-----|-----|
| **Default sort** | `sla_deadline ASC` (найближчий дедлайн першим) | Те саме |
| **Overdue first** | ✅ (overdue cases on top) | ✅ |
| **Filter: case_type** | — | ✅ |
| **Filter: owner** | — | ✅ |
| **Filter: risk_category** | — | ✅ |

### 8.6 Data Freshness

| Компонент | Refresh strategy | Last Updated indicator |
|-----------|------------------|------------------------|
| **KPI tiles** | Polling 30s / Real-time (MVP) | ✅ показувати "Updated X sec ago" |
| **At Risk Queue** | Polling 30s / Real-time (MVP) | ✅ |
| **Approvals Health** | Polling 30s / Real-time (MVP) | ✅ |

> **Fallback:** якщо real-time subscription fails → автоматично перемикатись на polling.

---

## 9. UI Components (відповідність Style Reference)

Компоненти відповідають [UI Style Reference](./ui_style_reference.md):

### 9.1 KPI Tile

```css
.kpi-tile {
  background: var(--bg-sidebar);      /* #F3F3F3 */
  border: 1px solid var(--border);    /* #D0D0D0 */
  border-radius: var(--radius-md);    /* 10px */
  padding: 16px;
}

.kpi-value {
  font-size: 28px;
  font-weight: 600;
  color: var(--text);                 /* #1E1E1E */
}

.kpi-label {
  font-size: 12px;
  color: var(--text-secondary);       /* #5A5A5A */
}

.kpi-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.kpi-indicator--ok { background: #22C55E; }
.kpi-indicator--warning { background: #F59E0B; }
.kpi-indicator--critical { background: #EF4444; }
```

### 9.2 Risk Queue Row

```css
.risk-row {
  padding: 12px 16px;
  border-bottom: 1px solid var(--divider);  /* #E0E0E0 */
}

.risk-row:hover {
  background: var(--bg-hover);              /* rgba(0,0,0,0.04) */
  cursor: pointer;
}

.risk-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: var(--radius-sm);          /* 6px */
  background: #FEE2E2;
  color: #DC2626;
}
```

### 9.3 Escalation Card

```css
.escalation-card {
  background: var(--bg-editor);             /* #FFFFFF */
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 16px;
  margin-bottom: 12px;
  box-shadow: var(--shadow-sm);             /* 0 1px 2px rgba(0,0,0,0.08) */
}

.escalation-actions {
  display: flex;
  gap: 8px;
  margin-top: 12px;
}
```

---

## 10. ROI Metrics (Вимірювання ефекту IMCP)

Owner Dashboard має показувати не тільки "стан", а й "ефект" платформи.

### 10.1 PoC Metrics

| Метрика | Опис | Target | Definition (Sec. 5) |
|---------|------|--------|---------------------|
| **Time-to-quote (avg)** | Час від створення до відправки ціни | ↓ 30% | `QUOTE_SENT - CASE_CREATED` |
| **Approval latency P90** | 90-й перцентиль часу до рішення | < 2h | `percentile_cont(0.9)` |
| **No-edit rate** | % approvals прийнятих без правок | > 60% | `has_edits = false` |
| **Conflicts detected** | Кількість виявлених системою розбіжностей | ↑ | `computed.risks[].code` |
| **At risk resolution** | Час від at-risk до resolved | ↓ 20% | Event timestamps |

### 10.2 MVP Metrics (додатково)

| Метрика | Опис |
|---------|------|
| **Cases per manager** | Throughput на менеджера |
| **SLA compliance** | % кейсів у межах SLA |
| **Escalation rate** | % кейсів з ескалаціями |
| **Manual interventions** | Кількість ручних втручань |

---

## 11. Anti-patterns (чого НЕ робити)

| ❌ Anti-pattern | Проблема | ✅ Рішення |
|-----------------|----------|------------|
| Дашборд як "BI звіт" | Немає можливості діяти | Drill-down до кейса |
| 30 графіків | Когнітивне перевантаження | 5-6 ключових метрик |
| Метрики без порогів | Незрозуміло чи добре/погано | Кольорові індикатори |
| Показувати все | Втрачається фокус | Показувати critical path |
| Немає drill-down | Owner не може діяти | Клікабельні елементи |
| Refresh вручну | Застарілі дані | Real-time updates |

---

## 12. Acceptance Criteria (Definition of Done)

### 12.1 PoC Acceptance

Owner Dashboard v0 вважається успішним, якщо:

- [ ] Owner за **30 секунд** розуміє:
  - скільки кейсів активних
  - де ризики
  - де approvals зависли
- [ ] Owner може за **2 кліки** перейти в проблемний кейс
- [ ] Метрики оновлюються **автоматично** (real-time або polling)
- [ ] Немає ручного "заповнення дашборду"
- [ ] Responsive layout (desktop-first)

### 12.2 MVP Acceptance (додатково)

- [ ] Escalations з можливістю Approve/Reject inline
- [ ] Фільтри по case_type / team / time period
- [ ] Export даних (CSV)
- [ ] Mobile-friendly view

---

## 13. Roadmap: v0 → v1 → v2

### v0 (PoC)

| Компонент | Включено |
|-----------|----------|
| KPI tiles (6) | ✅ |
| At risk queue | ✅ |
| Approvals health | ✅ |
| Bottlenecks (top 3) | ✅ |
| Escalations | — |
| Filters | — |

### v1 (MVP)

| Компонент | Включено |
|-----------|----------|
| v0 features | ✅ |
| Escalations | ✅ |
| Reassign owners | ✅ |
| Filters (case_type, team, period) | ✅ |
| SLA policies configuration | ✅ |
| Export (CSV) | ✅ |

### v2 (Scale)

| Компонент | Включено |
|-----------|----------|
| v1 features | ✅ |
| Predictive risk scoring | ✅ |
| Cost/margin analytics | ✅ |
| Manager performance | ✅ |
| AI performance monitoring | ✅ |
| Custom dashboards | ✅ |

---

## 14. Technical Implementation Notes

### 14.1 Data Fetching Strategy

| Підхід | PoC | MVP |
|--------|-----|-----|
| **KPI tiles** | Polling (30s) | Real-time subscriptions |
| **At Risk Queue** | Polling (30s) | Real-time subscriptions |
| **Approvals** | Polling (30s) | Real-time subscriptions |

### 14.2 Supabase Views (рекомендовано)

```sql
-- View for At Risk cases
-- Примітка: status IN ('OPEN', 'BLOCKED') — обидва є "активними" для dashboard
CREATE VIEW v_cases_at_risk AS
SELECT 
  c.id,
  c.case_number,
  c.case_type,
  c.state,
  c.status,
  c.owner_user_id,
  c.org_id,
  c.sla_deadline,
  c.computed->'risks' as risks,
  -- Risk category (пріоритет: SLA > BLOCKED > HAS_RISKS > APPROVAL_OVERDUE)
  CASE 
    WHEN c.sla_deadline < now() THEN 'SLA_OVERDUE'
    WHEN c.sla_deadline < now() + interval '24 hours' THEN 'SLA_CRITICAL'
    WHEN c.status = 'BLOCKED' THEN 'BLOCKED_EXTERNAL'
    WHEN jsonb_array_length(COALESCE(c.computed->'risks', '[]'::jsonb)) > 0 THEN 'HAS_RISKS'
    WHEN EXISTS (
      SELECT 1 FROM approvals a 
      WHERE a.case_id = c.id 
        AND a.status = 'PENDING' 
        AND a.requested_at < now() - interval '2 hours'
    ) THEN 'APPROVAL_OVERDUE'
    ELSE 'OTHER'
  END as risk_category,
  -- Sort priority (lower = more urgent)
  CASE 
    WHEN c.sla_deadline < now() THEN 1
    WHEN c.sla_deadline < now() + interval '24 hours' THEN 2
    WHEN c.status = 'BLOCKED' THEN 3
    ELSE 4
  END as sort_priority
FROM cases c
WHERE c.status IN ('OPEN', 'BLOCKED')  -- Include both OPEN and BLOCKED
  AND (
    c.sla_deadline < now() + interval '24 hours'
    OR c.status = 'BLOCKED'
    OR jsonb_array_length(COALESCE(c.computed->'risks', '[]'::jsonb)) > 0
    OR EXISTS (
      SELECT 1 FROM approvals a 
      WHERE a.case_id = c.id 
        AND a.status = 'PENDING' 
        AND a.requested_at < now() - interval '2 hours'
    )
  )
ORDER BY sort_priority, c.sla_deadline NULLS LAST;

-- View for Approvals Health (з org_id для RLS)
CREATE VIEW v_approvals_health AS
SELECT 
  a.approval_type,
  c.org_id,
  COUNT(*) as pending_count,
  AVG(EXTRACT(EPOCH FROM (now() - a.requested_at))/60)::int as avg_minutes,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - a.requested_at))/60)::int as p50_minutes,
  percentile_cont(0.9) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - a.requested_at))/60)::int as p90_minutes,
  COUNT(*) FILTER (WHERE a.requested_at < now() - interval '2 hours') as overdue_count
FROM approvals a
JOIN cases c ON c.id = a.case_id
WHERE a.status = 'PENDING'
GROUP BY a.approval_type, c.org_id;
```

### 14.3 RLS Considerations

Owner Dashboard потребує доступу до агрегованих даних організації:

```sql
-- Policy for owner dashboard access
CREATE POLICY "owner_dashboard_read" ON cases
  FOR SELECT
  USING (
    org_id = auth.jwt() ->> 'org_id'
    AND (
      auth.jwt() ->> 'role' IN ('ADMIN', 'OWNER', 'OPS_LEAD', 'MANAGER')
    )
  );
```

> **Примітка:** Ролі `MANAGER`, `OPS_LEAD`, `ADMIN` мають доступ до Owner Dashboard. Роль `ENGINEER` має доступ до Engineering Dashboard (див. [02_core_data_model.md](../core/02_core_data_model.md) Sec 7.2).

### 14.4 Idempotency для Escalation Approvals

При виконанні Owner-level рішень (Approve/Reject escalation) використовується `idempotency_key`:

```sql
-- Escalation approval decision
UPDATE approvals 
SET 
  status = 'APPROVED',
  decided_by = $owner_user_id,
  decided_at = now(),
  decision_snapshot = $snapshot,
  decision_comment = $comment
WHERE 
  id = $approval_id 
  AND status = 'PENDING'
  AND idempotency_key = $expected_key  -- додаткова перевірка
RETURNING *;
```

> **Контракт:** `idempotency_key` забезпечує захист від дублювання рішень (див. [02_core_data_model.md](../core/02_core_data_model.md) Sec 6.4).

---

## 15. Integration with Case Cockpit

Owner Dashboard інтегрується з [Case Cockpit](../core/04_case_cockpit_ux.md):

| Взаємодія | Напрямок | Опис |
|-----------|----------|------|
| Drill-down | Dashboard → Cockpit | Click на кейс відкриває Cockpit |
| Context | Cockpit ← Dashboard | Cockpit може показувати "ви тут з Dashboard" |
| Escalations | Dashboard → Cockpit | View Case перед рішенням |
| Notifications | Cockpit → Dashboard | NBA "Escalate to Owner" |

---

## 16. End Note

Owner Dashboard в IMCP — це **"костюм" для бізнес-овнера**:
- не замінює управління
- робить його швидким, прозорим і контрольованим

> **Ключова функція:** показувати ризики, вузькі місця та якість рішень, щоб бізнес міг діяти.

---

**Пов'язана документація:**
- [/docs/core/](../core/) — core документація IMCP
- [/docs/case_templates/](../case_templates/) — шаблони кейсів
- [ui_style_reference.md](./ui_style_reference.md) — дизайн-токени
