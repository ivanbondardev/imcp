# Timeline / Event Log Specification
## IMCP — Append-only Audit Log усіх подій платформи

**Версія:** 1.0  
**Статус:** Spec (PoC → MVP)  
**Тип документа:** Spec  
**Аудиторія:** Product, UX/UI, Frontend, Architecture, Compliance  
**Changelog:**  
- v1.0 — initial spec based on demo/timeline.html

**Попередні документи:**  
- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](../core/01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](../core/02_core_data_model.md) — модель даних  
- [03_approval_pattern.md](../core/03_approval_pattern.md) — патерн підтверджень  

**Пов'язані документи:**  
- [UI Style Reference](./ui_style_reference.md) — дизайн-токени  
- [Case List Spec](./case_list_spec.md) — специфікація Case List  
- [Owner Dashboard Spec](./owner_dashboard_spec.md) — специфікація Owner Dashboard  
- [Personal Settings Spec](./personal_settings_spec.md) — персональні налаштування  

---

## 1. Purpose — Навіщо потрібна сторінка Timeline

Timeline в IMCP — це **append-only audit log**, централізований журнал усіх подій платформи для забезпечення повної прозорості та accountability.

### 1.1 Основні задачі Timeline

| Задача | Опис |
|--------|------|
| **Аудит** | Повна історія дій: хто, коли, що, на основі чого |
| **Розслідування** | Швидкий пошук подій при інцидентах |
| **Моніторинг** | Real-time потік подій системи |
| **Compliance** | Незмінний audit trail для регуляторних вимог |
| **Debugging** | Відстеження integration та AI events |

### 1.2 Timeline ≠ Case History

| Аспект | Timeline | Case History (в Cockpit) |
|--------|----------|--------------------------|
| **Scope** | Усі кейси платформи | Один конкретний кейс |
| **Фокус** | Потік подій системи | Історія рішень у кейсі |
| **Аудиторія** | Ops Lead, Admin, Compliance | Менеджер кейса |
| **Ключове питання** | "Що відбувається в системі?" | "Як розвивався цей кейс?" |
| **Фільтри** | Actor type, event type, case | — (всі події кейса) |

> **Timeline** — це "централізований журнал подій", а не "історія кейса".

### 1.3 Відповідність Shared Mental Model

Timeline реалізує принципи [Shared Mental Model](../core/00_shared_mental_model.md):

| Принцип | Реалізація в Timeline |
|---------|----------------------|
| **Accountability** | Кожна подія фіксує actor, timestamp, context |
| **Transparency** | Append-only, видалення заборонено |
| **Human-in-the-Loop** | Чітке розділення HUMAN / SYSTEM / AI |
| **Audit Trail** | Повна історія для compliance |

---

## 2. Scope (PoC vs MVP)

### 2.1 PoC (Minimal Viable)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| **Stats Overview** | HIGH | 4 KPI tiles (events today, by actor type) |
| **Event Timeline** | HIGH | Хронологічний список подій |
| **Event Grouping** | HIGH | Групування по датах (Сьогодні, Вчора) |
| **Actor Type Markers** | HIGH | Візуальне розділення HUMAN/SYSTEM/AI/INTEGRATION |
| **Case Links** | HIGH | Клікабельні посилання на кейси |
| **Basic Filters** | MEDIUM | Sidebar quick filters по actor type |

### 2.2 MVP (Extended)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| PoC features | — | Все з PoC |
| **Advanced Filters** | HIGH | Filter modal з multiple criteria |
| **Search** | MEDIUM | Пошук по case_number, actor, event_type |
| **Export** | MEDIUM | CSV/JSON export для audit |
| **Date Range** | MEDIUM | Фільтр по періоду |
| **Event Details** | LOW | Expandable event metadata |

### 2.3 v2 (Scale)

