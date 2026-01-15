# Case List Specification
## IMCP — Операційна черга кейсів для менеджерів

**Версія:** 1.0  
**Статус:** Spec (PoC → MVP)  
**Тип документа:** Spec  
**Аудиторія:** Product, UX/UI, Frontend, Operations Manager  
**Changelog:**  
- v1.0 — initial spec based on demo/case-list.html

**Попередні документи:**  
- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](../core/01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](../core/02_core_data_model.md) — модель даних  
- [03_approval_pattern.md](../core/03_approval_pattern.md) — патерн підтверджень  
- [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) — UX/UI Case Cockpit  

**Пов'язані документи:**  
- [UI Style Reference](./ui_style_reference.md) — дизайн-токени  
- [Owner Dashboard Spec](./owner_dashboard_spec.md) — специфікація Owner Dashboard  
- [Personal Settings Spec](./personal_settings_spec.md) — персональні налаштування  

---

## 1. Purpose — Навіщо потрібна сторінка Case List

Case List в IMCP — це **операційна черга менеджера**, центральний хаб для роботи з усіма активними кейсами.

### 1.1 Основні задачі менеджера

| Задача | Опис |
|--------|------|
| **Пріоритезація** | Зрозуміти що робити першим (approvals, SLA at risk) |
| **Огляд потоку** | Бачити всі активні кейси та їх статуси |
| **Швидкий доступ** | Перейти до конкретного кейса за 1 клік |
| **Створення кейсів** | Ініціювати нові кейси різних типів |
| **Фільтрація** | Знайти потрібні кейси серед потоку |

### 1.2 Case List ≠ Owner Dashboard

| Аспект | Case List | Owner Dashboard |
|--------|-----------|-----------------|
| **Фокус** | Мої кейси, моя черга | Весь потік організації |
| **Аудиторія** | Менеджер/оператор | Business Owner, Ops Lead |
| **Ключове питання** | "Що мені робити зараз?" | "Де буксує система?" |
| **Дії** | Відкрити кейс, створити кейс | Escalations, reassign, policy |
| **Метрики** | Мої KPI (pending, at risk) | Org-wide KPI |

> **Case List** — це "робочий стіл менеджера", а не "контрольна вежа".

### 1.3 Відповідність Shared Mental Model

Case List реалізує принципи [Shared Mental Model](../core/00_shared_mental_model.md):

| Принцип | Реалізація в Case List |
|---------|------------------------|
| **Human-in-the-Loop** | Pending approvals виділені, потребують дії |
| **Fatigue-Aware Design** | Групування за пріоритетом, не весь список |
| **Actionable Interface** | Кожен рядок клікабельний → Case Cockpit |
| **NBA (Next Best Action)** | Колонка NBA показує наступний крок |
| **Clear Status** | Status badges, SLA indicators |

---

## 2. Scope (PoC vs MVP)

### 2.1 PoC (Minimal Viable)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| **Stats Overview** | HIGH | 4 KPI tiles (approvals, SLA, active, completed) |
| **Grouped Sections** | HIGH | Потребують підтвердження, В роботі, Очікують |
| **Case Table** | HIGH | Клікабельні рядки з key columns |
| **New Case Modal** | HIGH | Створення кейса з вибором типу |
| **Basic Filters** | MEDIUM | Sidebar quick filters |

### 2.2 MVP (Extended)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| PoC features | — | Все з PoC |
| **Advanced Filters** | MEDIUM | Filter modal з multiple criteria |
| **Search** | MEDIUM | Пошук по case_number, client |
| **Column Customization** | LOW | Налаштування видимих колонок |
| **Bulk Actions** | LOW | Multi-select + batch operations |
| **Saved Views** | LOW | Інтеграція з Personal Settings |

### 2.3 v2 (Scale)

| Функціональність | Опис |
|------------------|------|
| Kanban view | Drag-and-drop по state |
| Calendar view | Візуалізація по SLA deadline |
| Export | CSV/Excel export |
| Keyboard shortcuts | Vim-like navigation |

---

## 3. Information Architecture

### 3.1 Загальна структура сторінки

