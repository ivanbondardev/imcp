# Approvals Specification

Централізований інтерфейс для Human Approval Gates.

---

## Мета

Approvals — черга рішень, що відповідає на питання: **"Що потребує мого рішення?"**

### Основні задачі
- **Прийняття рішень**: Approve / Edit & Approve / Reject
- **Огляд черги**: всі pending approvals в одному місці
- **Контекст рішення**: request_snapshot, ризики, AI-аналіз
- **Історія рішень**: нещодавно прийняті рішення
- **Пріоритезація**: SLA-критичні approvals першими

---

## Approvals ≠ Case Cockpit

| Approvals Page | Case Cockpit |
|----------------|--------------|
| Усі pending approvals | Один кейс |
| "Що потребує рішення?" | "Що відбувається в цьому кейсі?" |
| Inline actions | Повний контекст |

---

## Human Approval Gates — Концепція

Approval = контрольна точка де:
1. **Система підготувала** draft (quote, BL, dims тощо)
2. **Людина приймає рішення**: Approve / Edit & Approve / Reject

Кожне рішення фіксується в audit trail.

---

## Інформаційна архітектура

### Pending Section
Картки pending approvals:
- Approval type + icon
- Status badge
- SLA badge (якщо критичний)
- Case link
- Title та context
- Request snapshot
- Risk flags
- Action buttons

### Recently Decided Section
Таблиця нещодавніх рішень:
- Type
- Case
- Decision (APPROVED / REJECTED)
- Who
- When

---

## Approval Card

Кожна картка містить:
- **Header**: type, status, case link
- **Body**: human-readable опис, context, snapshot
- **Risk flags**: виділені попередження
- **Actions**: Approve, Edit & Approve, Reject

### Request Snapshot
Відображення підготовлених системою даних.
Формат залежить від типу approval.

---

## UX принципи

### Показувати контекст рішення
Request snapshot завжди видимий.

### Risk flags виділені
Попередження про ризики чітко видимі.

### Inline actions
Кнопки дій прямо на картці.

### SLA пріоритезація
Термінові approvals візуально виділені та йдуть першими.

### Audit trail
Кожне рішення логується з actor, timestamp, reason.

---

## Approval Types (F1_SEA_IMPORT)

| Type | Опис |
|------|------|
| `QUOTE_APPROVAL` | Підтвердження калькуляції |
| `DIMS_CHANGE_APPROVAL` | Зміна габаритів |
| `BL_DRAFT_APPROVAL` | Перевірка коносаменту |
| `ONEC_DEAL_CREATE_APPROVAL` | Створення угоди в 1C |
| `POA_SCAN_APPROVAL` | Перевірка довіреності |

Кожен тип має свої thresholds для warning/critical.

---

## Reject Flow

При reject обов'язковий коментар з причиною (мінімальна довжина).

---

## Core залежності

Дивись [03_approval_pattern.md](../core/03_approval_pattern.md) для:
- Approval lifecycle
- Status transitions
- Snapshot semantics