| Функціональність | Опис |
|------------------|------|
| Real-time streaming | WebSocket-based live updates |
| Event aggregation | Згортання схожих подій |
| Advanced search | Full-text search |
| Retention policies | Archival після X днів |
| Export scheduling | Автоматичний export для compliance |

---

## 3. Information Architecture

### 3.1 Загальна структура сторінки

```
┌──────────────────────────────────────────────────────────────┐
│ Sidebar Navigation                                            │
├──────────────────────────────────────────────────────────────┤
│ Header: "Timeline / Event Log" + Actions (Фільтри, Експорт)   │
├──────────────────────────────────────────────────────────────┤
│ Info Banner: Audit Trail & Accountability                     │
├──────────────────────────────────────────────────────────────┤
│ Stats Grid (4 KPI tiles)                                      │
├──────────────────────────────────────────────────────────────┤
│ Card: "Усі події (останні)"                                   │
│ └── Timeline with date separators                             │
├──────────────────────────────────────────────────────────────┤
│ Card: "Event Taxonomy (IMCP)"                                 │
│ └── Reference of event types                                  │
├──────────────────────────────────────────────────────────────┤
│ Card: "Actor Types"                                           │
│ └── Reference of actor types                                  │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Sidebar Navigation

| Секція | Пункти | Badge |
|--------|--------|-------|
| **Main** | Owner Dashboard, Кейси, Підтвердження, Документи, Таймлайн | Кейси: count, Підтвердження: pending count |
| **Фільтри подій** | HUMAN, SYSTEM, AI, INTEGRATION | — |
| **Налаштування** | Персональні | — |

### 3.3 Drill-down Model

```
Timeline → Event Row (Case Link) → Case Cockpit
     ↓
Stats Tile Click → Filtered Timeline
     ↓
Filter Selection → Filtered Timeline
```

---

## 4. Stats Overview (KPI Tiles)

### 4.1 KPI Tiles Specification

| KPI | Label | Опис | Icon | Color |
|-----|-------|------|------|-------|
| **Подій сьогодні** | Events today | Загальна кількість подій за сьогодні | Clock | Primary |
| **HUMAN events** | Human events | Кількість подій від менеджерів | User | Success |
| **SYSTEM events** | System events | Кількість подій від n8n/triggers | Circle | Info |
| **AI events** | AI events | Кількість подій від AI extraction/generation | Sparkle | Warning |

### 4.2 KPI Tile Behavior

| Tile | Click Action | Secondary Info |
|------|--------------|----------------|
| Подій сьогодні | → Show all today's events | vs вчора (trend) |
| HUMAN events | → Filter to HUMAN only | % від усіх |
| SYSTEM events | → Filter to SYSTEM only | % від усіх |
| AI events | → Filter to AI only | % від усіх |

### 4.3 SQL Queries для KPI

```sql
-- Подій сьогодні
SELECT COUNT(*) 
FROM case_events 
WHERE org_id = $org_id
  AND created_at >= date_trunc('day', now());

-- Подій вчора (для тренду)
SELECT COUNT(*) 
FROM case_events 
WHERE org_id = $org_id
  AND created_at >= date_trunc('day', now() - interval '1 day')
  AND created_at < date_trunc('day', now());

-- By actor type (сьогодні)
SELECT 
  actor_type,
  COUNT(*) as count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) as percentage
FROM case_events 
WHERE org_id = $org_id
  AND created_at >= date_trunc('day', now())