```
┌──────────────────────────────────────────────────────────────┐
│ Sidebar Navigation                                            │
├──────────────────────────────────────────────────────────────┤
│ Header: "Кейси" + Actions (Filters, New Case)                 │
├──────────────────────────────────────────────────────────────┤
│ Stats Grid (4 KPI tiles)                                      │
├──────────────────────────────────────────────────────────────┤
│ Section: "Потребують підтвердження" (Pending Approvals)       │
│ └── Table with approval cases                                 │
├──────────────────────────────────────────────────────────────┤
│ Section: "В роботі" (In Progress)                             │
│ └── Table with active cases                                   │
├──────────────────────────────────────────────────────────────┤
│ Section: "Очікують зовнішню відповідь" (Waiting External)     │
│ └── Table with blocked cases                                  │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Sidebar Navigation

| Секція | Пункти | Badge |
|--------|--------|-------|
| **Main** | Owner Dashboard, Кейси, Підтвердження, Документи, Таймлайн | Кейси: count, Підтвердження: pending count |
| **Фільтри** | Мої кейси, Потребують уваги, SLA скоро | Counts |
| **Налаштування** | Персональні | — |

### 3.3 Drill-down Model

```
Case List → Case Row Click → Case Cockpit
     ↓
Stats Tile Click → Filtered Case List
     ↓
New Case Button → New Case Modal → Case Cockpit (new case)
```

---

## 4. Stats Overview (KPI Tiles)

### 4.1 KPI Tiles Specification

| KPI | Label | Опис | Порогове значення | Icon |
|-----|-------|------|-------------------|------|
| **Потребують approval** | Pending Approvals | Кейси з `PENDING` approval | > 0 = warning | Checkmark |
| **SLA at risk** | SLA < 24h | Кейси з дедлайном < 24 год | > 0 = danger | Clock |
| **Активних кейсів** | Active Cases | `status IN ('OPEN', 'BLOCKED')` | — | Package |
| **Завершено цього місяця** | Completed this month | `status = 'DONE'` + this month | vs попередній = trend | Check circle |

### 4.2 KPI Tile Behavior

| Tile | Click Action | Color Logic |
|------|--------------|-------------|
| Потребують approval | → Filter to approval cases | 🟡 warning if > 0 |
| SLA at risk | → Filter to at-risk cases | 🔴 danger if > 0 |
| Активних кейсів | → Show all active | 🔵 primary |
| Завершено | → Filter to completed | 🟢 success, show trend |

### 4.3 SQL Queries для KPI

```sql
-- Потребують approval (для поточного менеджера)
SELECT COUNT(*) 
FROM cases c
JOIN approvals a ON a.case_id = c.id
WHERE c.owner_user_id = $user_id
  AND a.status = 'PENDING';

-- SLA at risk
SELECT COUNT(*) 
FROM cases 
WHERE owner_user_id = $user_id
  AND status IN ('OPEN', 'BLOCKED')
  AND sla_deadline < now() + interval '24 hours';

-- Активних кейсів
SELECT COUNT(*) 
FROM cases 
WHERE owner_user_id = $user_id
  AND status IN ('OPEN', 'BLOCKED');

-- Завершено цього місяця
SELECT COUNT(*) 
FROM cases 
WHERE owner_user_id = $user_id
  AND status = 'DONE'
  AND updated_at >= date_trunc('month', now());
```

---

## 5. Case Sections Specification

### 5.1 Секція "Потребують підтвердження"

**Мета:** Показати кейси, які чекають на approval дію від менеджера.

#### Критерії включення

```sql
SELECT c.* 
FROM cases c
JOIN approvals a ON a.case_id = c.id
WHERE c.owner_user_id = $user_id
  AND a.status = 'PENDING'
