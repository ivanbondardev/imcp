# System / Engineering Dashboard Specification
## IMCP — Platform Health + AI Quality (Engineering Control Tower)

**Версія:** 0.1  
**Статус:** Draft (v0 → v1)  
**Тип документа:** Spec  
**Аудиторія:** Engineering, Ops Lead, Architecture, Product, UX/UI, Frontend  

**Попередні документи:**  
- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](../core/01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](../core/02_core_data_model.md) — контракти даних (Event Taxonomy, RLS)  
- [03_approval_pattern.md](../core/03_approval_pattern.md) — approvals + snapshots  
- [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) — UX/UI Case Cockpit  

**Пов'язані документи:**  
- [Owner Dashboard Spec](./owner_dashboard_spec.md) — операційний контроль потоку (Owner/Ops)  
- [UI Style Reference](./ui_style_reference.md) — дизайн-токени  

---

## 1. Purpose — Навіщо потрібен System/Engineering Dashboard

Це окремий дашборд “контрольної вежі” для інженерів та Ops Lead, який відповідає на питання:

- **Чому** система буксує (помилки інтеграцій, деградація AI, черги/латентності)
- **Що саме** погано (які поля найчастіше правлять, де падає впевненість, які помилки повторюються)
- **Де дебажити** (drill-down до конкретних `ai_run`, `case`, `integration_failed` подій, payloads)

> Принцип: **інженерний дебаг не змішуємо** з Owner Dashboard, щоб уникнути когнітивного шуму та змішування ролей.

---

## 2. Personas / Access (RLS)

> **Контракт:** Ролі визначено в [02_core_data_model.md](../core/02_core_data_model.md) Sec 7.2.

| Роль | Engineering Dashboard Access | Що бачить |
|------|------------------------------|-----------|
| **ENGINEER** | ✅ Full | Усі технічні метрики, AI runs, integration logs, payloads |
| **ADMIN** | ✅ Full | Те саме + можливість змінювати конфігурації |
| **OPS_LEAD** | ⚠️ Partial | Агрегати (KPI), списки кейсів, але без raw payloads/logs |
| **MANAGER** | ❌ No access | Тільки Owner Dashboard |
| **OWNER** | ❌ No access | Тільки Owner Dashboard (+ high-level "System degraded" banner, якщо потрібно) |

> **Примітка:** `ENGINEER` — спеціалізована роль для технічного моніторингу. У core моделі еквівалентна `ADMIN` з обмеженими правами на бізнес-рішення.

---

## 3. Information Architecture

### 3.1 Навігація

| Секція | v0 | v1 | Призначення |
|--------|----|----|-------------|
| **Overview** | ✅ | ✅ | Стан платформи та ключові алерти |
| **AI Quality** | ✅ | ✅ | Confidence, acceptance, no-edit, quality loop |
| **Quality Loop** | ✅ | ✅ | Найчастіші ручні правки по полях/типах |
| **Integrations** | ✅ | ✅ | `INTEGRATION_FAILED` та деградації інтеграцій |
| **Data Freshness** | ✅ | ✅ | Staleness/pipeline lag, real-time health |
| **Incidents** | — | ✅ | Підбірка інцидентів/ретраїв/помилок по часу |

### 3.2 Drill-down модель

```
System Overview → Problem Area (AI/Integration) → Top offenders → конкретний кейс/ран/подія → артефакти (payload/log)
```

---

## 4. Dashboard Blocks (v0)

### 4.1 System Overview (KPI Tiles)

Рекомендовані плитки (6–8, адаптивна сітка):

- **Integration failures (24h)**: `COUNT(case_events where event_type='INTEGRATION_FAILED')`
- **Failure rate by integration**: Top-3 системи (1C, Broker API, ZED) + %
- **AI confidence (avg / p10)**: по `ai_runs.confidence` або extracted fields confidence
- **LOW_CONFIDENCE flagged**: `COUNT(risks[].code='LOW_CONFIDENCE')`
- **AI acceptance rate**: загальна + по типу чернетки
- **No-edit rate (AI)**: як якість “draft-first”

**UX:**
- Плитки клікабельні → ведуть у відповідний список/графік
- Показувати **trend** (Δ vs previous period) як маленький subtext
- Показувати **Last updated** (і staleness)

### 4.2 AI Quality (Confidence + Flags)

Блок відповідає на "чи деградував AI".

**Data Sources:**
- `ai_runs.confidence` — числова впевненість (0–1)
- `ai_runs.flags` — масив флагів (`['LOW_CONFIDENCE', 'NEEDS_REVIEW']`)
- `case_events` з `event_type = 'AI_RUN_COMPLETED'` (metadata: `{ai_run_id, run_type, confidence}`)
- `cases.computed.risks[]` з `code = 'LOW_CONFIDENCE'` та `severity`

