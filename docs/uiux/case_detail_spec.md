# Case Detail Specification

Case Cockpit — основний робочий екран менеджера для роботи з кейсом.

---

## Мета

Case Cockpit — "кабіна пілота", що відповідає на питання: **"Що робити далі в цьому кейсі?"**

### Основні задачі
- **Зрозуміти контекст**: повна картина кейса
- **Виконати наступну дію**: NBA показує що робити
- **Прийняти рішення**: Approve / Edit / Reject drafts
- **Переглянути документи**: з статусами верифікації
- **Відстежити історію**: timeline подій

---

## Інформаційна архітектура

### Case Header
Ключова інформація про кейс:
- Case ID + type
- State + Status badges
- SLA indicator
- Owner
- Priority
- Client

### Next Best Action (NBA)
Виділена картка з поточною дією:
- Опис що потрібно зробити
- Контекст (ключові дані)
- Risk warnings якщо є
- Action buttons: Approve / Edit & Approve / Reject

NBA завжди видимий та візуально акцентований.

### Main Work Area
- Quote/Draft preview
- Pending approvals
- Documents section
- Timeline (останні події)

### Context Sidebar
Контекстна інформація:
- Cargo summary
- Route info
- Client info
- Key dates
- Risk flags
- Integrations status

---

## UX принципи

### NBA завжди видимий
Наступна дія очевидна з першого погляду.

### Draft-First Interaction
Система готує draft → людина верифікує/підтверджує.

### Context accessible
Вся необхідна інформація на одному екрані.

### Audit trail
Timeline показує історію рішень та подій.

### Human-in-the-Loop
Критичні рішення проходять через Approval Gates.

---

## Approval Actions

### Approve Flow
1. Click Approve
2. Confirmation modal
3. API call
4. State change + NBA update

### Edit & Approve Flow
1. Click Edit & Approve
2. Modal з editable полями
3. Save & Approve
4. State change

### Reject Flow
1. Click Reject
2. Modal з причиною (required)
3. Confirm
4. State rollback

---

## Documents Section

Список документів кейса:
- File name + type badge
- Source (CLIENT, ZED, BROKER)
- Status badge
- AI confidence indicator (якщо є)
- View / Verify actions

---

## Timeline Section

Хронологічний список подій:
- Actor type marker (HUMAN / SYSTEM / AI / INTEGRATION)
- Timestamp
- Event type
- Details

Link "Всі події" веде на Timeline page.

---

## Core залежності

Дивись [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) для деталей UX принципів.
Дивись [03_approval_pattern.md](../core/03_approval_pattern.md) для approval semantics.