GROUP BY actor_type;
```

---

## 5. Event Timeline Specification

### 5.1 Event Item Structure

Кожна подія в timeline має наступну структуру:

| Елемент | Опис | Приклад |
|---------|------|---------|
| **Marker** | Іконка з кольором actor type | 🔵 SYSTEM |
| **Actor** | Тип актора + ім'я (для HUMAN) | `HUMAN (Іван П.)` |
| **Timestamp** | Час події | `12:10` |
| **Event Type** | Тип події (код) | `STATE_CHANGED` |
| **Details** | Контекст події | Case: F1-SEA-2026-02451 • `QUOTE_READY → QUOTE_APPROVAL_PENDING` |

### 5.2 Actor Types

| Actor Type | Icon | Border Color | Опис |
|------------|------|--------------|------|
| **HUMAN** | User | `var(--accent)` | Дії менеджера |
| **SYSTEM** | Circle | `var(--text-secondary)` | n8n workflows, triggers |
| **AI** | Sparkle | `var(--warning)` | Extraction, generation |
| **INTEGRATION** | Code | `var(--success)` | 1C, API callbacks |

### 5.3 Event Types Taxonomy

#### Core Events

| Event Type | Опис | Actor Types |
|------------|------|-------------|
| `CASE_CREATED` | Кейс створено | HUMAN, SYSTEM |
| `STATE_CHANGED` | Змінено бізнес-стан | HUMAN, SYSTEM |
| `STATUS_CHANGED` | Змінено технічний статус | HUMAN, SYSTEM |

#### Approval Events

| Event Type | Опис | Actor Types |
|------------|------|-------------|
| `APPROVAL_CREATED` | Створено запит на approval | SYSTEM |
| `APPROVAL_APPROVED` | Затверджено | HUMAN |
| `APPROVAL_REJECTED` | Відхилено | HUMAN |

#### Document Events

| Event Type | Опис | Actor Types |
|------------|------|-------------|
| `DOC_UPLOADED` | Завантажено документ | HUMAN, INTEGRATION |
| `DOC_EXTRACTED` | AI витягнув дані | AI |
| `DOC_VERIFIED` | Верифіковано людиною | HUMAN |

#### Integration Events

| Event Type | Опис | Actor Types |
|------------|------|-------------|
| `INTEGRATION_STARTED` | Розпочато інтеграцію | SYSTEM |
| `INTEGRATION_SUCCESS` | Успішно виконано | INTEGRATION |
| `INTEGRATION_FAILED` | Помилка інтеграції | INTEGRATION |

#### Communication Events

| Event Type | Опис | Actor Types |
|------------|------|-------------|
| `EMAIL_SENT` | Email відправлено | SYSTEM |
| `NOTIFICATION_SENT` | Notification відправлено | SYSTEM |

### 5.4 Date Separators

Timeline групується по датах з візуальними сепараторами:

| Дата | Label |
|------|-------|
| Сьогодні | `Сьогодні` |
| Вчора | `Вчора` |
| Цього тижня | День тижня (напр. `Понеділок`) |
| Раніше | Дата (напр. `12 січня 2026`) |

### 5.5 Event Details Metadata

Кожен тип події має специфічні metadata fields:

| Event Type | Metadata Fields |
|------------|-----------------|
| `STATE_CHANGED` | `from_state`, `to_state`, `reason` |
| `APPROVAL_CREATED` | `approval_type`, `status` |
| `APPROVAL_APPROVED` | `approval_type`, `has_edits` |
| `APPROVAL_REJECTED` | `approval_type`, `reason` |
| `DOC_UPLOADED` | `doc_type`, `source`, `file_name` |
| `DOC_EXTRACTED` | `confidence`, `fields_count`, `model` |
| `QUOTE_CALCULATED` | `total`, `version` |
| `INTEGRATION_SUCCESS` | `integration_type`, `action`, `external_id` |
| `EMAIL_SENT` | `to`, `template`, `subject` |

---

## 6. Sidebar Quick Filters

### 6.1 Filter Options

| Фільтр | Icon | Опис | Query |
|--------|------|------|-------|
| HUMAN | User | Показати тільки людські дії | `actor_type = 'HUMAN'` |
| SYSTEM | Circle | Показати системні події | `actor_type = 'SYSTEM'` |
| AI | Sparkle | Показати AI events | `actor_type = 'AI'` |
| INTEGRATION | Code | Показати integration events | `actor_type = 'INTEGRATION'` |

### 6.2 Filter Behavior

- Фільтри є **exclusive** (один actor type за раз) або **additive** (multiple) — MVP
- Активний фільтр підсвічується
- Клік на активний фільтр — деактивує його
- PoC: показувати всі, фільтри як toggle

---

## 7. Header Actions

### 7.1 Фільтри Button

| Елемент | Опис |
|---------|------|
| **Icon** | Filter icon |
| **Label** | "Фільтри" |
| **Action** | Open filter modal (MVP) або toggle sidebar filters (PoC) |

### 7.2 Експорт Button

| Елемент | Опис |
|---------|------|
| **Icon** | Download icon |
| **Label** | "Експорт" |
| **Action** | Export current filtered view |

#### Export Formats (MVP)

| Format | Опис | Use Case |
|--------|------|----------|
| CSV | Comma-separated | Spreadsheet analysis |
| JSON | Structured data | Programmatic processing |
| PDF | Formatted report | Compliance audit |

---

## 8. Wireframe — Timeline (PoC)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Timeline                                                  [User Avatar]│
├────────────────────┬─────────────────────────────────────────────────────────┤
│ ІМСР               │ Timeline / Event Log                                     │
│ Case Platform      │ Append-only audit log усіх подій платформи              │
│ ─────────────────  │                                    [Фільтри]  [Експорт]  │
│                    │ ───────────────────────────────────────────────────────│
│ □ Owner Dashboard  │                                                         │
│ □ Кейси        12  │ ┌─────────────────────────────────────────────────────┐ │
│ □ Підтвердження 3  │ │ 🛡️ Audit Trail & Accountability                     │ │
│ □ Документи        │ │ Кожна подія фіксується з повним контекстом:        │ │
│ ■ Таймлайн         │ │ хто, коли, що, на основі чого.                      │ │
│                    │ │ case_events — append-only, видалення заборонено.    │ │
│ ─────────────────  │ └─────────────────────────────────────────────────────┘ │
│ Фільтри подій      │                                                         │
│ ─────────────────  │ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│ 👤 HUMAN           │ │ Подій     │ │ HUMAN     │ │ SYSTEM    │ │ AI        │ │
│ ⚙️ SYSTEM          │ │ сьогодні  │ │ events    │ │ events    │ │ events    │ │
│ 🤖 AI              │ │           │ │           │ │           │ │           │ │
│ 🔗 INTEGRATION     │ │    47     │ │    18     │ │    21     │ │     8     │ │
│                    │ │ +12 vs    │ │   38%     │ │   45%     │ │   17%     │ │
│ ─────────────────  │ └───────────┘ └───────────┘ └───────────┘ └───────────┘ │
│ Налаштування       │                                                         │
│ ─────────────────  │ ─────────────────────────────────────────────────────── │
│ ⚙ Персональні      │ 🕐 Усі події (останні)                  Case: All       │
│                    │                                         Actor: All      │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ СЬОГОДНІ                                                │
│                    │                                                         │
│                    │ 🔵 SYSTEM                                        12:10  │
│                    │    APPROVAL_CREATED                                     │
│                    │    Case: F1-SEA-2026-02451 • Type: QUOTE_APPROVAL       │
│                    │    status: PENDING                                      │
│                    │                                                         │
│                    │ 🔵 SYSTEM                                        12:10  │
│                    │    STATE_CHANGED                                        │
│                    │    Case: F1-SEA-2026-02451                              │
│                    │    QUOTE_READY → QUOTE_APPROVAL_PENDING                 │
│                    │                                                         │
│                    │ 🔵 SYSTEM                                        12:08  │
│                    │    QUOTE_CALCULATED                                     │
│                    │    Case: F1-SEA-2026-02451 • total: $2,800 • v: 1       │
│                    │                                                         │
│                    │ 🟢 HUMAN (Іван П.)                               11:42  │
│                    │    STATE_CHANGED                                        │
│                    │    Case: F1-SEA-2026-02451                              │
│                    │    CLIENT_INFO_COLLECTED → QUOTE_READY                  │
│                    │                                                         │
│                    │ 🟠 AI                                            10:30  │
│                    │    DOC_EXTRACTED                                        │
│                    │    Case: F1-SEA-2026-02451 • confidence: 92%            │
│                    │    fields_count: 7 • model: gpt-4                       │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ ВЧОРА                                                   │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ 🟢 HUMAN (Марія К.)                              16:42  │
│                    │    APPROVAL_APPROVED                                    │
│                    │    Case: F1-SEA-2026-02444 • type: QUOTE_APPROVAL       │
│                    │    has_edits: false                                     │
│                    │                                                         │
│                    │ 🟢 INTEGRATION (1C)                              14:25  │
│                    │    INTEGRATION_SUCCESS                                  │
│                    │    Case: F1-SEA-2026-02443 • action: CREATE_DEAL        │
│                    │    external_id: DEAL-2026-0443                          │
│                    │                                                         │
│                    │ 🔴 HUMAN (Олена С.)                              11:05  │
│                    │    APPROVAL_REJECTED                                    │
│                    │    Case: F1-SEA-2026-02442 • type: DIMS_CHANGE_APPROVAL │
│                    │    reason: Потрібне перевимірювання                     │
│                    │                                                         │
└────────────────────┴─────────────────────────────────────────────────────────┘
```

