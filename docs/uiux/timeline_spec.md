# Timeline / Event Log Specification

Append-only audit log усіх подій платформи.

---

## Мета

Timeline — централізований журнал подій, що відповідає на питання: **"Що відбувалося в системі?"**

### Основні задачі
- **Аудит**: хто, коли, що, на основі чого
- **Розслідування**: пошук подій при інцидентах
- **Моніторинг**: real-time потік подій
- **Compliance**: незмінний audit trail
- **Debugging**: integration та AI events

---

## Timeline ≠ Case History

| Timeline | Case History (в Cockpit) |
|----------|--------------------------|
| Усі кейси платформи | Один кейс |
| Потік подій системи | Історія рішень |
| Ops Lead, Admin, Compliance | Менеджер кейса |

---

## Інформаційна архітектура

### KPI блок
- Подій сьогодні (+ trend vs вчора)
- HUMAN events (% від усіх)
- SYSTEM events
- AI events

### Event Timeline
Хронологічний список подій з date separators.

Кожна подія:
- Actor type marker (HUMAN / SYSTEM / AI / INTEGRATION)
- Timestamp
- Event type
- Details
- Case link

---

## Actor Types

| Actor | Опис |
|-------|------|
| `HUMAN` | Дії менеджера |
| `SYSTEM` | n8n workflows, triggers |
| `AI` | Extraction, generation |
| `INTEGRATION` | 1C, API callbacks |

Кожен тип має візуально відмінний marker.

---

## Event Types

### Core
- `CASE_CREATED`
- `STATE_CHANGED`
- `STATUS_CHANGED`

### Approval
- `APPROVAL_CREATED`
- `APPROVAL_APPROVED`
- `APPROVAL_REJECTED`

### Document
- `DOC_UPLOADED`
- `DOC_EXTRACTED`
- `DOC_VERIFIED`

### Integration
- `INTEGRATION_STARTED`
- `INTEGRATION_SUCCESS`
- `INTEGRATION_FAILED`

### Communication
- `EMAIL_SENT`
- `NOTIFICATION_SENT`

---

## UX принципи

### Immutability
Події не можна змінити чи видалити.

### Transparency
Повний контекст кожної події.

### Accountability
Чітке розділення actor types.

### Accessibility
Швидкий пошук і фільтрація.

---

## Фільтри

Quick filters по actor type:
- HUMAN
- SYSTEM
- AI
- INTEGRATION

Date range filter (MVP).

---

## Date Grouping

| Умова | Label |
|-------|-------|
| Сьогодні | "Сьогодні" |
| Вчора | "Вчора" |
| Цього тижня | День тижня |
| Раніше | Повна дата |

---

## Core залежності

Дивись [02_core_data_model.md](../core/02_core_data_model.md) для:
- `case_events` table schema
- Event Taxonomy
- Actor types enum