ORDER BY a.requested_at ASC;  -- FIFO
```

#### Колонки таблиці

| Колонка | Ключ | Опис | Приклад |
|---------|------|------|---------|
| Case ID | `case_number` | Badge + ID | `[SEA] F1-SEA-2026-02451` |
| Клієнт | `client_name` | Назва клієнта | ACME Ltd |
| Стан | `state` | State badge | `QUOTE_APPROVAL_PENDING` |
| Approval | `approval_type` | Type badge | `QUOTE_APPROVAL` |
| SLA | `sla_deadline` | Relative time + indicator | `2d 4h` / 🟡 `8h` |
| Owner | `owner` | Avatar + name | `[ІП] Іван П.` |
| Action | — | Primary button | `[Відкрити]` |

#### SLA Indicator Colors

| Умова | Колір | CSS Class |
|-------|-------|-----------|
| SLA > 48h | default (text-secondary) | — |
| SLA 24h-48h | 🟡 warning | `sla-warning` |
| SLA < 24h | 🔴 danger | `sla-danger` |
| SLA overdue | 🔴 danger + bold | `sla-overdue` |

---

### 5.2 Секція "В роботі"

**Мета:** Активні кейси, які прогресують нормально.

#### Критерії включення

```sql
SELECT c.* 
FROM cases c
LEFT JOIN approvals a ON a.case_id = c.id AND a.status = 'PENDING'
WHERE c.owner_user_id = $user_id
  AND c.status = 'OPEN'
  AND a.id IS NULL  -- Немає pending approvals
ORDER BY c.sla_deadline ASC;
```

#### Колонки таблиці

| Колонка | Ключ | Опис | Приклад |
|---------|------|------|---------|
| Case ID | `case_number` | Badge + ID | `[SEA] F1-SEA-2026-02450` |
| Клієнт | `client_name` | Назва клієнта | Світ Товарів |
| Стан | `state` | State badge | `WAITING_CLIENT_INFO` |
| NBA | `nba` | Next Best Action | Очікуємо відповідь клієнта |
| SLA | `sla_deadline` | Relative time | `3d 8h` |
| Owner | `owner` | Avatar + name | `[ІП] Іван П.` |
| Action | — | Secondary button | `[Відкрити]` |

---

### 5.3 Секція "Очікують зовнішню відповідь"

**Мета:** Заблоковані кейси, які чекають на зовнішню сторону.

#### Критерії включення

```sql
SELECT c.* 
FROM cases c
WHERE c.owner_user_id = $user_id
  AND c.status = 'BLOCKED'
ORDER BY c.updated_at DESC;
```

#### Візуальне оформлення

- Рядки мають `opacity: 0.7` — "приглушені"
- Button style: `btn-ghost` замість `btn-primary`

#### Колонки таблиці

| Колонка | Ключ | Опис | Приклад |
|---------|------|------|---------|
| Case ID | `case_number` | Badge + ID | `[SEA] F1-SEA-2026-02442` |
| Клієнт | `client_name` | Назва клієнта | Toys World |
| Стан | `state` | State badge (blocked style) | `WAITING_WAREHOUSE_DIMS` |
| Очікуємо | `waiting_for` | На кого чекаємо | Yiwu Warehouse |
| Очікування | `waiting_duration` | Час очікування | 3 дні |
| Owner | `owner` | Avatar + name | `[ІП] Іван П.` |
| Action | — | Ghost button | `[Відкрити]` |

---

## 6. New Case Modal

### 6.1 Modal Structure

```
┌─────────────────────────────────────────────────────────────┐
│ Створити новий кейс                                    [×]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Тип кейсу *                                                 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [Оберіть тип кейсу... ▾]                                │ │
│ │ 🚢 Морський імпорт (Китай → Україна)                    │ │
│ │ ✈️ Авіа імпорт                                          │ │
│ │ 🚂 Залізничний імпорт                                   │ │
│ │ 🚛 Авто експорт                                         │ │
│ └─────────────────────────────────────────────────────────┘ │
│ [Help text про вибраний тип]                                │
│                                                             │
│ Клієнт *                                                    │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [Оберіть клієнта... ▾]                                  │ │
│ │ ACME Ltd                                                │ │
│ │ Global Trade LLC                                        │ │
│ │ + Додати нового клієнта                                 │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ [New Client Fields - conditional]                           │
│                                                             │
│ [Dynamic Case Type Fields - conditional]                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                           [Скасувати]  [+ Створити кейс]    │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Case Types та Dynamic Fields