---

## 9. UX Behavior & Interactions

### 9.1 Event Row Click Behavior

| Element | Action | Result |
|---------|--------|--------|
| Case link | Click | → Navigate to Case Cockpit |
| Event row | Hover | → Show subtle highlight |
| Event row | Click | → Expand event details (MVP) |

### 9.2 Stats Tile Click

| Tile | Action |
|------|--------|
| Подій сьогодні | Clear filters, show today's events |
| HUMAN events | Filter to actor_type = 'HUMAN' |
| SYSTEM events | Filter to actor_type = 'SYSTEM' |
| AI events | Filter to actor_type = 'AI' |

### 9.3 Loading Behavior

| Component | Loading State |
|-----------|---------------|
| Stats tiles | Skeleton loaders (4 boxes) |
| Timeline | Skeleton event items |
| Initial load | Show most recent events first |

### 9.4 Infinite Scroll / Pagination

| Approach | PoC | MVP |
|----------|-----|-----|
| Load strategy | Load last 50 events | Infinite scroll |
| "Load more" | Button at bottom | Auto-load on scroll |
| Performance | — | Virtualization for 500+ events |

### 9.5 Empty States

| State | Message |
|-------|---------|
| No events today | "Сьогодні ще немає подій" |
| No events matching filter | "Немає подій для вибраного фільтру" |
| Error loading | "Помилка завантаження подій" + Retry button |

