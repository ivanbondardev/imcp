# Documents Specification

Централізований хаб управління документами з AI-екстракцією.

---

## Scope / Non-scope (для продуктового ревʼю)

**Scope (P0):**
- Черга документів, які потребують верифікації
- Viewer + швидка верифікація/правка витягнутих даних

**Non-scope (P1+):**
- Повний DMS (версіювання/права/шаблони) як окремий продукт
- Масові batch‑операції над десятками документів

---

## MVP (P0) vs Next (P1)

**P0:**
- Verification queue (needs review first)
- Upload flow + статуси + viewer
- Верифікація витягнутих даних (людина підтверджує/править)

**P1:**
- Batch verify / bulk actions
- Розширені правила routing (case_type specific)

---

## Entry points / Deep links

- `/documents` (вхід з навігації)
- `/documents/:document_id` (viewer)
- З документу є шлях у `/cases/:case_number`

---

## Мета

Documents — черга верифікації документів, що відповідає на питання: **"Які документи потребують моєї уваги?"**

### Основні задачі
- **Верифікація AI-екстракції**: перевірити та підтвердити дані
- **Огляд документів**: всі документи по кейсах
- **Завантаження**: нові документи до кейсів
- **Моніторинг AI**: статистика extraction rate

---

## Documents ≠ Case Cockpit Documents Tab

| Documents Page | Case Cockpit Tab |
|----------------|------------------|
| Всі документи по всіх кейсах | Документи одного кейса |
| Черга верифікації | Статус документів кейса |
| Batch operations | View/upload |

---

## Інформаційна архітектура

### KPI блок
- Потребують верифікації
- Завантажено сьогодні
- Верифіковано за період
- AI Extraction Rate (avg confidence)

### Verification Queue
Список документів що потребують перевірки людиною:
- File name + type badge
- Source
- Case link
- AI confidence %
- Status badge
- View / Verify buttons

### Recently Uploaded
Таблиця нещодавно завантажених документів з усіма метаданими.

---

## Document Statuses

| Status | Опис |
|--------|------|
| `UPLOADED` | Щойно завантажено |
| `PROCESSING` | AI обробляє |
| `VERIFIED` | Людина підтвердила |
| `REPLACED` | Замінено новою версією |
| `ARCHIVED` | Архівовано |

**Важливо**: "needs human review" — це не статус, а derived flag на основі confidence threshold.

---

## Confidence Thresholds

| Рівень | Обробка |
|--------|---------|
| High (≥95%) | Quick verify або auto-verify |
| Medium | Стандартна верифікація |
| Low (<threshold) | Пріоритетна перевірка |

Низький confidence візуально виділяється.

---

## UX принципи

### Human-in-the-Loop
AI екстрагує → людина верифікує.

### Confidence-based routing
Документи з низьким confidence — пріоритет.

### Case context
Кожен документ прив'язаний до кейса.

### Clear status
Статуси з відповідними кольорами.

---

## Upload Flow

1. Drag-and-drop або file picker
2. Вибір кейса (required)
3. Опціонально: тип документа (AI визначить автоматично)
4. Upload → status = UPLOADED
5. Workflow (n8n) ставить `status=PROCESSING` на час обробки (опційно, але канонічний статус існує в core)
6. Після екстракції:
   - якщо confidence >= threshold: workflow може auto‑merge у `cases.payload.*` і перевести документ у `VERIFIED`
   - якщо confidence < threshold: документ **не стає VERIFIED**; UI показує derived flag “needs human review” (це не статус), а workflow додає risk/flag у `cases.computed.*`

Supported formats: PDF, Excel, Word, Images

---

## Verification Modal

Side-by-side view:
- Document preview
- AI extracted data
- Confidence score
- Checkbox "Дані коректні"
- Actions (MVP): **Підтвердити** / **Редагувати & Підтвердити** / **Замінити файлом (нова версія)**

Примітка щодо контракту (core):
- UI не змінює `cases.state/status/computed`.
- Верифікація фіксується як дія людини: UPDATE `cases.payload.*` (за потреби) + INSERT `case_events` (`DOC_VERIFIED`).
- Переведення `documents.status` у `VERIFIED` робить workflow (n8n) у відповідь на людську верифікацію (event-driven).

---

## Document Types (приклад F1_SEA_IMPORT)

| Type | Опис | AI Extraction |
|------|------|---------------|
| `CONTRACT` | Контракт | Parties, amounts |
| `INVOICE` | Інвойс | Items, totals |
| `PACKING_LIST` | Пакувальний лист | Dims, weights |
| `BL_DRAFT` | Коносамент | BL fields |
| `POA` | Довіреність | Signatures, dates |

---

## Core залежності

Дивись [02_core_data_model.md](../core/02_core_data_model.md) для:
- `documents.status` enum
- AI extraction schema
- RLS policies

---

## Edge cases (P0)

- **Upload succeeded, processing failed**: статус/прапорець + CTA “Retry processing” (якщо дозволено) або “Replace file”
- **Replaced version**: UI показує, що є нова версія, і веде на актуальну
- **RLS denied**: viewer не розкриває метадані документу, якщо немає доступу

---

## Product Acceptance Checklist (P0)

- [ ] На `/documents` є verification queue (“needs review first”) і видно link на кейс
- [ ] Для кожного документа видно: file/type, source, status, confidence (якщо є)
- [ ] Upload flow: після завантаження документ зʼявляється у списку зі статусом `UPLOADED`/`PROCESSING`
- [ ] Viewer/verification: preview + extracted data + правка + підтвердження (без зайвих кроків)
- [ ] “needs human review” показаний як derived flag (не статус) і пояснюється “чому”
- [ ] Empty state: “нема документів для верифікації” + CTA “Upload”
