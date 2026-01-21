# System / Engineering Dashboard Specification

Інженерний дашборд для моніторингу платформи та AI якості.

---

## Мета

Engineering Dashboard — "контрольна вежа" для інженерів, що відповідає на:

- **Чому** система буксує (помилки інтеграцій, деградація AI)
- **Що саме** погано (які поля правлять, де падає впевненість)
- **Де дебажити** (drill-down до конкретних подій)

---

## Розмежування

| Owner Dashboard | Engineering Dashboard |
|-----------------|----------------------|
| Операційні bottlenecks | Технічні проблеми |
| Бізнес-метрики | AI quality, integration health |
| Owner, Ops Lead | Engineer, Admin |

---

## Access (RLS)

| Роль | Доступ |
|------|--------|
| ENGINEER | Full access |
| ADMIN | Full + config changes |
| OPS_LEAD | Partial (агрегати без raw logs) |
| MANAGER/OWNER | No access |

---

## Інформаційна архітектура

### System Overview (KPI)
- Integration failures (24h)
- Failure rate by integration
- AI confidence (avg / p10)
- LOW_CONFIDENCE flagged
- AI acceptance rate
- No-edit rate (AI)

### AI Quality
- Confidence trend over time
- Breakdown by case_type, draft type
- Risk flags: LOW_CONFIDENCE, DOC_PARSE_FAILED, CONFLICT_DETECTED
- AI Runs log (MVP)

### Quality Loop
Аналіз розбіжностей між `request_snapshot` і `decision_snapshot`:
- Most edited fields
- Edit frequency by approval_type
- Drill-down до кейсів

### Integrations
- Top errors by `error_code`
- Heatmap failures by hour
- Breakdown by `integration_type`
- Recent failures list

Error codes:
- TIMEOUT
- AUTH
- VALIDATION
- RATE_LIMIT
- UNKNOWN

### Data Freshness
- Subscription health
- Polling fallback rate
- Max lag

---

## Human Expertise Signals

Патерни де люди "рятують" систему:
- Які поля найчастіше правлять
- У яких контекстах
- Топ комбінації field × case_type × approval_type

**Без персоналізації** — лише агрегати по патернах.

---

## Decision Reasons (MVP)

Чому люди відхиляють/правлять:
- reason_code taxonomy
- reason distribution
- Drill-down by reason

Reason codes:
- MISSING_CONTEXT
- POLICY_EXCEPTION
- DATA_CONFLICT
- CLIENT_REQUEST
- RISK_TOO_HIGH
- PRICE_ADJUSTMENT
- DOC_ISSUE
- INTEGRATION_ERROR
- OTHER

---

## Improvement Impact (MVP)

Tracking ефекту від змін:
- Changelog (prompt/rule/integration changes)
- Before/after metrics
- Regressions

---

## UX принципи

### Drill-down
Від метрики → до кейса/події/ai_run.

### Severity levels
3-рівневі сигнали: OK / Warn / Critical.

### Privacy
Raw payloads/logs за ролями.

---

## Integration з Case Cockpit

| Дія | Результат |
|-----|-----------|
| Click case_id | → Case Cockpit |
| Click ai_run | → Timeline з AI_RUN_COMPLETED |
| Click integration_id | → Timeline з INTEGRATION_FAILED |

---

## Core залежності

Дивись [02_core_data_model.md](../core/02_core_data_model.md) для:
- `case_events` Event Taxonomy
- `ai_runs` schema
- `integrations` status
- RLS policies
