# Approval Pattern  
## IMCP — Human Approval Gates (Draft-First + Accountability)

**Версія:** 1.0  
**Статус:** Core  
**Попередні документи:**  
- [00_shared_mental_model.md](./00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](./01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](./02_core_data_model.md) — модель даних (таблиця `approvals`)  

**Наступний документ:** [04_case_cockpit_ux.md](./04_case_cockpit_ux.md) — UX/UI патерни  

**Мета:** описати єдиний канонічний патерн "підтвердження людиною" для всіх кейсів IMCP.

---

## 1. Purpose — Навіщо цей документ

IMCP будується навколо принципу:

> **ШІ генерує і пропонує. Людина верифікує і приймає фінальне рішення.**

Щоб цей принцип працював не "на словах", у платформі існує базовий механізм **Human Approval Gates**.  
Цей документ визначає:

- що таке approval у IMCP і чому він є окремою сутністю
- як виглядає життєвий цикл approval
- які обов'язкові поля і які гарантії (auditability, idempotency)
- як UI, Supabase і n8n взаємодіють навколо approval
- як масштабувати approval на будь-які домени/кейси

---

## 2. Що таке Approval у IMCP

**Approval** — це контрольна точка, в якій:
- система (AI / workflow) підготувала **чернетку рішення** або пропозицію дії
- **людина** повинна:
  - підтвердити
  - відредагувати і підтвердити
  - або відхилити

Approval існує для трьох цілей:

| Ціль | Опис |
|------|------|
| **Safety & Control** | Система не робить незворотних кроків без людини |
| **Accountability** | Ми знаємо, хто і що схвалив |
| **Quality Loop** | Рішення людини стає навчальним сигналом для платформи |

> Approval — це не "статус", а **подія і артефакт рішення**.

---

## 3. Draft-First як базовий UX/процес

Кожен approval в IMCP — це реалізація принципу **draft-first**:

1. Система створює **чернетку** (draft)
2. Людина її **перевіряє**
3. Людина:
   - **approve** (як є)
   - **edit+approve** (з правками)
   - **reject** (з причиною/коментарем)

---

## 4. Коли потрібен Approval (класифікація)

### 4.1 Must-have approvals (високий ризик / незворотність)

Approval обов'язковий, якщо дія:
- створює фінансові зобов'язання (ціна, рахунок, оплата)
- має юридичний ефект (документи, довіреність, декларації)
- впливає на умови поставки/контракт
- призводить до незворотних змін у зовнішніх системах (1C, відміна рейсу)
- містить суттєві ризики (dangerous goods, VAT, customs)

### 4.2 Should-have approvals (середній ризик)

- зміна критичних полів (габарити, вага, порт, інкотермс)
- нестандартні маршрути або підрядники
- нестача даних з низьким confidence

### 4.3 No approvals (рутина / низький ризик)

- нагадування
- внутрішні нотифікації
- підготовка чернеток без відправки
- заповнення допоміжних таблиць (якщо не критично)

---

## 5. Approval як "контракт" між UI ↔ n8n ↔ Supabase

Approval завжди проходить через Supabase як джерело істини:

| Компонент | Роль |
|-----------|------|
| **UI** | Показує approvals і фіксує рішення людини (decision) |
| **Supabase** | Зберігає approval і рішення (Source of Truth) |
| **n8n** | Реагує на зміну рішення (`approvals.status` з `PENDING` → фінальний стан) і виконує наступний крок |

> n8n **не очікує** в довгому workflow.  
> Approval створюється в одному workflow, а рішення людини тригерить інший workflow.

---

## 6. Data Contract (канонічні поля approvals)

Див. [02_core_data_model.md](./02_core_data_model.md) (таблиця `approvals`). Тут — коротка інтерпретація.

### 6.1 Обов'язкові атрибути

| Поле | Призначення |
|------|-------------|
| `approval_type` | Тип підтвердження |
| `status` | `PENDING / APPROVED / REJECTED / CANCELLED` |
| `request_snapshot` | Що пропонує система |
| `decision_snapshot` | Що підтвердила людина (після рішення) |
| `requested_by`, `requested_at` | Хто і коли ініціював |
| `decided_by`, `decided_at` | Хто і коли прийняв рішення |
| `case_id` | Посилання на кейс |

### 6.2 Snapshot semantics

- `request_snapshot` **незмінний** після створення approval
- `decision_snapshot` — фінальна версія (може відрізнятись від request, якщо людина редагувала)

### 6.3 Idempotency key

- для будь-якого approval рекомендовано використовувати `idempotency_key`
- щоб уникнути дублювання при retries / повторних запусках workflow

---

## 7. Approval Lifecycle (State Machine)

### 7.1 Основні стани

```
┌─────────────────────────────────────────────┐
│                                             │
│    ┌──────────┐                             │
│    │ PENDING  │──────┬──────────┬──────────┐│
│    └──────────┘      │          │          ││
│         │            │          │          ││
│         ▼            ▼          ▼          ││
│    ┌──────────┐ ┌──────────┐ ┌───────────┐ ││
│    │ APPROVED │ │ REJECTED │ │ CANCELLED │ ││
│    └──────────┘ └──────────┘ └───────────┘ ││
│                                             │
└─────────────────────────────────────────────┘
```