### 9.6 Real-time Updates

| Strategy | PoC | MVP |
|----------|-----|-----|
| Update method | Polling (30s) | Real-time subscriptions |
| New event indicator | — | Toast notification + "New events" button |
| Auto-scroll | — | Optional (user preference) |

---

## 10. Data Sources (Supabase)

### 10.1 Core Table

Timeline базується на таблиці `case_events` ([02_core_data_model.md](../core/02_core_data_model.md)):

```sql
CREATE TABLE case_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id UUID NOT NULL REFERENCES cases(id),
  event_type TEXT NOT NULL,           -- e.g., 'STATE_CHANGED', 'APPROVAL_APPROVED'
  actor_type TEXT NOT NULL,           -- 'HUMAN', 'SYSTEM', 'AI', 'INTEGRATION'
  actor_id UUID,                      -- NULL for SYSTEM
  org_id UUID NOT NULL,
  metadata JSONB DEFAULT '{}',        -- Event-specific data
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Index for efficient querying
CREATE INDEX idx_case_events_org_created ON case_events(org_id, created_at DESC);
CREATE INDEX idx_case_events_actor_type ON case_events(actor_type);
CREATE INDEX idx_case_events_event_type ON case_events(event_type);
```

### 10.2 Views (рекомендовано)

```sql
-- View for timeline with actor names
CREATE VIEW v_timeline_events AS
SELECT 
  e.id,
  e.case_id,
  c.case_number,
  e.event_type,
  e.actor_type,
  e.actor_id,
  COALESCE(u.raw_user_meta_data->>'full_name', e.actor_type) as actor_name,
  e.metadata,
  e.created_at,
  e.org_id,
  -- Date grouping helper
  CASE 
    WHEN e.created_at >= date_trunc('day', now()) THEN 'today'
    WHEN e.created_at >= date_trunc('day', now() - interval '1 day') THEN 'yesterday'
    WHEN e.created_at >= date_trunc('week', now()) THEN 'this_week'
    ELSE 'earlier'
  END as date_group
FROM case_events e
LEFT JOIN cases c ON c.id = e.case_id
LEFT JOIN auth.users u ON u.id = e.actor_id
ORDER BY e.created_at DESC;
```

