# Platform UI Contracts

Глобальні UI патерни для всіх екранів IMCP.

---

## 1. App Shell

### Структура
- **Top Bar**: пошук, сповіщення, user menu, org context, health indicator
- **Sidebar**: навігація + badges з counts
- **Main Content**: поточний екран

### Глобальні вимоги
- Стабільна навігація між ключовими зонами
- Realtime індикатор стану підключення
- Єдиний спосіб показувати помилки/завантаження/empty states

---

## 2. Navigation

### Основні розділи
- Cases — операційна черга
- Case Cockpit — детальний екран кейса
- Approvals — черга рішень
- Documents — документи з AI верифікацією
- Timeline — audit log
- Settings — персональні налаштування

### Deep linking
Усі ключові артефакти повинні мати пряме посилання:
- `case_number` → Case Cockpit
- `approval_id` → Approval detail
- `document_id` → Document viewer
- `case_event_id` → Timeline з highlight

---

## 3. Auth & Access (RLS-aware)

### Принцип
UI не "вгадує" доступ — RLS у Supabase є джерелом істини.

### Стани
- **Signed-out**: Login screen
- **Session expired**: inline modal з можливістю re-login
- **403 / RLS denied**: пояснення + CTA повернутись
- **404**: Not found без витоку інформації

---

## 4. Global Errors / Loading / Empty States

### Error boundary
Кожен екран має обробляти помилки:
- Короткий меседж
- correlation_id якщо є
- CTA: Retry / Back

### Loading
- Skeleton loaders для списків/карток
- Background refresh з "Last updated" індикатором

### Realtime health
- Зелений: realtime OK
- Жовтий: degraded (fallback на polling)
- Червоний: offline

### Empty states
- Пояснюють "чому пусто"
- Пропонують next action

---

## 5. Global Search

### Scope
Пошук повинен знаходити:
- Cases по case_number, client name, title
- Approvals по case_number, type
- Documents по filename, doc_type

### UX
- Keyboard-first: `/` або `Cmd+K` для фокусу
- Quick results + "View all"
- Поважає RLS

---

## 6. Notifications

### Призначення
Канал для подій, що потребують уваги, але не замінює Approvals чи Timeline.

### Категорії
- **Critical**: approvals, SLA at risk, integration degraded
- **Updates**: document processed, case state changed

### Anti-noise правила
- Toasts для коротких підтверджень
- Агрегація при багатьох подіях за короткий час
- Quiet hours з preferences