| Стан | Опис |
|------|------|
| `PENDING` | Очікує рішення людини |
| `APPROVED` | Підтверджено (можливо з правками) |
| `REJECTED` | Відхилено |
| `CANCELLED` | Знято з розгляду (кейс закрито або змінено контекст) |

### 7.2 Дозволені переходи

- `PENDING → APPROVED`
- `PENDING → REJECTED`
- `PENDING → CANCELLED`

Після рішення `APPROVED/REJECTED/CANCELLED` approval **не змінюється** (immutable decision).

---

## 8. UI Pattern (Approvals Inbox)

UI повинен підтримувати (детальніше в [04_case_cockpit_ux.md](./04_case_cockpit_ux.md)):

### 8.1 Inbox View

- список approvals зі статусом `PENDING`
- сортування за: ризиком, дедлайном, типом

### 8.2 Approval Details

У деталях approval менеджер бачить:
- короткий summary (що це і чому важливо)
- `request_snapshot` (чернетка)
- контекст: ключові поля кейса + посилання на документи
- ризик/прапорці (наприклад `LOW_CONFIDENCE`)
- кнопки: **Approve** | **Edit & Approve** | **Reject**

### 8.3 UX принципи

- показувати "що зміниться після підтвердження"
- робити редагування без переходу в інші екрани (мінімізувати context switching)
- зберігати "пояснення" як частину рішення (`decision_comment`)

---

## 9. Workflow Pattern (n8n conventions around approvals)

### 9.1 Створення approval (Workflow A)

Коли система дійшла до "воріт":

1. Зібрати контекст кейса (`cases.payload`, документи, computed)
2. Сформувати `request_snapshot`
3. Створити запис в `approvals` зі статусом `PENDING`
4. Додати `case_event`: `APPROVAL_CREATED`
5. Опціонально: надіслати нотифікацію менеджеру

### 9.2 Реакція на рішення (Workflow B)

Коли `approvals.status` змінюється з `PENDING` на фінальний стан:

| Рішення | Дії |
|---------|-----|
| `APPROVED` | Виконати критичну дію → записати результат інтеграції (`INTEGRATION_STARTED/INTEGRATION_SUCCESS/INTEGRATION_FAILED`) → змінити `cases.state` (`STATE_CHANGED`) |
| `REJECTED` | Створити follow-up (опційно) → повернути кейс → змінити `cases.state` (`STATE_CHANGED`) |
| `CANCELLED` | Зафіксувати факт, нічого не виконувати |

### 9.3 Важливий принцип

> Approval "відкриває" право на дію, але не гарантує успіху інтеграції.  
> Результат дії завжди логується як окрема подія.

---

## 10. Examples (Типові approvals)

| Тип | `approval_type` | Що в `request_snapshot` |
|-----|-----------------|-------------------------|
| **Quote** | `QUOTE_APPROVAL` | Breakdown собівартості, маржа, умови, ризики |
| **BL Draft** | `BL_DRAFT_APPROVAL` | BL version, diff з контрактом, висновок |
| **1C Deal** | `ONEC_DEAL_CREATE_APPROVAL` | Payload для 1C, idempotency_key |
| **Dims Change** | `DIMS_CHANGE_APPROVAL` | Client vs warehouse dims, вплив на ціну |

---

## 11. Error Handling & Edge Cases

| Сценарій | Рішення |
|----------|---------|
| **Дублі approvals** | `idempotency_key` + UNIQUE constraint |
| **Застарілий approval** | `CANCELLED` старий → створити новий з новим snapshot |
| **Approval approved, інтеграція failed** | Логувати `ACTION_FAILED`, створити incident |
| **Конкурентні рішення** | DB constraint + optimistic lock (`UPDATE ... WHERE status=PENDING`) |

---

## 12. Security & Permissions

- Approvals можуть підтверджувати лише ролі з правами (RLS)
- Документи в approval відкриваються через signed URLs
- Усі рішення логуються в `case_events`

---

## 13. Why this pattern is core for IMCP

Approval Pattern забезпечує:

- реалізацію mental model "людина вирішує" на рівні даних і workflow
- контроль над ризиком
- відтворюваність і довіру
- базу для "learning loop" (аналіз прийнятих рішень)

> **IMCP без approvals перетворюється або на "робот-автопілот", або на "ручний трекер задач".  
> Approvals — це механізм, який робить IMCP "Iron Man Suit".**

---

## 14. HITL 2.0: Learning Loop через Approvals

Approvals у IMCP — це не тільки контроль, а й **механізм взаємного навчання** між людиною та AI.