### 10.3 RLS Policies

```sql
-- Read access for org members
CREATE POLICY "org_members_read_events" ON case_events
  FOR SELECT
  USING (org_id = (auth.jwt() ->> 'org_id')::uuid);

-- Insert only (no update/delete for audit integrity)
CREATE POLICY "system_insert_events" ON case_events
  FOR INSERT
  WITH CHECK (true);  -- Controlled by service role

-- No UPDATE or DELETE policies — append-only
```

---

## 11. UI Components

### 11.1 Info Banner

```css
.info-banner {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  padding: 16px;
  background: var(--success-bg);         /* Light green */
  border-radius: var(--radius-md);
  margin-bottom: 24px;
}

.info-banner-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: var(--radius-sm);
  background: var(--success);
  color: white;
}

.info-banner h4 {
  color: var(--success);
  margin-bottom: 4px;
}

.info-banner code {
  background: rgba(0, 0, 0, 0.1);
  padding: 2px 6px;
  border-radius: var(--radius-xs);
  font-size: 12px;
}
```

### 11.2 Timeline Container

```css
.timeline {
  position: relative;
  padding-left: 24px;
}

.timeline::before {
  content: '';
  position: absolute;
  left: 7px;
  top: 0;
  bottom: 0;
  width: 2px;
  background: var(--divider);
}
```

### 11.3 Timeline Item

```css
.timeline-item {
  display: flex;
  gap: 12px;
  padding: 12px 0;
  position: relative;
}

.timeline-marker {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 2px solid var(--text-secondary);
  background: var(--bg-editor);
  position: relative;
  z-index: 1;
  flex-shrink: 0;
}

.timeline-marker svg {
  width: 12px;
  height: 12px;
}

/* Actor type colors */
.timeline-marker.human {
  border-color: var(--accent);
}

.timeline-marker.system {
  border-color: var(--text-secondary);
}

.timeline-marker.ai {
  border-color: var(--warning);
}

.timeline-marker.integration {
  border-color: var(--success);
}
```

### 11.4 Timeline Content

```css
.timeline-content {
  flex: 1;
  min-width: 0;
}

.timeline-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}

.timeline-actor {
  font-size: 12px;
  font-weight: 600;
  color: var(--text);
}

.timeline-time {
  font-size: 12px;
  color: var(--text-secondary);
}

.timeline-event {
  font-size: 13px;
  font-weight: 500;
  color: var(--text);
  margin-bottom: 4px;
}

.timeline-details {
  font-size: 12px;
  color: var(--text-secondary);
}

.timeline-details code {
  background: var(--bg-panel);
  padding: 2px 6px;
  border-radius: var(--radius-xs);
  font-size: 11px;
}
```

### 11.5 Date Separator

```css
.date-separator {
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin: 20px 0 12px;
  padding-left: 24px;
  font-size: 11px;
  font-weight: 600;
  color: var(--text-secondary);
}
```

### 11.6 Stats Grid