**Візуалізація:**
- Sparkline/line chart confidence по часу (7/30 днів)
- Breakdown по `case_type` і по "draft type" (QUOTE/BL/DIMS/…)
- Top risk flags: `LOW_CONFIDENCE`, `DOC_PARSE_FAILED`, `CONFLICT_DETECTED`
- Показувати **severity** (`LOW/MEDIUM/HIGH`) для кожного флагу, не лише code

**AI Runs Log (v1):**
- Таблиця останніх N `AI_RUN_COMPLETED` подій
- Колонки: timestamp, case_id, run_type, confidence, flags, duration_ms
- Drill-down до case / ai_run деталей

### 4.3 Quality Loop (Most edited fields)

Мета: показати розрив між `request_snapshot` і `decision_snapshot` як backlog для AI/правил.

**Data Sources (узгоджено з core):**
- `case_events` з `event_type = 'APPROVAL_APPROVED'`
- `metadata.has_edits` (boolean) — чи були правки
- `metadata.edited_fields[]` (optional, string[]) — список змінених полів
- `metadata.ai_generated` (from `APPROVAL_CREATED` event) — чи AI-чернетка

> **Примітка:** `edited_fields[]` — optional поле, додане в [02_core_data_model.md](../core/02_core_data_model.md) Sec 4.2. Якщо не заповнено — можна порівнювати `request_snapshot` vs `decision_snapshot` напряму.

**Візуалізація:**
- Таблиця: **Field path** / **Edit frequency** / **Avg delta** / **Top case_types** / **Example link**
- Фільтри: period, case_type, approval_type
- Drill-down: клік на field → список кейсів де це поле правили

### 4.4 Integrations (Error taxonomy)

**Data Sources (узгоджено з core):**
- `case_events` з `event_type = 'INTEGRATION_FAILED'`
- `metadata.integration_id` — UUID інтеграції
- `metadata.error` — текст помилки
- `metadata.error_code` — нормалізований код (TIMEOUT, AUTH, VALIDATION, RATE_LIMIT, UNKNOWN)
- `metadata.retry_count` — кількість спроб
- `metadata.retryable` — чи можна повторити

> **Примітка:** `error_code` та `retryable` додано в [02_core_data_model.md](../core/02_core_data_model.md) Sec 4.4 для стандартизації error taxonomy.

**Normalized Error Codes:**
| error_code | Опис | Типовий action |
|------------|------|----------------|
| `TIMEOUT` | Час очікування вичерпано | Auto-retry |
| `AUTH` | Помилка авторизації | Manual fix (tokens/creds) |
| `VALIDATION` | Невалідні дані | Fix payload |
| `RATE_LIMIT` | Ліміт запитів | Wait & retry |
| `UNKNOWN` | Невідома помилка | Manual investigation |

**Візуалізація:**
- Top-10 error_code за період (pie/bar chart)
- Heatmap "failures by hour" (для виявлення patterns)
- Breakdown by `integration_type` (ONEC, ZED, BROKER, STORAGE)
- Список останніх N падінь з посиланням на кейс + payload ref (за доступом)

### 4.5 Data Freshness / Pipeline Lag

Мета: швидко зрозуміти “дані протухли” vs “реально все ок”.

Метрики:
- `subscription_health` (ok/degraded/off)
- `polling_fallback_rate`
- `max_lag_seconds` по ключових агрегатах

### 4.6 Human Expertise Signals (Patterns, not people)

Мета: зробити **експертний людський досвід видимим** через повторювані патерни, але **без персоналізації** (щоб не перетворити дашборд на “контроль людей”).

Цей блок відповідає на питання:
- Де саме люди “рятують” систему (які поля/кроки найчастіше виправляються)
- У яких контекстах це відбувається (case_type / approval_type / state)
- Чи є “гарячі зони” для покращень (топ комбінації field × case_type × approval_type)

**Data Sources (v0):**
- `case_events` з `event_type='APPROVAL_APPROVED'`
  - `metadata.has_edits` (bool)
  - `metadata.edited_fields[]` (optional)
  - `metadata.approval_type`
  - `metadata.ai_generated` (bool, з `APPROVAL_CREATED`)
- `approvals` (для агрегатів по approval_type / latency) — за доступом
- `cases.case_type`, `cases.state`, `cases.status`

**Візуалізація (v0):**
- KPI mini-cards: “Human overrides (30d)”, “Top edited field”, “Edits concentrated in (case_type)”
- Таблиця: **Field** / **Edit rate** / **Top contexts** / **Drill-down**

