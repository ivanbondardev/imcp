# IMCP UI/UX Guide  
## Case Cockpit — Human Approval UX Patterns

**Версія:** 1.0  
**Статус:** Core (IMCP UI/UX)  
**Аудиторія:** Product, UX/UI, Frontend, Automation (n8n), Architecture  

**Попередні документи:**  
- [00_shared_mental_model.md](./00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](./01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](./02_core_data_model.md) — модель даних  
- [03_approval_pattern.md](./03_approval_pattern.md) — патерн підтверджень  

**Мета:** зафіксувати канонічний UX підхід для IMCP, який масштабується на різні кейси та "кабіни".

---

## 1. UX-філософія IMCP

### 1.1 Основний принцип

> **Менеджер не "користується системою" — він у ній працює.**

IMCP — це робоче середовище для керування складними кейсами, а не просто CRM/таблиця/чат.

### 1.2 Iron Man UX (людина + ШІ)

| Роль | Функція |
|------|---------|
| **ШІ = екзоскелет** | Швидкість, варіанти, рутина, контроль повноти |
| **Людина = пілот** | Верифікація, контекст, відповідальність, фінальне рішення |

### 1.3 UX-цілі IMCP

- прискорювати роботу менеджера
- знижувати помилки через втому
- зберігати контроль у критичних точках
- робити процес прозорим і відтворюваним

### 1.4 Що UI НЕ має робити

| ❌ Заборонено | Чому |
|---------------|------|
| Чат як основний інтерфейс | Складно відстежувати стан кейса |
| Ховати ризики та невизначеність | Менеджер має бачити повну картину |
| Довгі AI-есе | Когнітивне перевантаження |
| Незворотні дії без підтвердження | Порушує human-in-the-loop |

### 1.5 Що UI має робити завжди

| ✅ Обов'язково | Реалізація |
|----------------|------------|
| Показувати **де ми зараз** | State + Status badge |
| Показувати **що робити далі** | Next Best Action (NBA) |
| Показувати **що зробила система** | Event log / Timeline |
| Давати **контроль людині** | Approval Gates |

---

## 2. Ключові UX-патерни IMCP

### 2.1 Case Cockpit (кабіна пілота)

**Один кейс = один головний екран**, де зібрано:
- стан/контекст
- дії
- approvals
- документи
- таймлайн

> Менеджер працює **всередині кейса**, а не "між вкладками".

### 2.2 Next Best Action (NBA)

UI завжди показує 1–3 наступні дії, які рухають кейс.

| Мета | Результат |
|------|-----------|
| Прибрати "а що робити далі?" | Фокус на критичному шляху |
| Мінімізувати когнітивне навантаження | Менше помилок |
| Фокусувати на critical path | Швидше закриття кейсів |

### 2.3 Draft-First Interaction

Система **не виконує фінал**, вона готує чернетку:
- листа, калькуляції, документа, повідомлення

Менеджер:
- **підтверджує** → **редагує** → **відхиляє**

### 2.4 Human Approval Gates як UX-об'єкт

Approval — не "ви впевнені?" модалка.  
Approval — це **керований мікро-процес**:

1. Показати "що пропонується"
2. Показати "ризики"
3. Дозволити "прийняти/відредагувати/відхилити"
4. Зафіксувати "хто і що вирішив"

### 2.5 Progressive Disclosure

Показуємо **мінімум необхідного** одразу, решту — по кліку:

```
Summary → Деталі → Глибокий контекст
```

AI reasoning (якщо показувати) — **тільки по кліку**.

### 2.6 Generation–Verification Loop

Патерн для документів та витягування даних:

1. Система показує **результат**
2. Система підсвічує **джерело**
3. Людина швидко **верифікує**
4. Рішення фіксується як **audit**

---

## 3. Структура Case Cockpit

### 3.1 Базовий layout

```
┌─────────────────────────────────────────────────┐
│ CASE HEADER (ID, status, SLA, owner, priority) │
├─────────────────────────────────────────────────┤
│ STICKY NEXT BEST ACTION                         │
├───────────────────────────┬─────────────────────┤
│ MAIN WORK AREA (8/12)     │ CONTEXT SIDEBAR     │
│                           │ (4/12)              │
│ - Draft view              │ - Cargo summary     │
│ - Form view               │ - Route info        │
│ - Document review         │ - Risk flags        │
│ - Approval detail         │ - Quick links       │
├───────────────────────────┴─────────────────────┤
│ TIMELINE / EVENT LOG                            │
└─────────────────────────────────────────────────┘
```