```css
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}

.stat-card {
  background: var(--bg-editor);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 16px;
}

.stat-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
}

.stat-label {
  font-size: 12px;
  color: var(--text-secondary);
}

.stat-icon {
  width: 32px;
  height: 32px;
  border-radius: var(--radius-sm);
  display: flex;
  align-items: center;
  justify-content: center;
}

.stat-icon.primary { background: var(--accent-bg); color: var(--accent); }
.stat-icon.success { background: var(--success-bg); color: var(--success); }
.stat-icon.info { background: var(--info-bg); color: var(--info); }
.stat-icon.warning { background: var(--warning-bg); color: var(--warning); }

.stat-value {
  font-size: 28px;
  font-weight: 600;
  color: var(--text);
  line-height: 1;
  margin-bottom: 4px;
}

.stat-change {
  font-size: 12px;
  color: var(--text-secondary);
}

.stat-change.positive { color: var(--success); }
.stat-change.negative { color: var(--danger); }
```

---

## 12. Accessibility (A11y)

### 12.1 ARIA Attributes

```html
<!-- Timeline -->
<div role="feed" aria-label="Журнал подій">
  <article role="article" aria-label="Подія STATE_CHANGED о 12:10">
    <!-- Event content -->
  </article>
</div>

<!-- Date separator -->
<h3 role="heading" aria-level="3" class="date-separator">
  Сьогодні
</h3>

<!-- Actor type indicator -->
<span 
  class="timeline-marker system" 
  aria-label="Системна подія"
  role="img"
>
  <!-- Icon -->
</span>
```

### 12.2 Keyboard Navigation

| Key | Action |
|-----|--------|
| Tab | Navigate between interactive elements |
| Enter | Follow case link |
| Arrow Up/Down | Navigate between events (MVP) |
| Home/End | Go to first/last event (MVP) |

### 12.3 Screen Reader Support

- Announce actor type and event type
- Announce relative time ("12 хвилин тому")
- Announce case number for context

---

## 13. Anti-patterns (чого НЕ робити)

| ❌ Anti-pattern | Проблема | ✅ Рішення |
|-----------------|----------|------------|
| Показувати всі події без пагінації | Performance issues | Infinite scroll / pagination |
| Дозволяти видалення подій | Порушення audit integrity | Append-only, no delete |
| Занадто багато фільтрів | Overwhelm користувача | Quick filters в sidebar |
| Не показувати actor type | Незрозуміло хто автор | Візуальні маркери |
| Загальні назви подій | "Event happened" | Специфічні event types |
| Немає посилань на кейси | Втрата контексту | Клікабельні case links |

---

## 14. Acceptance Criteria (Definition of Done)

### 14.1 PoC Acceptance

Timeline v0 вважається успішним, якщо:

**Functional:**
- [ ] Показує хронологічний список подій
- [ ] Групує події по датах (Сьогодні, Вчора)
- [ ] Візуально розрізняє actor types (HUMAN/SYSTEM/AI/INTEGRATION)
- [ ] Case links клікабельні → перехід до Cockpit
- [ ] 4 KPI tiles з актуальними даними
- [ ] Sidebar quick filters працюють

**UX:**
- [ ] Кожна подія має чіткий actor, timestamp, event type
- [ ] Info banner пояснює призначення
- [ ] Event taxonomy для референсу
- [ ] Responsive layout (desktop-first)

**Performance:**
- [ ] Page load < 1s
- [ ] Renders 100 events without lag

### 14.2 MVP Acceptance (додатково)

- [ ] Advanced filter modal
- [ ] Date range filter
- [ ] Search by case_number/actor
- [ ] Export to CSV/JSON
- [ ] Expandable event details
- [ ] Real-time updates

---

## 15. Roadmap: v0 → v1 → v2

### v0 (PoC)