### 14.1 Принцип Learning Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                      APPROVAL LEARNING LOOP                      │
│                                                                  │
│  ┌────────────┐         ┌──────────────┐        ┌─────────────┐ │
│  │ AI generates│ ──────► │  Approval    │ ──────► │ Human       │ │
│  │ draft       │         │  created     │         │ decides     │ │
│  └────────────┘         └──────────────┘        └──────┬──────┘ │
│                                                         │        │
│  ┌────────────────────────────────────────────────────┐ │        │
│  │                                                    │ │        │
│  │   ┌─────────────────────────────────────────────┐  │ │        │
│  │   │  Correction Signal                          │  │◄┘        │
│  │   │  • what was edited                          │  │          │
│  │   │  • why (root_cause)                         │  │          │
│  │   │  • severity                                 │  │          │
│  │   └─────────────────────────────────────────────┘  │          │
│  │                                                    │          │
│  │                    QUALITY FEEDBACK                │          │
│  └────────────────────────────────────────────────────┘          │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Pattern Analysis                                          │  │
│  │  • Which fields are corrected most?                        │  │
│  │  • Which AI runs need improvement?                         │  │
│  │  • What root causes are most common?                       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AI Improvement Actions                                    │  │
│  │  • Prompt refinement                                       │  │
│  │  • Training data augmentation                              │  │
│  │  • Rule-based overrides                                    │  │
│  │  • Confidence threshold adjustment                         │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Correction Signal — структурований feedback

Кожен approval із правками генерує `correction_signal`:

| Поле | Призначення |
|------|-------------|
| `correction_type` | APPROVE_AS_IS / MINOR_EDIT / MAJOR_EDIT / REJECT_REGENERATE |
| `edited_fields` | Список змінених полів з original → corrected |
| `root_cause` | Чому AI помилився: HALLUCINATION / OUTDATED_CONTEXT / MISSING_DATA |
| `correction_severity` | NONE / LOW / MEDIUM / HIGH |
| `should_retrain_on` | Чи є цей приклад корисним для донавчання |

### 14.3 Як UI збирає correction signal

1. Людина відкриває approval
2. Бачить `request_snapshot` (що пропонує AI)
3. Бачить `reasoning` (чому AI так вирішив) — **по кліку**
4. Редагує поля (система фіксує diff)
5. При збереженні — опціонально вказує "чому виправив"
6. Система автоматично формує `correction_signal`

### 14.4 Verification Modes (протидія cognitive biases)

| Режим | Опис | Коли застосовується |
|-------|------|---------------------|
| `STANDARD` | Звичайний approval flow | За замовчуванням |
| `DEEP` | Розширена верифікація з чеклістом | Для високоризикових approvals |
| `SPOT_CHECK` | Випадкова поглиблена перевірка | Періодично, для підтримки пильності |

### 14.5 Метрики якості (Quality Dashboard)

На основі correction signals система збирає метрики:

| Метрика | Опис | Використання |
|---------|------|--------------|
| `correction_rate` | % approvals з правками | Загальна якість AI |
| `avg_time_to_decide` | Середній час на рішення | UX оптимізація |
| `fields_most_corrected` | Топ полів, які правлять | Фокус для покращення |
| `root_causes_distribution` | Розподіл причин помилок | Пріоритезація покращень |
| `confidence_calibration` | Чи відповідає confidence реальній точності | Калібрування моделі |

### 14.6 Приклад correction_signal

```json
{
  "correction_type": "MINOR_EDIT",
  "edited_fields": [
    {
      "field_path": "cargo.total_cbm",
      "original_value": 3.2,
      "corrected_value": 3.8,
      "correction_reason": "client_clarification"
    }
  ],
  "root_cause": "MISSING_DATA",
  "fields_changed_count": 1,
  "total_fields_count": 15,
  "correction_severity": "LOW",
  "should_retrain_on": true,
  "pattern_tag": "dims_extraction_error"
}
```

---

## 15. LLM-as-Critic Pattern (Advanced)

Для зменшення навантаження на людину, IMCP може використовувати **проміжний AI-шар**:

### 15.1 Концепція

```
AI Agent         LLM Critic           Human
(generates) ────► (evaluates) ────► (if needed)
                      │
                      ▼
              ┌──────────────┐
              │ PASS: auto   │
              │ FAIL: reject │
              │ ESCALATE:    │
              │   to human   │
              └──────────────┘
```

### 15.2 Коли LLM-критик корисний

| Сценарій | Дія критика |
|----------|-------------|
| Очевидно правильний результат, низький ризик | PASS → автоматичне виконання |
| Явна помилка (hallucination, format error) | FAIL → повернути на перегенерацію |
| Неоднозначність, потрібен контекст | ESCALATE → передати людині |

### 15.3 Вимоги до впровадження

- Критик має бути **іншою моделлю** або **іншим промптом**
- Критик не замінює human approval для HIGH-risk операцій
- Всі рішення критика логуються в `case_events`
- Людина може переглядати рішення критика в аудит-логу

> **Принцип:** LLM-критик — це фільтр, який зменшує approval fatigue, але не замінює людський контроль для критичних рішень.

---

**Наступний документ:** [04_case_cockpit_ux.md](./04_case_cockpit_ux.md) — як approvals виглядають і працюють в UI.