| Case Type | Code | Icon | Dynamic Fields |
|-----------|------|------|----------------|
| Морський імпорт | `F1_SEA_IMPORT` | 🚢 | origin_warehouse, ready_date, cargo_description, packages_count, broker_owner, dangerous_goods, stackable |
| Авіа імпорт | `F1_AIR_IMPORT` | ✈️ | origin_city, ready_date, cargo_description, weight_kg, urgency |
| Залізничний імпорт | `F1_RAIL_IMPORT` | 🚂 | origin_station, dest_station, container_type |
| Авто експорт | `F1_AUTO_EXPORT` | 🚛 | origin_city, dest_country, load_date, truck_type |

### 6.3 F1_SEA_IMPORT Fields (приклад)

| Поле | Type | Required | Options |
|------|------|----------|---------|
| Склад відправлення | Select | ✅ | Yiwu, Shenzhen, Guangzhou, Other |
| Дата готовності | Date | ❌ | — |
| Опис вантажу | Textarea | ❌ | — |
| Кількість місць | Number | ❌ | min: 1 |
| Брокер | Select | ❌ | Наш брокер, Брокер клієнта |
| Небезпечний вантаж | Checkbox | ❌ | default: false |
| Можна штабелювати | Checkbox | ❌ | default: true |

### 6.4 New Client Inline Form

При виборі "+ Додати нового клієнта":

| Поле | Type | Required |
|------|------|----------|
| Назва компанії | Text | ✅ |
| Контактна особа | Text | ❌ |
| Email | Email | ❌ |
| Телефон | Tel | ❌ |

### 6.5 Create Case Flow

```
1. User selects case_type → dynamic fields appear
2. User selects/creates client
3. User fills required fields
4. Click "Створити кейс"
   ↓
5. API: POST /cases with payload
   ↓
6. Success → Redirect to Case Cockpit
   ↓
   Toast: "Кейс F1-SEA-2026-XXXXX створено"
```

### 6.6 Validation Rules

| Правило | Коли | Повідомлення |
|---------|------|--------------|
| case_type required | Submit without type | "Будь ласка, оберіть тип кейсу" |
| client required | Submit without client | "Будь ласка, оберіть або додайте клієнта" |
| new_client_name required | client = 'new' + empty name | "Будь ласка, вкажіть назву нового клієнта" |
| origin_warehouse required | F1_SEA_IMPORT + empty | "Оберіть склад відправлення" |

---

## 7. Sidebar Quick Filters

### 7.1 Filter Options

| Фільтр | Icon | Опис | Query |
|--------|------|------|-------|
| Мої кейси | ✓ | Кейси де я owner | `owner_user_id = $user_id` |
| Потребують уваги | ⚠ | Pending approvals + at-risk | `has_pending OR is_at_risk` |
| SLA скоро | 📅 | SLA < 48h | `sla_deadline < now() + 48h` |

### 7.2 Filter Behavior

- Фільтри є **адитивними** (AND логіка)
- Активний фільтр підсвічується
- Badge показує count matching кейсів
- Клік на активний фільтр — деактивує його

---