| Компонент | Включено |
|-----------|----------|
| Stats tiles (4) | ✅ |
| Chronological timeline | ✅ |
| Date separators | ✅ |
| Actor type markers | ✅ |
| Case links | ✅ |
| Sidebar quick filters | ✅ |
| Event taxonomy reference | ✅ |
| Search | — |
| Advanced filters | — |
| Export | — |

### v1 (MVP)

| Компонент | Включено |
|-----------|----------|
| v0 features | ✅ |
| Search | ✅ |
| Advanced filter modal | ✅ |
| Date range filter | ✅ |
| Export (CSV/JSON) | ✅ |
| Expandable event details | ✅ |
| Real-time updates | ✅ |
| Infinite scroll | ✅ |

### v2 (Scale)

| Компонент | Включено |
|-----------|----------|
| v1 features | ✅ |
| Event aggregation | ✅ |
| Full-text search | ✅ |
| Retention policies | ✅ |
| Scheduled exports | ✅ |
| Webhooks for events | ✅ |

---

## 16. Technical Implementation Notes

### 16.1 Frontend Implementation

| Аспект | Рекомендація |
|--------|--------------|
| **Framework** | React + Supabase JS client |
| **State** | React Query for server state |
| **Virtualization** | react-virtual for large lists |
| **Date formatting** | date-fns з українською локалізацією |

### 16.2 Real-time Subscription

```typescript
// Supabase subscription for real-time events
supabase
  .channel('timeline_events')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'case_events',
      filter: `org_id=eq.${orgId}`
    },
    (payload) => {
      // Add new event to timeline
      queryClient.setQueryData(['timeline'], (old) => ({
        ...old,
        events: [payload.new, ...old.events]
      }));
      
      // Show notification
      toast('Нова подія в системі');
    }
  )
  .subscribe();
```

### 16.3 Performance Optimizations

| Optimization | Implementation |
|--------------|----------------|
| **Pagination** | Cursor-based (created_at) |
| **Caching** | React Query with staleTime: 30s |
| **Virtualization** | react-virtual для 100+ events |
| **Index** | `(org_id, created_at DESC)` |

---

## 17. Integration Points

### 17.1 Integration with Case Cockpit

| Взаємодія | Напрямок | Опис |
|-----------|----------|------|
| Case link click | Timeline → Cockpit | Navigate to `/cases/{id}` |
| Case events tab | Cockpit → Timeline | Filtered by case_id |

### 17.2 Event Sources

| Source | Event Types |
|--------|-------------|
| **UI Actions** | STATE_CHANGED, DOC_UPLOADED (by HUMAN) |
| **n8n Workflows** | APPROVAL_CREATED, EMAIL_SENT (by SYSTEM) |
| **AI Runs** | DOC_EXTRACTED (by AI) |
| **External APIs** | INTEGRATION_SUCCESS/FAILED (by INTEGRATION) |

### 17.3 Audit Requirements

| Requirement | Implementation |
|-------------|----------------|
| **Immutability** | No UPDATE/DELETE policies on case_events |
| **Completeness** | All state changes logged |
| **Non-repudiation** | actor_id + timestamp + metadata |
| **Retention** | Configurable (default: indefinite) |

---

## 18. End Note

Timeline в IMCP — це **append-only audit log**.

> Сторінка відповідає на питання: **"Що відбувалося в системі?"**

Ключові принципи:
- **Immutability** — події не можна змінити чи видалити
- **Transparency** — повний контекст кожної події
- **Accountability** — чітке розділення actor types
- **Accessibility** — швидкий пошук і фільтрація

---

**Пов'язана документація:**
- [/docs/core/](../core/) — core документація IMCP
- [/docs/case_templates/](../case_templates/) — шаблони кейсів
- [ui_style_reference.md](./ui_style_reference.md) — дизайн-токени
- [case_list_spec.md](./case_list_spec.md) — Case List
- [owner_dashboard_spec.md](./owner_dashboard_spec.md) — Owner Dashboard
- [personal_settings_spec.md](./personal_settings_spec.md) — персональні налаштування
