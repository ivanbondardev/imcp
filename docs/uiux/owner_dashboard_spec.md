# Owner Dashboard Specification

Панель управління для бізнес-овнерів та операційних керівників.

---

## Мета

Owner Dashboard — "контрольна вежа", що відповідає на питання: **"Де буксує потік кейсів і що робити?"**

### Основні задачі
- **Здоров'я операцій**: throughput, bottlenecks, SLA compliance
- **Контроль ризиків**: at-risk кейси, ескалації, blocked
- **Ефект від IMCP**: ROI, якість AI drafts
- **Owner-level рішення**: approvals високого рівня

---

## Owner Dashboard ≠ Case Cockpit

| Case Cockpit | Owner Dashboard |
|--------------|-----------------|
| Один кейс | Весь потік кейсів |
| Менеджер | Business Owner, Ops Lead |
| "Що робити далі в кейсі?" | "Де буксує система?" |

---

## Scope

Owner Dashboard показує **операційні bottlenecks**, а не технічні.

Engineering/System insights (AI quality, integration errors, debug) — окремий дашборд.

---

## Термінологія

| Термін | Опис |
|--------|------|
| `status` | Технічний агрегат: OPEN, BLOCKED, DONE, ARCHIVED |
| `state` | Бізнес-стан в state machine |
| **Active cases** | `status IN ('OPEN', 'BLOCKED')` |
| **At Risk** | SLA + pending approvals + risks + blocked |

---

## Інформаційна архітектура

### Operational Overview (KPI Tiles)
- Active cases
- At risk (з пороговими значеннями)
- Pending approvals
- Time-to-quote
- Approval latency
- AI acceptance rate

Кожна плитка клікабельна → drill-down.

### At Risk Queue
Кейси що потребують уваги:
- Case ID + type
- Risk reason
- State
- Owner
- SLA/deadline
- Action button

Критерії At Risk:
- SLA < 24h
- Pending approval > threshold
- Missing critical data
- Conflict detected
- Blocked external

### Approvals Health
- Pending total
- By type
- Avg time by type
- Overdue approvals

### Top Bottlenecks
- Top states з найбільшим avg time-in-state
- Top причини блокувань

### Escalations (MVP)
Owner-level рішення:
- High value shipment
- Non-standard margin
- Policy exception

Inline Approve / Reject + Comment.

---

## UX принципи

### Drill-down завжди доступний
Від метрики → до кейса за 2 кліки.

### Actionable metrics
KPI з можливістю діяти.

### Fatigue-aware
Показувати critical path, не все підряд.

### Owner ніколи не застрягає
Завжди можна перейти в кейс.

---

## Key Metrics

| Metric | Опис |
|--------|------|
| Active cases | status IN (OPEN, BLOCKED) |
| At risk | SLA/blocking/risks |
| Time-to-quote | Час до відправки ціни |
| Approval latency | P50/P90 часу до рішення |
| No-edit rate | % AI approvals без правок |

---

## Core залежності

Дивись [02_core_data_model.md](../core/02_core_data_model.md) для:
- Status vs State
- Approval thresholds
- Event types для метрик