## 8. Wireframe — Case List (PoC)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Кейси                                                    [User Avatar]│
├────────────────────┬─────────────────────────────────────────────────────────┤
│ ІМСР               │ Кейси                                                   │
│ Case Platform      │ Операційна черга — 12 активних кейсів                   │
│ ─────────────────  │                              [Фільтри]  [+ Новий кейс]  │
│                    │ ───────────────────────────────────────────────────────│
│ □ Owner Dashboard  │                                                         │
│ ■ Кейси        12  │ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│ □ Підтвердження 3  │ │ Потребують│ │ SLA at    │ │ Активних  │ │ Завершено │ │
│ □ Документи        │ │ approval  │ │ risk      │ │ кейсів    │ │ цього міс │ │
│ □ Таймлайн         │ │           │ │           │ │           │ │           │ │
│                    │ │     3     │ │     2     │ │    12     │ │    28     │ │
│ ─────────────────  │ │    🟡     │ │    🔴     │ │    🔵     │ │    🟢     │ │
│ Фільтри            │ └───────────┘ └───────────┘ └───────────┘ └───────────┘ │
│ ─────────────────  │                                                         │
│ ✓ Мої кейси        │ ─────────────────────────────────────────────────────── │
│ ⚠ Потребують       │ ☑ ПОТРЕБУЮТЬ ПІДТВЕРДЖЕННЯ                          3   │
│   уваги         5  │ ─────────────────────────────────────────────────────── │
│ 📅 SLA скоро    2  │                                                         │
│                    │ ┌─────────────────────────────────────────────────────┐ │
│ ─────────────────  │ │ Case ID      │ Клієнт     │ Стан      │ Approval   │ │
│ Налаштування       │ │ ─────────────────────────────────────────────────── │ │
│ ─────────────────  │ │ [SEA]        │ ACME Ltd   │ QUOTE_    │ QUOTE_     │ │
│ ⚙ Персональні      │ │ F1-SEA-..51  │            │ APPROVAL_ │ APPROVAL   │ │
│                    │ │              │            │ PENDING   │            │ │
│                    │ │ SLA: 2d 4h 🟡│ Owner: ІП  │           │ [Відкрити] │ │
│                    │ │ ─────────────────────────────────────────────────── │ │
│                    │ │ [SEA]        │ Global     │ DIMS_     │ DIMS_      │ │
│                    │ │ F1-SEA-..48  │ Trade LLC  │ APPROVAL_ │ CHANGE_    │ │
│                    │ │              │            │ PENDING   │ APPROVAL   │ │
│                    │ │ SLA: 8h   🔴 │ Owner: МК  │           │ [Відкрити] │ │
│                    │ └─────────────────────────────────────────────────────┘ │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ ↻ В РОБОТІ                                          6   │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ ┌─────────────────────────────────────────────────────┐ │
│                    │ │ Case ID      │ Клієнт     │ Стан      │ NBA        │ │
│                    │ │ ─────────────────────────────────────────────────── │ │
│                    │ │ [SEA]        │ Світ       │ WAITING_  │ Очікуємо   │ │
│                    │ │ F1-SEA-..50  │ Товарів    │ CLIENT_   │ відповідь  │ │
│                    │ │              │            │ INFO      │ клієнта    │ │
│                    │ │ SLA: 3d 8h   │ Owner: ІП  │           │ [Відкрити] │ │
│                    │ └─────────────────────────────────────────────────────┘ │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ ⏱ ОЧІКУЮТЬ ЗОВНІШНЮ ВІДПОВІДЬ                       3   │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ ┌─────────────────────────────────────────────────────┐ │
│                    │ │ Case ID      │ Клієнт     │ Стан      │ Очікуємо   │ │
│                    │ │ ─────────────────────────────────────────────────── │ │
│                    │ │ [SEA]        │ Toys       │ WAITING_  │ Yiwu       │ │
│                    │ │ F1-SEA-..42  │ World      │ WAREHOUSE │ Warehouse  │ │
│                    │ │ (opacity .7) │            │ _DIMS     │ 3 дні      │ │
│                    │ │              │ Owner: ІП  │           │ [Відкрити] │ │
│                    │ └─────────────────────────────────────────────────────┘ │
│                    │                                                         │
└────────────────────┴─────────────────────────────────────────────────────────┘
```

---

## 9. UX Behavior & Interactions

### 9.1 Row Click Behavior

| Element | Action | Result |
|---------|--------|--------|
| Row (anywhere) | Click | → Navigate to Case Cockpit |
| "Відкрити" button | Click | → Navigate to Case Cockpit |

> **Примітка:** Весь рядок клікабельний. Button — для явного affordance.

### 9.2 Stats Tile Click

| Tile | Action |
|------|--------|
| Потребують approval | Filter to cases with pending approvals |
| SLA at risk | Filter to cases with SLA < 24h |
| Активних кейсів | Clear filters, show all active |
| Завершено | Show completed cases (status = DONE) |

### 9.3 New Case Modal

| Trigger | Behavior |
|---------|----------|
| Button click | Open modal with animation |
| ESC key | Close modal |
| Overlay click | Close modal |
| Submit success | Close modal + redirect to new case |
| Submit error | Show validation errors, keep modal open |

### 9.4 Empty States

| Section | Empty State |
|---------|-------------|
| Потребують підтвердження | "Немає кейсів, що потребують підтвердження 🎉" |
| В роботі | "Немає активних кейсів" |
| Очікують зовнішню відповідь | "Немає заблокованих кейсів" |
| All sections empty | "Створіть свій перший кейс" + CTA button |

### 9.5 Loading States

| Component | Loading Behavior |
|-----------|-----------------|
| Stats tiles | Skeleton loaders (4 boxes) |
| Sections | Skeleton table rows |
| New Case Modal | Submit button shows spinner |

### 9.6 Sorting

| Section | Default Sort | Custom Sort |
|---------|--------------|-------------|
| Потребують підтвердження | `approval.requested_at ASC` (FIFO) | — |
| В роботі | `sla_deadline ASC` | MVP: sortable headers |
| Очікують зовнішню відповідь | `updated_at DESC` | — |

---

## 10. Data Sources (Supabase)

### 10.1 Core Tables

| Таблиця | Дані для Case List |
|---------|-------------------|
| `cases` | All case data, status, state, owner |
| `approvals` | Pending approvals count, type |
| `clients` | Client name for display |
| `users` | Owner name, avatar |
| `case_type_configs` | Case type metadata |

### 10.2 Views (рекомендовано)

```sql
-- View for manager's case list
CREATE VIEW v_manager_cases AS
SELECT 
  c.id,
  c.case_number,
  c.case_type,
  c.state,
  c.status,
  c.sla_deadline,
  c.owner_user_id,
  c.org_id,
  c.payload->'client'->>'name' as client_name,
  c.computed->'nba'->>'action' as nba,
  c.updated_at,
  -- Pending approval info
  (
    SELECT jsonb_build_object(
      'id', a.id,
      'type', a.approval_type,
      'requested_at', a.requested_at
    )
    FROM approvals a
    WHERE a.case_id = c.id AND a.status = 'PENDING'
    ORDER BY a.requested_at ASC
    LIMIT 1
  ) as pending_approval,
  -- Owner info
  jsonb_build_object(
    'id', u.id,
    'name', u.raw_user_meta_data->>'full_name',
    'initials', u.raw_user_meta_data->>'initials'
  ) as owner,
  -- Computed flags
  CASE WHEN c.sla_deadline < now() + interval '24h' THEN true ELSE false END as is_sla_at_risk,
  CASE WHEN EXISTS (
    SELECT 1 FROM approvals a WHERE a.case_id = c.id AND a.status = 'PENDING'
  ) THEN true ELSE false END as has_pending_approval
