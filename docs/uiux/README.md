# IMCP UI/UX Documentation

Документація інтерфейсів платформи IMCP.

---

## Документи

| Документ | Опис |
|----------|------|
| [platform_ui_contracts.md](./platform_ui_contracts.md) | Глобальні патерни: навігація, авторизація, помилки, пошук |
| [ui_patterns.md](./ui_patterns.md) | Каталог повторюваних UI патернів (badges, ризики, approvals, стани, empty/loading/error) |
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

## Матриця відповідності (Core → UI/UX)

Таблиця для швидкого ревʼю узгодженості: **який core-принцип/контракт** і **де саме він реалізований в UI/UX спеках**.

| Core документ / контракт | Що це означає для UI | Де покрито в UI/UX |
|---|---|---|
| `00_shared_mental_model.md` — **Human-in-the-Loop, Draft-First, Transparency** | UI показує **дію + контекст + ризики**, AI не “маскує” невпевненість, рішення людини — фінальне | `case_detail_spec.md`, `approvals_spec.md`, `documents_spec.md`, `timeline_spec.md`, `personal_settings_spec.md` |
| `01_architecture_overview.md` — **UI Contract (can/can’t write)** | UI **не змінює** `cases.state/status/computed`; рішення людини пише в `approvals` (optimistic lock), події людини — `case_events` (`actor_type='HUMAN'`) | `case_detail_spec.md` (Approval flows), `documents_spec.md` (verification notes), цей README (контракт) |
| `02_core_data_model.md` — **State vs Status** | UI завжди відрізняє `state` (бізнес) і `status` (агрегат) | `case_detail_spec.md` (Header), `case_list_spec.md` (групи/колонки), `owner_dashboard_spec.md` (термінологія) |
| `02_core_data_model.md` — **Approvals: статуси/переходи/immutability** | `PENDING → APPROVED/REJECTED/CANCELLED`, `request_snapshot` immutable, рішення атомарне | `approvals_spec.md`, `case_detail_spec.md` |
| `02_core_data_model.md` — **Documents: статуси + source enum** | `UPLOADED/PROCESSING/VERIFIED/REPLACED/ARCHIVED`, `source` включає `SYSTEM|AI` | `documents_spec.md`, `case_detail_spec.md` (Documents section) |
| `02_core_data_model.md` — **Event Taxonomy (`case_events.event_type`)** | Timeline відображає **канонічні** події, append-only | `timeline_spec.md` |
| `03_approval_pattern.md` — **Approval як мікро‑процес (контекст/ризики/CTA/decision_comment)** | Approval не “ви впевнені?” модалка, а керований процес з контекстом і фіксацією рішення | `approvals_spec.md`, `case_detail_spec.md` |
| `04_case_cockpit_ux.md` — **Case Cockpit/NBA/Progressive Disclosure** | 1 екран на кейс, NBA 1–3 дії, мінімум когнітивного шуму | `case_detail_spec.md`, `case_list_spec.md`, `platform_ui_contracts.md` |
| `02_core_data_model.md` — **Roles/RLS** | UI не “вгадує” доступ, показ/дії залежать від ролі та RLS | `platform_ui_contracts.md` (RLS-aware), `system_engineering_dashboard_spec.md` (access), `owner_dashboard_spec.md` (scope) |
| `02_core_data_model.md` — **AI runs / quality loop signals** | Можливість drill‑down до AI/integration подій і метрик, без порушення приватності | `system_engineering_dashboard_spec.md`, `timeline_spec.md` |

---

## UX Priority Stack (для продуктового ревʼю)

Порядок пріоритетів — що “обовʼязково має працювати” в UX до того, як ми поліруємо деталі:

- **P0: Фокус і контроль**
  - NBA (Next Best Action) завжди відповідає “що робити далі?”
  - Approvals як керований мікро‑процес (контекст → ризики → рішення)
  - Auditability: Timeline як джерело довіри (“що сталося”)
- **P0: Anti-noise**
  - ризики/невпевненість показуються чітко, але без “спаму”
  - progressive disclosure для reasoning/деталей
- **P1: Прискорення**
  - глобальний пошук і keyboard-first flows
  - batch-операції для документів/черг
- **P1: Quality Loop**
  - метрики якості AI (no-edit/correction rate) + drill‑down
  - spot-check/deep verify режими, калібрування навантаження

---

## Глосарій UI термінів (коротко)

| Термін | Значення (для продукту) |
|---|---|
| `case_id` vs `case_number` | `case_number` — людський ID у UI; `case_id` — технічний UUID |
| `state` vs `status` | `state` — бізнес‑стан; `status` — агрегат `OPEN/BLOCKED/DONE/ARCHIVED` |
| `event_type` vs notification key | `event_type` — канонічна audit‑подія (Timeline); notification key — налаштування сповіщень (product-level) |
| “needs human review” | derived flag (не статус) — коли confidence/ризики вимагають уваги людини |

---

## Порядок читання

1. Core README → загальне розуміння
2. platform_ui_contracts.md → глобальні патерни
3. ui_patterns.md → канонічні UI патерни
3. ui_style_reference.md → дизайн-система
4. Специфікації екранів за потребою