**RLS / Privacy:**
- ENGINEER/ADMIN: агрегати + drill-down до кейсів/подій
- OPS_LEAD: лише агрегати (без raw payload/logs)
- Не показувати персональні імена/рейтинги. Дозволені лише агрегати “по команді/ролі”, якщо це справді потрібно.

### 4.7 Decision Reasons (Why humans changed / rejected)

Мета: зрозуміти **чому** люди відхиляють/правлять (це найбільш цінний сигнал для покращення AI та правил).

**Контракт (рекомендовано для v1):**
- `case_events` з `event_type in ('APPROVAL_APPROVED','APPROVAL_REJECTED')`
  - `metadata.reason_code` (string) — нормалізована причина (taxonomy)
  - `metadata.reason_note` (string, optional) — короткий коментар (sanitize/limits)
  - `metadata.edited_fields[]` (optional)

**Reason taxonomy (proposal):**
- `MISSING_CONTEXT` — бракує контексту/вхідних даних
- `POLICY_EXCEPTION` — нестандартний кейс/виняток з політики
- `DATA_CONFLICT` — конфлікт між джерелами
- `CLIENT_REQUEST` — рішення через побажання клієнта/партнера
- `RISK_TOO_HIGH` — ризик/комплаєнс/небезпечний вантаж
- `PRICE_ADJUSTMENT` — корекція ціни/маржі
- `DOC_ISSUE` — проблема документа (формат/помилки)
- `INTEGRATION_ERROR` — наслідок збою інтеграції
- `OTHER` — інше

**Візуалізація:**
- Donut/бар-чарт “Reasons distribution (30d)”
- Top reasons by approval_type / case_type
- Drill-down: reason_code → список кейсів/подій (з обмеженнями RLS)

### 4.8 Improvement Impact (Release / Prompt / Rule changes)

Мета: перетворити дашборд на **механізм еволюції**: бачити, які покращення дали ефект, і де регресії.

**Data Sources (v0):**
- Lightweight “changelog” (може бути таблиця в DB або навіть конфіг/лог подій):
  - `change_id`, `change_type` (PROMPT/RULE/INTEGRATION), `owner`, `created_at`
  - `notes` (коротко: що змінено)
- Метрики до/після по ключових KPI:
  - No-edit rate (AI), AI acceptance, risk flags rate, integration failures, approval latency

**Візуалізація (v0):**
- Список останніх змін з бейджем “+/- impact” (7–14 днів)
- Mini-metrics: “No-edit ↑”, “LOW_CONFIDENCE ↓”, “Failures ↓”
- “Regressions” block: що погіршилось після зміни

---

## 5. UI/UX Look & Feel (узгоджено з Style Reference)

- **Cards/панелі**: світлі, мінімум рамок, hover підсвітка, тонкий accent на selected
- **Сигнали**: 3-рівневі (OK/Warn/Critical) + текст “чому” при hover (не лише колір)
- **Деталізація**: кожна метрика веде в список “offenders” з сортуванням за impact
- **Безпека**: технічні payload/logs ховати за ролями, показувати редактовані/масковані фрагменти

---

## 6. Integration with Case Cockpit

Engineering Dashboard інтегрується з [Case Cockpit](../core/04_case_cockpit_ux.md):

| Взаємодія | Напрямок | Опис |
|-----------|----------|------|
| Drill-down | Eng Dashboard → Cockpit | Click на case_id відкриває Case Cockpit |
| AI Run details | Eng Dashboard → Cockpit | Click на ai_run → Timeline з подією `AI_RUN_COMPLETED` |
| Integration failure | Eng Dashboard → Cockpit | Click на integration_id → Timeline з подією `INTEGRATION_FAILED` |
| Context | Cockpit ← Eng Dashboard | Cockpit може показувати "ви тут з Engineering Dashboard" |

---

## 7. Definition of Done (v0)

- [ ] Є окремий дашборд (навігація/URL) для Engineering
- [ ] RLS: доступ лише для `ENGINEER`, `ADMIN` (partial для `OPS_LEAD`)
- [ ] KPI Tiles: failures, confidence, low-confidence flags, AI acceptance, no-edit rate (AI)
- [ ] Integrations: топ помилок (з `error_code`) + список подій `INTEGRATION_FAILED`
- [ ] Quality Loop: топ редагованих полів (або хоча б проксі-метрика "has_edits" по типах)
- [ ] AI Quality: показувати severity для risk flags
- [ ] Drill-down: з будь-якого блоку можна перейти до конкретного кейса/події/ai_run