FROM cases c
LEFT JOIN auth.users u ON u.id = c.owner_user_id
WHERE c.status IN ('OPEN', 'BLOCKED');
```

### 10.3 RLS Policies

```sql
-- Manager can see cases where they are owner or assignee
CREATE POLICY "manager_cases" ON cases
  FOR SELECT
  USING (
    owner_user_id = auth.uid()
    OR id IN (SELECT case_id FROM case_assignments WHERE user_id = auth.uid())
  );
```

---

## 11. UI Components

### 11.1 Case Type Badge

```css
.case-type-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  font-size: 11px;
  font-weight: 600;
  border-radius: var(--radius-sm);      /* 6px */
  background: var(--bg-panel);           /* #F3F3F3 */
  color: var(--text-secondary);          /* #5A5A5A */
  text-transform: uppercase;
}
```

### 11.2 Status Badge

```css
.status-badge {
  display: inline-flex;
  padding: 4px 10px;
  font-size: 11px;
  font-weight: 500;
  border-radius: var(--radius-sm);
  white-space: nowrap;
}

.status-badge.status-pending {
  background: #FEF3C7;                   /* Yellow light */
  color: #92400E;                        /* Amber dark */
}

.status-badge.status-open {
  background: #DBEAFE;                   /* Blue light */
  color: #1E40AF;                        /* Blue dark */
}

.status-badge.status-blocked {
  background: #FEE2E2;                   /* Red light */
  color: #991B1B;                        /* Red dark */
}
```

### 11.3 SLA Indicator

```css
.sla-warning {
  color: #F59E0B;                        /* Amber */
}