### 3.2 Чому саме так

- Менеджер бачить **контекст + дію** одночасно
- Approvals не губляться
- Мінімум перемикань між сторінками
- Легко масштабувати на різні case types

---

## 4. Ключові екрани IMCP

### 4.1 Case List / Inbox (черга роботи)

Не "таблиця всього", а **операційна черга**.

| Секція | Призначення |
|--------|-------------|
| Requires my approval | Потребує рішення |
| At risk / SLA скоро | Дедлайни |
| Waiting for external input | Чекаємо відповідь |
| In progress | У роботі |
| Done / Archived | Завершені |

**Мінімальні колонки:** Case ID + тип, статус, NBA, SLA, ризик-індикатор, owner

### 4.2 Case Cockpit (основний екран)

Місце, де відбувається робота. Детальна структура — див. секцію 3.

### 4.3 Approval Screen / Approval Drawer

Approval як окрема зона (екран або drawer справа).

**Містить:**
- `request_snapshot` (що пропонує система)
- diff (якщо є)
- ризики/flags
- CTA: **Approve** | **Edit & Approve** | **Reject**
- поле для коментаря

### 4.4 Document Review

Особливо важливо для: коносаментів, інвойсів, контрактів, довіреностей.

**Потрібні можливості:**
- preview документа
- підсвічування витягнутих полів
- "звідки взято" (source reference)
- швидке виправлення

### 4.5 Activity / Timeline

Audit view, який формує довіру та контроль:

| Елемент | Призначення |
|---------|-------------|
| Timestamp | Коли відбулось |
| Actor | Human / System / AI / Integration |
| Опис | Що саме сталося |
| Details | Expandable контекст |

---

## 5. Як показувати AI (HITL 2.0: Transparency)

### 5.1 Показувати результат, а не "магію"

Менеджеру потрібне:
- що система пропонує
- наскільки впевнена
- що може бути ризиком
- що робити далі

### 5.2 AI має бути "невидимим двигуном"

| ✅ Рекомендовано | ❌ Не рекомендовано |
|------------------|---------------------|
| "System prepared a draft…" | "AI thinks…" |
| "Suggested options…" | "The model believes…" |
| "Needs approval…" | "According to AI analysis…" |

### 5.3 Прозорість невпевненості (HITL 2.0)

> **Принцип:** AI не має права маскувати невпевненість "авторитетним тоном".

#### Обов'язковий показ:
- **Confidence per field** — не тільки загальний, а для кожного критичного поля
- **Uncertainty areas** — де саме AI не впевнений
- **Суперечності** — коли дані з різних джерел конфліктують
- **Assumptions** — припущення, які зробив AI

#### UI елементи для невпевненості:

```
┌─────────────────────────────────────────────────────────┐
│ 📦 Cargo Weight                                         │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │ 2,500 kg                           🟡 Medium    │   │
│   │                                                 │   │
│   │ ⚠️ Extracted from invoice, not confirmed by     │   │
│   │    warehouse. Client email mentioned "~2.5t"    │   │
│   │                                                 │   │
│   │ 💡 Suggested verification: Request warehouse    │   │
│   │    weighing before quote confirmation           │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│ [ Confirm as-is ]  [ Edit value ]  [ Request verify ]   │
└─────────────────────────────────────────────────────────┘
```

#### Confidence Indicators:

| Індикатор | Значення | UI |
|-----------|----------|----|
| 🟢 High | 0.85+ | Зелений badge |
| 🟡 Medium | 0.6-0.85 | Жовтий badge + hover tooltip |
| 🔴 Low | < 0.6 | Червоний badge + inline warning |

### 5.4 AI reasoning (HITL 2.0: Explainability)

Reasoning **обов'язково доступний**, але показується **по кліку**:

#### Структура reasoning panel:

