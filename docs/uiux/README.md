# IMCP UI/UX Documentation

Документація інтерфейсів платформи IMCP.

---

## Документи

| Документ | Опис |
|----------|------|
| [platform_ui_contracts.md](./platform_ui_contracts.md) | Глобальні патерни: навігація, авторизація, помилки, пошук |
| [ui_style_reference.md](./ui_style_reference.md) | Дизайн-токени та візуальна мова |
| [case_list_spec.md](./case_list_spec.md) | Операційна черга кейсів |
| [case_detail_spec.md](./case_detail_spec.md) | Case Cockpit — екран кейса |
| [documents_spec.md](./documents_spec.md) | Управління документами |
| [approvals_spec.md](./approvals_spec.md) | Human Approval Gates |
| [timeline_spec.md](./timeline_spec.md) | Timeline / Event Log |
| [owner_dashboard_spec.md](./owner_dashboard_spec.md) | Дашборд для бізнес-овнерів |
| [personal_settings_spec.md](./personal_settings_spec.md) | Персональні налаштування |
| [system_engineering_dashboard_spec.md](./system_engineering_dashboard_spec.md) | Інженерний дашборд |

---

## Core залежності

UI/UX документи базуються на:

- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — UX-принципи, Human-in-the-Loop
- [01_architecture_overview.md](../core/01_architecture_overview.md) — UI Contract
- [02_core_data_model.md](../core/02_core_data_model.md) — Структура даних, enums
- [03_approval_pattern.md](../core/03_approval_pattern.md) — Approval semantics
- [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) — Case Cockpit принципи

---

## Ключові контракти з Core

### State vs Status
- `cases.state` — бізнес-стан (наприклад `WAITING_CLIENT_INFO`)
- `cases.status` — технічний агрегат: `OPEN | BLOCKED | DONE | ARCHIVED`

### Approval Status
- `approvals.status`: `PENDING | APPROVED | REJECTED | CANCELLED`

### Document Status
- `documents.status`: `UPLOADED | PROCESSING | VERIFIED | REPLACED | ARCHIVED`

### Actor Types
- `case_events.actor_type`: `HUMAN | SYSTEM | AI | INTEGRATION`

### UI Contract
UI може:
- UPDATE `cases.payload.*`
- INSERT `approvals` зі `status='PENDING'`
- UPDATE `approvals` для рішення (атомарно, лише якщо `status='PENDING'`)
- INSERT `documents` та `case_events` (для `actor_type='HUMAN'`)

UI не може:
- UPDATE `cases.state` / `cases.status` / `cases.computed`
- DELETE/UPDATE `case_events`

---

## Порядок читання

1. Core README → загальне розуміння
2. platform_ui_contracts.md → глобальні патерни
3. ui_style_reference.md → дизайн-система
4. Специфікації екранів за потребою