.sla-danger {
  color: #EF4444;                        /* Red */
  font-weight: 600;
}

.sla-overdue {
  color: #DC2626;                        /* Red darker */
  font-weight: 700;
}
```

### 11.4 Avatar

```css
.avatar {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: var(--accent);             /* #007ACC */
  color: white;
  font-size: 11px;
  font-weight: 600;
}

.avatar-sm {
  width: 24px;
  height: 24px;
}
```

### 11.5 Section Header

```css
.inbox-section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  font-weight: 600;
  color: var(--text);
}

.inbox-section-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 20px;
  height: 20px;
  padding: 0 6px;
  font-size: 12px;
  font-weight: 500;
  border-radius: 10px;
  background: var(--bg-panel);
  color: var(--text-secondary);
}
```

### 11.6 Modal Styles

```css
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: var(--bg-editor);          /* #FFFFFF */
  border-radius: var(--radius-xl);       /* 12px */
  box-shadow: var(--shadow-popup);
  width: 100%;
  max-width: 600px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  animation: modalSlideIn 0.2s ease;
}

@keyframes modalSlideIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid var(--divider);
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  padding: 16px 20px;
  border-top: 1px solid var(--divider);
  background: var(--bg-panel);
  border-radius: 0 0 var(--radius-xl) var(--radius-xl);
}
```

---

## 12. Accessibility (A11y)

### 12.1 Keyboard Navigation

| Key | Action |
|-----|--------|
| Tab | Navigate through interactive elements |
| Enter/Space | Activate button/link |
| Escape | Close modal |
| Arrow Up/Down | Navigate table rows (MVP) |

### 12.2 ARIA Attributes

```html
<!-- Table -->
<table role="grid" aria-label="Кейси що потребують підтвердження">
  <thead>
    <tr role="row">
      <th scope="col">Case ID</th>
      <!-- ... -->
    </tr>
  </thead>
  <tbody>
    <tr role="row" tabindex="0" aria-label="Кейс F1-SEA-2026-02451, ACME Ltd">
      <!-- ... -->
    </tr>
  </tbody>
</table>

<!-- Modal -->
<div 
  role="dialog" 
  aria-modal="true" 
  aria-labelledby="modal-title"
>
  <h3 id="modal-title">Створити новий кейс</h3>
  <!-- ... -->
</div>

<!-- SLA indicator -->
<span 
  class="sla-warning" 
  aria-label="SLA дедлайн: 2 дні 4 години"
>
  2d 4h
