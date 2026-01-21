# Personal Settings Specification

Персональні налаштування менеджера.

---

## Scope / Non-scope (для продуктового ревʼю)

**Scope (P0):**
- Зменшення шуму (notifications) + базові вподобання робочого середовища
- Налаштування inbox/views без зміни бізнес‑процесу

**Non-scope (P1+):**
- Зміна approval gates / state machine / required fields (це не налаштування користувача)

---

## MVP (P0) vs Next (P1)

**P0:**
- Quiet hours + locked critical events
- Inbox display (фільтри/сортування/колонки)
- UI density

**P1:**
- Розширені правила нотифікацій (по case_type, по ролі, по SLA tiers)

---

## Entry points / Deep links

- `/settings/personal` (вхід з навігації)

---

## Мета

Personal Settings — інструмент адаптації робочого середовища для ефективності та зменшення когнітивного навантаження.

### Основні цілі
- **Зменшення шуму**: фільтрувати нерелевантні сповіщення
- **Швидкість реакції**: пріоритезувати critical events
- **Адаптація**: налаштувати inbox та UI під себе
- **Зручність**: персоналізовані шаблони

---

## Що Settings змінюють

| Змінюється | Як саме |
|------------|---------|
| Notifications | Канали, типи подій, quiet hours |
| Inbox display | Фільтри, сортування, колонки |
| Draft templates | Шаблони повідомлень |
| UI density | Compact / Comfortable |

---

## Що Settings НЕ змінюють

| НЕ змінюється | Причина |
|---------------|---------|
| State machine | Бізнес-процес єдиний |
| Approval gates | Обов'язкові контрольні точки |
| Required fields | Визначаються case_type |
| RLS policies | Права від ролі |
| SLA deadlines | Бізнес-правила |

---

## Інформаційна архітектура

### Notifications
- **Канали**: In-app, Email (instant/digest), Messenger (MVP)
- **Типи подій**: Critical (locked), Operational, Informational
- **Quiet hours**: start/end time + critical override

**Critical events** (не можна вимкнути):
- Approval required — завжди ON

### Inbox & Views
- Default filters: My cases only, At risk first, Pending approvals first
- Default sorting
- Visible columns

### Draft Templates
Шаблони повідомлень з placeholders:
- `{{case_id}}`, `{{client_name}}`, `{{quote_total}}` тощо

### UI Preferences (MVP)
- Density: compact / comfortable
- Panel states
- Theme

---

## Notifications — Event Types

> ⚠️ Важливо: нижче — **ключі налаштувань нотифікацій** (product-level).  
> Канонічні audit-події для Timeline живуть у `case_events.event_type` і мають формат `UPPER_SNAKE_CASE` (див. `docs/core/02_core_data_model.md#4-event-taxonomy`).
>
> Mapping робимо на рівні UI/notification-service:
> - `approval_required` → `APPROVAL_CREATED` (або derived condition “є pending approvals”)
> - `state_changed` → `STATE_CHANGED`
> - `doc_uploaded` → `DOC_UPLOADED`
> - `integration_failed` → `INTEGRATION_FAILED`
> - `ai_draft_generated` → `AI_RUN_COMPLETED` (run_type=GENERATE) / або derived від появи draft у `cases.computed.*`

### Critical
- `approval_required` — 🔒 locked ON
- `sla_risk_24h`
- `data_conflict`

### Operational
- `case_assigned`
- `state_changed`
- `doc_uploaded`
- `integration_failed`

### Informational
- `automation_completed`
- `ai_draft_generated`

---

## Quiet Hours

Режим тиші з можливістю:
- Встановити start/end time
- Critical events все одно приходять (critical_override locked ON)

Timezone береться з Profile.

---

## UX принципи

### Швидко
Мінімум кліків для налаштувань.

### Безпечно
Не можна вимкнути контрольні точки.

### Прозоро
Очевидно що зміниться.

### Відновлювано
Reset to defaults для кожної секції.

---

## Save Behavior

Explicit save з unsaved indicator:
- Button disabled якщо немає змін
- "Unsaved changes" badge
- Toast після збереження
- Error handling з retry

---

## Templates

### Placeholders
Whitelist дозволених placeholders:
- case_id, case_type, client_name, client_contact
- route_origin, route_destination
- sla_date, quote_total, quote_validity

Unknown placeholders → warning при збереженні.
Missing data → warning при використанні.

---

## Core залежності

Дивись [00_shared_mental_model.md](../core/00_shared_mental_model.md) для принципів:
- Fatigue-Aware Design
- Human-in-the-Loop
- Accountability

---

## Edge cases (P0)

- **Занадто “тихо”**: UI має попереджати, якщо користувач вимикає все окрім locked critical (ризик пропустити важливе)
- **Timezone/quiet hours**: явно показувати timezone і приклад “коли прийде наступний digest”

---

## Product Acceptance Checklist (P0)

- [ ] Critical events (locked) не можна вимкнути, і це зрозуміло в UI
- [ ] Quiet hours: користувач бачить timezone і приклад “коли буде тихо/коли прийдуть нотифікації”
- [ ] Зміни мають явний unsaved indicator і require explicit save
- [ ] Є “reset to defaults” (мінімально — для notifications та inbox display)
- [ ] Налаштування не дозволяють змінювати state machine / approval gates / required fields