```
┌─────────────────────────────────────────────────────────┐
│ 💭 How this was calculated                      [Close] │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Factors considered:                                     │
│ ├── 📄 Invoice INV-2026-0451 (HIGH weight)             │
│ ├── 📧 Client email from Jan 15 (MEDIUM weight)        │
│ └── 📊 Similar shipments avg (LOW weight)              │
│                                                         │
│ Assumptions made:                                       │
│ • Packaging weight included in declared weight          │
│ • Standard pallet dimensions assumed                    │
│                                                         │
│ Would reconsider if:                                    │
│ • Warehouse provides actual measurements                │
│ • Client confirms exact packing list                    │
│                                                         │
│ Alternatives rejected:                                  │
│ • Using historical average (less accurate for this      │
│   cargo type)                                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.5 Conflict Detection

Коли AI виявляє суперечності між джерелами:

```
┌─────────────────────────────────────────────────────────┐
│ ⚠️ DATA CONFLICT DETECTED                               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Field: cargo.total_cbm                                  │
│                                                         │
│ ┌──────────────────┐    ┌──────────────────┐           │
│ │ 📄 Invoice       │ vs │ 📧 Client email  │           │
│ │    3.2 cbm       │    │    4.1 cbm       │           │
│ └──────────────────┘    └──────────────────┘           │
│                                                         │
│ Delta: 28% difference                                   │
│                                                         │
│ [ Use Invoice value ]  [ Use Email value ]  [ Enter    │
│                                               manually ] │
└─────────────────────────────────────────────────────────┘
```

---

## 5.6 Verify Mode (HITL 2.0: Cognitive Bias Mitigation)

Для протидії **automation bias** та **complacency**, UI підтримує різні режими верифікації:

### Типи Verify Mode:

| Режим | Опис | Коли застосовується |
|-------|------|---------------------|
| `STANDARD` | Звичайний approval flow | За замовчуванням |
| `DEEP` | Чекліст обов'язкових перевірок | HIGH-risk approvals |
| `SPOT_CHECK` | Випадкова поглиблена перевірка | Періодично (1 з N) |

### DEEP Verify Mode UI:

```
┌─────────────────────────────────────────────────────────┐
│ 🔍 DEEP VERIFICATION REQUIRED                           │
│    This approval requires additional checks              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Verification Checklist:                                 │
│                                                         │
│ ☐ Cargo dimensions match invoice                        │
│   └── Source: Invoice p.2, line 15                      │
│                                                         │
│ ☐ Client name matches contract                          │
│   └── Source: Contract header                           │
│                                                         │
│ ☐ Incoterms verified with client                        │
│   └── Source: Email from Jan 14                         │
│                                                         │
│ ☐ Price calculation reviewed                            │
│   └── Click to expand calculation breakdown             │
│                                                         │
│ ────────────────────────────────────────────────────────│
│ ⚠️ All items must be checked before approval            │
│                                                         │
│ [ Cancel ]                              [ Approve (0/4) ]│
└─────────────────────────────────────────────────────────┘
```

### SPOT_CHECK Mode:

- Випадково активується для ~10% звичайних approvals
- Менеджер **не знає заздалегідь**, чи буде spot check
- Підтримує пильність через непередбачуваність
- Результати spot checks аналізуються для quality metrics

### Парадигма "Спростування" в UI:

Замість питання "Чи все правильно?" → "Де може бути помилка?"

```
┌─────────────────────────────────────────────────────────┐
│ 🔎 VERIFICATION FOCUS                                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ AI highlighted these areas for attention:               │
│                                                         │
│ 🎯 cargo.weight — extracted from low-quality scan       │
│ 🎯 quote.margin — lower than average for this route     │
│ 🎯 broker.export_declaration — marked as "unknown"      │
│                                                         │
│ Can you confirm these values are correct?               │
│                                                         │
│ [ All correct ]  [ I found issues ]                     │
└─────────────────────────────────────────────────────────┘
```

---

## 6. Anti-patterns (чого НЕ робити)

| ❌ Anti-pattern | Проблема |
|-----------------|----------|
| Чат як основний UX | Втрачається структура |
| Автоматичне виконання без approval | Немає контролю |
| "Розумні кнопки" без пояснення | Непередбачуваність |
| Приховані ризики/конфлікти | Помилки пізніше |
| Розмиті стани ("в процесі") | Незрозуміло що далі |
| Багато вільного тексту | Важко верифікувати |
| Відсутність історії | Немає audit trail |
| **Приховування невпевненості AI** | Automation bias, помилки |
| **Однаковий approval UX для всіх рівнів ризику** | Approval fatigue |
| **Відсутність reasoning** | "Чорна скринька", недовіра |

---

## 6.1 Learning Loop UX (HITL 2.0)

### Збір Correction Signal в UI

Коли менеджер редагує draft перед approval:

```
┌─────────────────────────────────────────────────────────┐
│ 📝 You made changes to the draft                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Changes detected:                                       │
│ • cargo.total_cbm: 3.2 → 3.8 cbm                       │
│                                                         │
│ Help us improve (optional):                             │
│                                                         │
│ Why did you make this change?                           │
│ ○ Client provided updated information                   │
│ ○ Found error in source document                        │
│ ○ AI extracted incorrectly                              │
│ ○ Policy/business rule override                         │
│ ○ Other: [________________]                             │
│                                                         │
│ [ Skip ]                              [ Submit & Approve ]│
└─────────────────────────────────────────────────────────┘
```

### Quality Feedback Badge

На approval history показуємо якість AI drafts:

```
┌─────────────────────────────────────────────────────────┐
│ 📊 AI Quality for this case type                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Quote Approval:  🟢 94% accuracy (last 30 days)         │
│ BL Verification: 🟡 78% accuracy (dims field weak)      │
│ Dims Extraction: 🔴 65% accuracy (improving...)         │
│                                                         │
│ Your corrections help improve these numbers!            │
└─────────────────────────────────────────────────────────┘
```

### Time Tracking (для метрик)

UI автоматично трекає:
- `time_to_first_action` — скільки часу менеджер дивився перед дією
- `time_spent_editing` — скільки часу редагував
- `verification_depth` — скільки полів розгорнув / перевірив

Ці метрики допомагають:
- Виявляти "rubber stamping" (занадто швидкі approvals)
- Оптимізувати UX для частих операцій
- Калібрувати spot check frequency

---

## 7. UX-чекліст для кожного екрану

Для кожного екрану і кожного стану кейса:

| # | Питання | Що перевіряємо |
|---|---------|----------------|
| 1 | Чи зрозуміло, де я зараз? | State + Status |
| 2 | Чи зрозуміло, що робити далі? | NBA |
| 3 | Чи видно, що зробила система? | Events / Draft |
| 4 | Чи видно, що зробив я? | Approvals / Decisions |
| 5 | Чи можна швидко підтвердити/виправити? | CTA доступність |
| 6 | Чи є "stop/rollback" для ризикових місць? | Cancel / Undo |
| 7 | Чи мінімум кліків до критичної дії? | UX flow |
| 8 | Чи не змушує UI "думати з нуля"? | Draft-first |
| 9 | Чи видно рівень впевненості AI? | Confidence indicators |
| 10 | Чи є доступ до reasoning AI? | Explainability |
| 11 | Чи підсвічуються конфлікти даних? | Conflict detection |
| 12 | Чи збирається feedback при редагуванні? | Learning Loop |

---

## 8. ASCII Wireframe — IMCP Case Cockpit

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Case #F1-SEA-02451        STATUS: WAITING_FOR_QUOTE        SLA: 2d 4h │
│ Client: ACME LTD     Route: CN → UA (SEA)        Owner: Ivan      Priority: 🔴│
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ 🔴 NEXT BEST ACTION                                                          │
│ ─────────────────────────────────────────────────────────────────────────────│
│ Quote needs approval                                                         │
│ • System prepared price calculation                                          │
│ • Price validity: 7–9 days                                                   │
│ • Risk: client-declared dimensions not yet verified                          │
│                                                                              │
│ [ Approve Quote ]   [ Edit Quote ]   [ Reject ]                              │
└──────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────┬──────────────────────────────────────────────┐
│ MAIN WORK AREA                │ CONTEXT SIDEBAR                              │
│                               │                                              │
│ ┌───────────────────────────┐ │ ┌──────────────────────────────────────────┐ │
│ │ 💰 Quote Draft            │ │ │ 📦 Cargo Summary                         │ │
│ │                           │ │ │ • Packages: 12                           │ │
│ │ Base cost:      $2,450    │ │ │ • Declared dims: 3.2 cbm                 │ │
│ │ Margin:           $350    │ │ │ • Stackable: Yes                         │ │
│ │ Total:          $2,800    │ │ │ • Dangerous: No                          │ │
│ │                           │ │ └──────────────────────────────────────────┘ │
│ │ Notes:                    │ │                                              │
│ │ "Price valid 7–9 days"    │ │ ┌──────────────────────────────────────────┐ │
│ └───────────────────────────┘ │ │ 🚚 Route & Logistics                     │ │
│                               │ │ • Origin: Yiwu Warehouse                 │ │
│                               │ │ • Port: Shenzhen                         │ │
│                               │ │ • Destination: Kyiv                      │ │
│                               │ │ • Broker: Client                         │ │
│                               │ └──────────────────────────────────────────┘ │
│                               │                                              │
│                               │ ┌──────────────────────────────────────────┐ │
│                               │ │ ⚠️ Risk Flags                            │ │
│                               │ │ • Dimensions not verified                │ │
│                               │ │ • Export declaration unclear             │ │
│                               │ └──────────────────────────────────────────┘ │
└───────────────────────────────┴──────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│ 🕓 TIMELINE / EVENT LOG                                                      │
│ ─────────────────────────────────────────────────────────────────────────────│
│ 12:10  SYSTEM   Quote draft generated                                        │
│ 11:42  HUMAN    Client info confirmed                                        │
│ 10:30  SYSTEM   Cargo data extracted from email                              │
│ 09:55  HUMAN    Case created                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. UI-компоненти (Figma/React)

### 9.1 CaseHeader

| Елемент | Призначення |
|---------|-------------|
| Case ID | Ідентифікація (з copy) |
| Case type | Badge з типом |
| Status | Badge зі станом |
| SLA / deadline | Час до дедлайну |
| Owner | Avatar менеджера |
| Priority | Індикатор пріоритету |

### 9.2 NextBestAction (Sticky)

| Елемент | Призначення |
|---------|-------------|
| Заголовок | Коротке формулювання (1 рядок) |
| Контекст | 2–3 bullet points |
| Ризик | Warning якщо є |
| Primary CTA | Approve / Continue / Request Info |
| Secondary CTA | Edit / Postpone / View Details |

**Варіанти станів NBA:**
- 🔴 Approval required
- 🟡 Waiting for input
- 🔵 Automated step running
- ⚪ Blocked
- 🟢 Completed

### 9.3 Main Work Area

| Режим | Коли використовується |
|-------|----------------------|
| Draft view | Quote draft, message draft |
| Form view | Data input |
| Document review | BL, invoice |
| Approval detail | Diff, snapshot |

### 9.4 Context Sidebar

- Summary cards (cargo, route, broker, dates)
- Risk flags
- Quick links (docs, integrations)

**Правила:** без primary actions, завжди релевантно поточному стану, компактно.

### 9.5 Timeline / Event Log

| Елемент | Формат |
|---------|--------|
| Timestamp | HH:MM або дата |
| Actor | HUMAN / SYSTEM / AI / INTEGRATION |
| Опис | Короткий текст |
| Details | Expandable |

---

## 10. Масштабування на нові кейси

### 10.1 Case Type → UI Composition

Для кожного `case_type` визначається:
- набір states
- набір required fields
- approvals map
- набір "main work area views"

> UI залишається тим самим, змінюється лише "вміст" Main Work Area.

### 10.2 Plug-in Panels

Модулі для Sidebar:
- Financials
- Compliance
- Insurance
- Carrier tracking
- Broker info

### 10.3 New Approvals без redesign

Новий approval тип — це:
- новий шаблон `request_snapshot`
- новий view у Main Work Area
- ті самі CTA і механіка

### 10.4 Multi-domain readiness

Той самий cockpit працює для:
- логістики (shipment case)
- legal (contract case)
- фінансів (payment case)

> Бо це **case-driven + approvals + events**, а не доменні кнопки.

---

## End Note

Цей документ задає **канонічний UX IMCP**, який:
- відповідає [shared mental model](./00_shared_mental_model.md)
- зручний для менеджера
- безпечний для бізнесу
- технічно простий для PoC
- масштабований на інші "кабіни/кейси"

---

**Пов'язана документація:**  
- [/docs/case_templates/](../case_templates/) — приклади конкретних кейсів та їх UI-специфіки