</span>
```

### 12.3 Focus Management

| Action | Focus Behavior |
|--------|----------------|
| Page load | Focus on first interactive element |
| Open modal | Focus on first form field |
| Close modal | Return focus to trigger button |
| Row navigation | Focus visible on current row |

---

## 13. Anti-patterns (чого НЕ робити)

| ❌ Anti-pattern | Проблема | ✅ Рішення |
|-----------------|----------|------------|
| Показувати всі кейси в одному списку | Нема пріоритезації | Групування по секціях |
| Приховувати SLA | Менеджер пропустить дедлайн | Завжди показувати SLA |
| Auto-refresh без warning | Втрата контексту | "Updated Xs ago" + manual refresh |
| Pagination для маленьких списків | Зайва складність | Infinite scroll або show all |
| Складні фільтри на PoC | Overwhelm | Sidebar quick filters |
| Modal без ESC close | Порушення UX patterns | ESC + overlay click = close |

---

## 14. Acceptance Criteria (Definition of Done)

### 14.1 PoC Acceptance

Case List v0 вважається успішним, якщо:

**Functional:**
- [ ] Менеджер бачить 4 KPI tiles з актуальними даними
- [ ] Менеджер бачить 3 секції кейсів (approval, in progress, waiting)
- [ ] Клік на рядок → перехід до Case Cockpit
- [ ] "Новий кейс" → модальне вікно
- [ ] Створення кейса → редірект на новий кейс
- [ ] Sidebar quick filters працюють

**UX:**
- [ ] Кейси з pending approvals візуально виділені
- [ ] SLA at risk показує відповідний колір
- [ ] Blocked кейси "приглушені" (opacity)
- [ ] Responsive layout (desktop-first)

**Performance:**
- [ ] Page load < 1s
- [ ] Table renders < 100 rows without lag

### 14.2 MVP Acceptance (додатково)

- [ ] Advanced filter modal
- [ ] Search by case_number/client
- [ ] Column customization
- [ ] Keyboard navigation (arrows)

---

## 15. Roadmap: v0 → v1 → v2

### v0 (PoC)

| Компонент | Включено |
|-----------|----------|
| Stats tiles (4) | ✅ |
| 3 grouped sections | ✅ |
| Clickable rows | ✅ |
| New Case modal | ✅ |
| Sidebar quick filters | ✅ |
| Search | — |
| Advanced filters | — |

### v1 (MVP)

| Компонент | Включено |
|-----------|----------|
| v0 features | ✅ |
| Search | ✅ |
| Advanced filter modal | ✅ |
| Column customization | ✅ |
| Saved views integration | ✅ |
| Bulk select | — |

### v2 (Scale)

| Компонент | Включено |
|-----------|----------|
| v1 features | ✅ |
| Kanban view | ✅ |
| Calendar view | ✅ |
| Export CSV/Excel | ✅ |
| Bulk actions | ✅ |
| Keyboard shortcuts | ✅ |

---

## 16. Technical Implementation Notes

### 16.1 Frontend Implementation

| Аспект | Рекомендація |
|--------|--------------|
| **Framework** | React + Supabase JS client |
| **State** | React Query for server state |
| **Routing** | Next.js or React Router |
| **Forms** | React Hook Form for New Case modal |
| **Validation** | Zod schemas |

### 16.2 Real-time Updates

```typescript
// Supabase subscription for real-time updates
supabase
  .channel('cases_changes')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'cases',
      filter: `owner_user_id=eq.${userId}`
    },
    (payload) => {
      // Refetch or update local state
      queryClient.invalidateQueries(['cases']);
    }
  )
  .subscribe();
```

### 16.3 Performance Optimizations

| Optimization | Implementation |
|--------------|----------------|
| **Pagination** | Cursor-based for large lists (MVP) |
| **Caching** | React Query with staleTime |
| **Prefetch** | Hover on row → prefetch case detail |
| **Virtualization** | react-virtual for 100+ rows |

---

## 17. Integration Points

### 17.1 Integration with Case Cockpit

| Взаємодія | Напрямок | Опис |
|-----------|----------|------|
| Row click | Case List → Cockpit | Navigate to `/cases/{id}` |
| New case | Case List → Cockpit | After create → redirect to new case |
| Back button | Cockpit → Case List | Browser back or explicit link |

### 17.2 Integration with Personal Settings

| Setting | Impact on Case List |
|---------|-------------------|
| `inbox_preferences.my_cases_only` | Pre-apply owner filter |
| `inbox_preferences.show_at_risk_first` | Sort at-risk cases first |
| `inbox_preferences.visible_columns` | Show/hide columns |

### 17.3 Integration with Owner Dashboard

| Metric | Shared Data |
|--------|-------------|
| Active cases count | Same query |
| At risk count | Same query |
| Pending approvals | Same query |

---

## 18. End Note

Case List в IMCP — це **центральний хаб менеджера**.

> Сторінка відповідає на головне питання: **"Що мені робити зараз?"**

Ключові принципи:
- Пріоритезація через групування (approvals → active → blocked)
- Швидкий доступ до кейсів (1 клік)
- Візуальні індикатори (SLA, status badges)
- Зрозумілі дії (NBA column, action buttons)

---

**Пов'язана документація:**
- [/docs/core/](../core/) — core документація IMCP
- [/docs/case_templates/](../case_templates/) — шаблони кейсів
- [ui_style_reference.md](./ui_style_reference.md) — дизайн-токени
- [owner_dashboard_spec.md](./owner_dashboard_spec.md) — Owner Dashboard
- [personal_settings_spec.md](./personal_settings_spec.md) — персональні налаштування
