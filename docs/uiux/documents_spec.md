# Documents Specification
## IMCP — Управління документами кейсів

**Версія:** 1.0  
**Статус:** Spec (PoC → MVP)  
**Тип документа:** Spec  
**Аудиторія:** Product, UX/UI, Frontend, Operations Manager  
**Changelog:**  
- v1.0 — initial spec based on demo/documents.html

**Попередні документи:**  
- [00_shared_mental_model.md](../core/00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](../core/01_architecture_overview.md) — архітектура  
- [02_core_data_model.md](../core/02_core_data_model.md) — модель даних  
- [03_approval_pattern.md](../core/03_approval_pattern.md) — патерн підтверджень  

**Пов'язані документи:**  
- [UI Style Reference](./ui_style_reference.md) — дизайн-токени  
- [Case List Spec](./case_list_spec.md) — операційна черга кейсів  
- [Personal Settings Spec](./personal_settings_spec.md) — персональні налаштування  

---

## 1. Purpose — Навіщо потрібна сторінка Documents

Documents в IMCP — це **централізований хаб управління документами** з інтеграцією AI-екстракції даних.

### 1.1 Основні задачі менеджера

| Задача | Опис |
|--------|------|
| **Верифікація AI-екстракції** | Перевірити та підтвердити дані, витягнуті AI |
| **Огляд документів** | Бачити всі завантажені документи по кейсах |
| **Завантаження** | Завантажити нові документи до кейсів |
| **Фільтрація** | Знайти документи за статусом, типом, кейсом |
| **Моніторинг AI** | Бачити статистику AI extraction rate |

### 1.2 Documents ≠ Case Cockpit Documents Tab

| Аспект | Documents Page | Case Cockpit Documents Tab |
|--------|----------------|---------------------------|
| **Фокус** | Всі документи по всіх кейсах | Документи одного кейса |
| **Аудиторія** | Менеджер, Data Operator | Менеджер кейса |
| **Ключове питання** | "Які документи потребують верифікації?" | "Які документи є в цьому кейсі?" |
| **Дії** | Batch verify, filter, upload | View, upload to case |
| **AI Focus** | Verification queue | Document status |

> **Documents Page** — це "черга верифікації документів" + глобальний огляд.

### 1.3 Відповідність Shared Mental Model

Documents реалізує принципи [Shared Mental Model](../core/00_shared_mental_model.md):

| Принцип | Реалізація в Documents |
|---------|------------------------|
| **Human-in-the-Loop** | AI екстрагує, людина верифікує |
| **Confidence-Based Routing** | Документи з низьким confidence виділені |
| **Fatigue-Aware Design** | Верифікаційна черга замість хаотичного списку |
| **Actionable Interface** | View та Verify кнопки на кожному документі |
| **Clear Status** | NEEDS VERIFICATION, VERIFIED, PROCESSING badges |

---

## 2. Scope (PoC vs MVP)

### 2.1 PoC (Minimal Viable)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| **Stats Overview** | HIGH | 4 KPI tiles (pending, uploaded today, verified, AI rate) |
| **Verification Queue** | HIGH | Список документів що потребують верифікації |
| **Recently Uploaded** | HIGH | Таблиця нещодавно завантажених документів |
| **Document Type Reference** | MEDIUM | Референсна таблиця типів документів |
| **Workflow Visualization** | LOW | Візуалізація Upload → Extract → Verify |

### 2.2 MVP (Extended)

| Функціональність | Пріоритет | Опис |
|------------------|-----------|------|
| PoC features | — | Все з PoC |
| **Upload Modal** | HIGH | Drag-and-drop upload з прив'язкою до кейса |
| **Advanced Filters** | MEDIUM | Фільтри по типу, статусу, кейсу, дате |
| **Verification Modal** | MEDIUM | Side-by-side: документ + AI extracted data |
| **Bulk Verify** | LOW | Multi-select + batch verification |
| **Search** | LOW | Пошук по назві документа, кейсу |

### 2.3 v2 (Scale)

| Функціональність | Опис |
|------------------|------|
| OCR Preview | Показ розпізнаного тексту |
| AI Confidence Tuning | Налаштування порогів confidence |
| Document Comparison | Порівняння версій документа |
| Auto-categorization | AI визначає тип документа |
| Export | Bulk download документів |

---

## 3. Information Architecture

### 3.1 Загальна структура сторінки

```
┌──────────────────────────────────────────────────────────────┐
│ Sidebar Navigation                                            │
├──────────────────────────────────────────────────────────────┤
│ Header: "Документи" + Actions (Фільтри, Завантажити)          │
├──────────────────────────────────────────────────────────────┤
│ Stats Grid (4 KPI tiles)                                      │
├──────────────────────────────────────────────────────────────┤
│ Section: "Потребують верифікації" (Verification Queue)        │
│ └── Doc list with View/Verify actions                         │
├──────────────────────────────────────────────────────────────┤
│ Section: "Нещодавно завантажені" (Recently Uploaded)          │
│ └── Table with document details                               │
├──────────────────────────────────────────────────────────────┤
│ Reference: Document Types (collapsible)                       │
├──────────────────────────────────────────────────────────────┤
│ Reference: Document Flow Visualization                        │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Sidebar Navigation

| Секція | Пункти | Badge |
|--------|--------|-------|
| **Main** | Owner Dashboard, Кейси, Підтвердження, Документи (active), Таймлайн | Кейси: count, Підтвердження: pending |
| **Фільтри** | Нещодавно завантажені, Потребують верифікації, Верифіковані | Потребують верифікації: count |
| **Налаштування** | Персональні | — |

### 3.3 Drill-down Model

```
Documents Page → Doc View → Document Viewer Modal
      ↓
Doc Click → Case Cockpit (linked case)
      ↓
Upload Button → Upload Modal → Documents Page (refreshed)
      ↓
Verify Button → Verification Modal → Documents Page (updated)
```

---

## 4. Stats Overview (KPI Tiles)

### 4.1 KPI Tiles Specification

| KPI | Label | Опис | Порогове значення | Icon |
|-----|-------|------|-------------------|------|
| **Потребують верифікації** | Needs Verification | Документи з AI extracted, pending human check | > 0 = warning | Alert Circle |
| **Завантажено сьогодні** | Uploaded Today | Документи завантажені сьогодні | vs yesterday = trend | Upload |
| **Верифіковано** | Verified | Верифіковані документи за місяць | — | Check Circle |
| **AI Extraction Rate** | AI Confidence | Avg confidence rate AI екстракції | — | Cookie (AI icon) |

### 4.2 KPI Tile Behavior

| Tile | Click Action | Color Logic |
|------|--------------|-------------|
| Потребують верифікації | → Filter to pending docs | 🟡 warning if > 0 |
| Завантажено сьогодні | → Filter to today's uploads | 🔵 primary |
| Верифіковано | → Filter to verified | 🟢 success |
| AI Extraction Rate | → Show extraction stats | 🔵 info |

### 4.3 SQL Queries для KPI

```sql
-- Потребують верифікації
SELECT COUNT(*) 
FROM documents 
WHERE org_id = $org_id
  AND status = 'NEEDS_VERIFICATION';

-- Завантажено сьогодні
SELECT COUNT(*) 
FROM documents 
WHERE org_id = $org_id
  AND created_at >= CURRENT_DATE;

-- Верифіковано (цього місяця)
SELECT COUNT(*) 
FROM documents 
WHERE org_id = $org_id
  AND status = 'VERIFIED'
  AND verified_at >= date_trunc('month', now());

-- AI Extraction Rate (avg confidence)
SELECT ROUND(AVG((ai_extraction->>'confidence')::numeric), 0) as avg_confidence
FROM documents 
WHERE org_id = $org_id
  AND ai_extraction IS NOT NULL
  AND created_at >= date_trunc('month', now());
```

---

## 5. Document Statuses

### 5.1 Status Lifecycle

```
UPLOADED → PROCESSING → NEEDS_VERIFICATION → VERIFIED
                ↓                 ↓
            FAILED         REJECTED (re-upload)
```

### 5.2 Status Definitions

| Status | Опис | UI Treatment |
|--------|------|--------------|
| `UPLOADED` | Щойно завантажений, очікує обробки | 🔵 info badge |
| `PROCESSING` | AI обробляє / екстрагує дані | 🔵 info badge + spinner |
| `NEEDS_VERIFICATION` | AI extracted, потребує людської перевірки | 🟡 warning badge |
| `LOW_CONFIDENCE` | AI confidence < 80%, пріоритетна перевірка | 🔴 danger badge |
| `VERIFIED` | Людина підтвердила AI extraction | 🟢 success badge |
| `REJECTED` | Документ відхилено (помилка, дублікат) | 🔴 danger badge |
| `FAILED` | AI не зміг обробити | 🔴 danger badge |

### 5.3 Confidence Thresholds

| Confidence | Status | UI Treatment |
|------------|--------|--------------|
| ≥ 95% | Auto-verify candidate | 🟢 Can skip manual verification |
| 85-94% | NEEDS_VERIFICATION | 🟡 Standard queue |
| < 85% | LOW_CONFIDENCE | 🔴 Priority review, highlighted |

---

## 6. Verification Queue Section

### 6.1 Section Header

```
☑ ПОТРЕБУЮТЬ ВЕРИФІКАЦІЇ                                    [4]
```

**Іконка:** Warning circle (🟡)  
**Badge:** Count of pending documents

### 6.2 Document Item Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ [📄]  Invoice_INV2026-0451.pdf                                  │
│       INVOICE • ZED • Case: F1-SEA-2026-02451 • AI: 92%         │
│                                    [NEEDS VERIFICATION] [View] [Verify] │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Document Item Data

| Елемент | Data Key | Приклад |
|---------|----------|---------|
| Icon | — | 📄 Document icon (color by status) |
| Name | `file_name` | Invoice_INV2026-0451.pdf |
| Type | `doc_type` | INVOICE |
| Source | `source` | ZED, CLIENT, BROKER |
| Case Link | `case_number` | F1-SEA-2026-02451 |
| AI Confidence | `ai_extraction.confidence` | 92% |
| Status Badge | `status` | NEEDS_VERIFICATION |
| Actions | — | View, Verify buttons |

### 6.4 Confidence Indicator Colors

| Умова | Колір | CSS |
|-------|-------|-----|
| Confidence ≥ 90% | default (text-secondary) | — |
| Confidence 80-89% | 🟡 warning | `color: var(--warning)` |
| Confidence < 80% | 🔴 danger | `color: var(--danger)` |

### 6.5 Action Buttons

| Button | Style | Action |
|--------|-------|--------|
| View | `btn-ghost btn-sm` | Open document viewer modal |
| Verify | `btn-success btn-sm` | Open verification modal / Quick verify |

---

## 7. Recently Uploaded Table

### 7.1 Table Columns

| Колонка | Ключ | Опис | Приклад |
|---------|------|------|---------|
| Документ | `file_name` | Icon + filename | 📄 Contract_ACME_2026.pdf |
| Тип | `doc_type` | Type badge | `CONTRACT` |
| Case | `case_number` | Link to case | F1-SEA-2026-02451 |
| Джерело | `source` | Upload source | CLIENT, ZED, BROKER |
| Статус | `status` | Status badge | VERIFIED, PROCESSING |
| Завантажено | `created_at` | Relative time | Сьогодні, 11:30 |
| Action | — | View button | [View] |

### 7.2 Table Sorting

| Column | Default | Sortable |
|--------|---------|----------|
| Документ | — | No |
| Тип | — | Yes (MVP) |
| Case | — | Yes (MVP) |
| Джерело | — | Yes (MVP) |
| Статус | — | Yes (MVP) |
| Завантажено | DESC | Yes |

### 7.3 Case Link Behavior

- **Click:** Navigate to Case Cockpit → Documents tab
- **Style:** `color: var(--accent)` underline on hover

---

## 8. Document Types Reference

### 8.1 Reference Table (F1_SEA_IMPORT)

| doc_type | Опис | Джерело | AI Extraction | Required State |
|----------|------|---------|---------------|----------------|
| `CONTRACT` | Контракт з клієнтом | CLIENT | Extract parties, amounts | CLIENT_INFO_COLLECTED |
| `INVOICE` | Інвойс постачальника | CLIENT / ZED | Extract items, totals | CONFIRMED |
| `PACKING_LIST` | Пакувальний лист | CLIENT | Extract dims, weights | CLIENT_INFO_COLLECTED |
| `BL_DRAFT` | Драфт коносаменту | ZED | Extract BL fields | BL_DRAFT_RECEIVED |
| `POA` | Довіреність | CLIENT | Verify signatures, dates | PRE_ARRIVAL_TASKS |
| `EXPORT_DEC` | Експортна декларація | BROKER | Limited extraction | Optional |

### 8.2 Display Behavior

- **PoC:** Показується як статична таблиця-довідник
- **MVP:** Collapsible card, збережений стан collapse
- **v2:** Фільтрація по case_type

---

## 9. Document Flow Visualization

### 9.1 Flow Steps

```
📤 UPLOADED    →    🤖 PROCESSING    →    ✅ VERIFIED
Human uploads       AI extracts           Human confirms
file               data                   extraction
```

### 9.2 Flow Visual Specification

| Step | Icon | Background | Text |
|------|------|------------|------|
| UPLOADED | 📤 | `var(--info-bg)` | `var(--info-text)` |
| PROCESSING | 🤖 | `var(--warning-bg)` | `var(--warning-text)` |
| VERIFIED | ✅ | `var(--success-bg)` | `var(--success-text)` |

### 9.3 Flow Note

> Якщо AI confidence < 85%, документ автоматично потребує верифікації людиною

---

## 10. Upload Modal (MVP)

### 10.1 Modal Structure

```
┌─────────────────────────────────────────────────────────────┐
│ Завантажити документ                                   [×]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │                                                         │ │
│ │          📄 Перетягніть файли сюди                      │ │
│ │               або натисніть для вибору                  │ │
│ │                                                         │ │
│ │          PDF, XLSX, DOCX, PNG, JPG (до 10MB)           │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ [Selected files preview - if any]                           │
│                                                             │
│ Кейс *                                                      │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [Оберіть кейс... ▾]                                     │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Тип документа (optional)                                    │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [AI визначить автоматично ▾]                            │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                           [Скасувати]  [📤 Завантажити]     │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Upload Fields

| Поле | Type | Required | Опис |
|------|------|----------|------|
| Files | File Input | ✅ | Drag-and-drop або file picker |
| Case | Select | ✅ | Пошук по case_number, client |
| doc_type | Select | ❌ | Optional, AI auto-detect if not set |

### 10.3 Supported File Types

| Format | Extensions | Max Size |
|--------|------------|----------|
| PDF | `.pdf` | 10MB |
| Excel | `.xlsx`, `.xls` | 10MB |
| Word | `.docx`, `.doc` | 10MB |
| Images | `.png`, `.jpg`, `.jpeg` | 10MB |

### 10.4 Upload Flow

```
1. User drops/selects files
2. User selects case (required)
3. User optionally selects doc_type
4. Click "Завантажити"
   ↓
5. API: POST /documents with multipart
   ↓
6. Files uploaded, status = UPLOADED
   ↓
7. n8n trigger → AI processing
   ↓
8. Status → PROCESSING → NEEDS_VERIFICATION
   ↓
9. Modal closes, toast: "X файлів завантажено"
10. Page refreshes, new docs in table
```

---

## 11. Verification Modal (MVP)

### 11.1 Modal Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│ Верифікація документа                                          [×]  │
│ Invoice_INV2026-0451.pdf                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ ┌────────────────────────────┐ ┌──────────────────────────────────┐ │
│ │                            │ │ AI Extracted Data                │ │
│ │                            │ │ ──────────────────────────────── │ │
│ │    [Document Preview]      │ │                                  │ │
│ │                            │ │ Invoice Number: INV2026-0451     │ │
│ │    (PDF viewer)            │ │ Date: 2026-01-12                 │ │
│ │                            │ │ Total: $12,450.00                │ │
│ │                            │ │ Currency: USD                    │ │
│ │                            │ │ Vendor: ACME Supplies            │ │
│ │                            │ │                                  │ │
│ │                            │ │ ──────────────────────────────── │ │
│ │                            │ │ AI Confidence: 92%               │ │
│ │                            │ │ ──────────────────────────────── │ │
│ │                            │ │                                  │ │
│ │                            │ │ [ ] Дані коректні                │ │
│ │                            │ │                                  │ │
│ │                            │ │ Коментар (optional):             │ │
│ │                            │ │ ┌──────────────────────────────┐ │ │
│ │                            │ │ │                              │ │ │
│ │                            │ │ └──────────────────────────────┘ │ │
│ └────────────────────────────┘ └──────────────────────────────────┘ │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│           [Відхилити]  [Редагувати дані]  [✅ Підтвердити]          │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Verification Actions

| Action | Button Style | Result |
|--------|--------------|--------|
| Підтвердити | `btn-success` | status → VERIFIED, ai_extraction.verified = true |
| Редагувати дані | `btn-secondary` | Open extracted data edit form |
| Відхилити | `btn-danger` (ghost) | status → REJECTED, requires reason |

### 11.3 Quick Verify (High Confidence)

Для документів з confidence ≥ 95%:
- Показати "Quick Verify" кнопку прямо в списку
- Click → API call → Status updated → Toast success
- Не потрібно відкривати модальне вікно

---

## 12. Sidebar Quick Filters

### 12.1 Filter Options

| Фільтр | Icon | Опис | Query |
|--------|------|------|-------|
| Нещодавно завантажені | Upload | Документи за останні 7 днів | `created_at > now() - 7d` |
| Потребують верифікації | Checkbox | Pending verification | `status = 'NEEDS_VERIFICATION'` |
| Верифіковані | Check Circle | Successfully verified | `status = 'VERIFIED'` |

### 12.2 Filter Behavior

- Фільтри є **exclusive** (radio логіка)
- Активний фільтр підсвічується
- Badge показує count документів
- Клік на активний фільтр — деактивує його (show all)

---

## 13. Wireframe — Documents (PoC)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ IMCP ▸ Документи                                                [User Avatar]│
├────────────────────┬─────────────────────────────────────────────────────────┤
│ ІМСР               │ Документи                                               │
│ Case Platform      │ Управління документами кейсів                           │
│ ─────────────────  │                                 [Фільтри]  [Завантажити]│
│                    │ ───────────────────────────────────────────────────────│
│ □ Owner Dashboard  │                                                         │
│ □ Кейси        12  │ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│ □ Підтвердження 3  │ │ Потребують│ │Завантажено│ │Верифіковано│ │ AI Extr. │ │
│ ■ Документи        │ │ верифік.  │ │ сьогодні  │ │           │ │   Rate   │ │
│ □ Таймлайн         │ │           │ │           │ │           │ │          │ │
│                    │ │     4     │ │     8     │ │    156    │ │   89%    │ │
│ ─────────────────  │ │    🟡     │ │    🔵     │ │    🟢     │ │    🔵    │ │
│ Фільтри            │ └───────────┘ └───────────┘ └───────────┘ └───────────┘ │
│ ─────────────────  │                                                         │
│ 📤 Нещодавно       │ ─────────────────────────────────────────────────────── │
│    завантажені     │ ⚠ ПОТРЕБУЮТЬ ВЕРИФІКАЦІЇ                             4  │
│ ☑ Потребують       │ ─────────────────────────────────────────────────────── │
│   верифікації   4  │                                                         │
│ ✓ Верифіковані     │ ┌─────────────────────────────────────────────────────┐ │
│                    │ │ 📄 Invoice_INV2026-0451.pdf                          │ │
│ ─────────────────  │ │    INVOICE • ZED • Case: F1-SEA-2026-02451          │ │
│ Налаштування       │ │    AI: 92% confidence                               │ │
│ ─────────────────  │ │                 [NEEDS VERIFICATION] [View] [Verify] │ │
│ ⚙ Персональні      │ ├─────────────────────────────────────────────────────┤ │
│                    │ │ 📄 BL_Draft_MSKU1234567890.pdf                       │ │
│                    │ │    BL_DRAFT • ZED • Case: F1-SEA-2026-02445         │ │
│                    │ │    AI: 76% confidence (LOW!)                        │ │
│                    │ │                   [LOW CONFIDENCE] [View] [Verify]   │ │
│                    │ └─────────────────────────────────────────────────────┘ │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ 📤 НЕЩОДАВНО ЗАВАНТАЖЕНІ                                │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ ┌─────────────────────────────────────────────────────┐ │
│                    │ │ Документ       │ Тип      │ Case      │ Статус    │ │
│                    │ │ ───────────────────────────────────────────────────│ │
│                    │ │ Contract_ACME  │ CONTRACT │ F1-SEA-.. │ VERIFIED  │ │
│                    │ │ Invoice_INV..  │ INVOICE  │ F1-SEA-.. │ PROCESSING│ │
│                    │ │ Packing_List   │ PACK_LST │ F1-SEA-.. │ VERIFIED  │ │
│                    │ │ BL_Draft_MSKU  │ BL_DRAFT │ F1-SEA-.. │ NEEDS VER │ │
│                    │ └─────────────────────────────────────────────────────┘ │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ ℹ ТИПИ ДОКУМЕНТІВ (F1_SEA_IMPORT)              [expand] │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
│                    │ ─────────────────────────────────────────────────────── │
│                    │ 📤 → 🤖 → ✅  DOCUMENT FLOW                             │
│                    │ UPLOADED  PROCESSING  VERIFIED                          │
│                    │ ─────────────────────────────────────────────────────── │
│                    │                                                         │
└────────────────────┴─────────────────────────────────────────────────────────┘
```

---

## 14. UX Behavior & Interactions

### 14.1 Document Row Click Behavior

| Element | Action | Result |
|---------|--------|--------|
| Row (except buttons) | Click | → Open Document Viewer modal |
| Case link | Click | → Navigate to Case Cockpit |
| "View" button | Click | → Open Document Viewer modal |
| "Verify" button | Click | → Open Verification modal |

### 14.2 Stats Tile Click

| Tile | Action |
|------|--------|
| Потребують верифікації | Filter to NEEDS_VERIFICATION docs |
| Завантажено сьогодні | Filter to today's uploads |
| Верифіковано | Filter to VERIFIED docs |
| AI Extraction Rate | Show AI stats modal (MVP) |

### 14.3 Upload Button

| Trigger | Behavior |
|---------|----------|
| Button click | Open Upload modal |
| ESC key | Close modal |
| Overlay click | Close modal |
| Submit success | Close modal + toast + refresh list |
| Submit error | Show error, keep modal open |

### 14.4 Empty States

| Section | Empty State |
|---------|-------------|
| Потребують верифікації | "Немає документів для верифікації 🎉" |
| Нещодавно завантажені | "Немає нещодавно завантажених документів" |
| All sections empty | "Завантажте перший документ" + Upload CTA |

### 14.5 Loading States

| Component | Loading Behavior |
|-----------|-----------------|
| Stats tiles | Skeleton loaders (4 boxes) |
| Verification Queue | Skeleton list items |
| Recently Uploaded table | Skeleton table rows |
| Upload Modal | Submit button shows spinner |
| Verification Modal | Document preview shows loader |

---

## 15. Data Sources (Supabase)

### 15.1 Core Tables

| Таблиця | Дані для Documents |
|---------|-------------------|
| `documents` | Document records, status, AI extraction |
| `cases` | Case reference for linking |
| `users` | Uploader info, verifier info |

### 15.2 Documents Table Schema

```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id),
  case_id UUID NOT NULL REFERENCES cases(id),
  
  -- File info
  file_name TEXT NOT NULL,
  file_path TEXT NOT NULL,       -- S3/Supabase Storage path
  file_size INTEGER,
  mime_type TEXT,
  
  -- Document classification
  doc_type TEXT,                  -- CONTRACT, INVOICE, BL_DRAFT, etc.
  source TEXT,                    -- CLIENT, ZED, BROKER
  
  -- Status
  status TEXT NOT NULL DEFAULT 'UPLOADED',  -- UPLOADED, PROCESSING, NEEDS_VERIFICATION, VERIFIED, REJECTED, FAILED
  
  -- AI extraction
  ai_extraction JSONB,            -- {confidence: 92, fields: {...}, extracted_at: ...}
  
  -- Verification
  verified_by UUID REFERENCES auth.users(id),
  verified_at TIMESTAMPTZ,
  verification_comment TEXT,
  
  -- Audit
  uploaded_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### 15.3 ai_extraction JSONB Schema

```json
{
  "confidence": 92,
  "extracted_at": "2026-01-15T10:30:00Z",
  "model": "gpt-4-vision",
  "fields": {
    "invoice_number": "INV2026-0451",
    "date": "2026-01-12",
    "total": 12450.00,
    "currency": "USD",
    "vendor": "ACME Supplies"
  },
  "raw_text": "...",
  "verified": false
}
```

### 15.4 Views (рекомендовано)

```sql
-- View for documents with case info
CREATE VIEW v_documents_list AS
SELECT 
  d.id,
  d.file_name,
  d.doc_type,
  d.source,
  d.status,
  d.ai_extraction,
  d.created_at,
  d.verified_at,
  -- Case info
  c.case_number,
  c.id as case_id,
  -- Uploader info
  jsonb_build_object(
    'id', u.id,
    'name', u.raw_user_meta_data->>'full_name'
  ) as uploaded_by,
  -- Computed
  CASE 
    WHEN (d.ai_extraction->>'confidence')::int < 85 THEN true 
    ELSE false 
  END as is_low_confidence
FROM documents d
JOIN cases c ON c.id = d.case_id
LEFT JOIN auth.users u ON u.id = d.uploaded_by;
```

### 15.5 RLS Policies

```sql
-- Users can see documents from their org
CREATE POLICY "org_documents" ON documents
  FOR SELECT
  USING (
    org_id IN (SELECT org_id FROM user_orgs WHERE user_id = auth.uid())
  );

-- Users can upload to cases they have access to
CREATE POLICY "upload_documents" ON documents
  FOR INSERT
  WITH CHECK (
    case_id IN (
      SELECT id FROM cases 
      WHERE owner_user_id = auth.uid() 
         OR id IN (SELECT case_id FROM case_assignments WHERE user_id = auth.uid())
    )
  );
```

---

## 16. UI Components

### 16.1 Document Icon

```css
.doc-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: var(--radius-sm);    /* 6px */
  background: var(--bg-panel);
  color: var(--text-secondary);
}

.doc-icon.warning {
  background: var(--warning-bg);
  color: var(--warning-text);
}

.doc-icon.success {
  background: var(--success-bg);
  color: var(--success-text);
}
```

### 16.2 Document Status Badge

```css
.badge {
  display: inline-flex;
  padding: 4px 8px;
  font-size: 11px;
  font-weight: 500;
  border-radius: var(--radius-sm);
  text-transform: uppercase;
  white-space: nowrap;
}

.badge-warning {
  background: #FEF3C7;
  color: #92400E;
}

.badge-danger {
  background: #FEE2E2;
  color: #991B1B;
}

.badge-success {
  background: #D1FAE5;
  color: #065F46;
}

.badge-info {
  background: #DBEAFE;
  color: #1E40AF;
}

.badge-gray {
  background: #F3F4F6;
  color: #4B5563;
}
```

### 16.3 Document Item

```css
.doc-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid var(--divider);
}

.doc-item:last-child {
  border-bottom: none;
}

.doc-item:hover {
  background: var(--bg-hover);
}

.doc-info {
  flex: 1;
}

.doc-name {
  font-weight: 500;
  color: var(--text);
}

.doc-meta {
  display: flex;
  gap: 4px;
  font-size: 12px;
  color: var(--text-muted);
  margin-top: 2px;
}

.doc-actions {
  display: flex;
  gap: 8px;
}
```

### 16.4 Flow Step

```css
.flow-step {
  flex: 1;
  min-width: 120px;
  text-align: center;
  padding: 14px;
  border-radius: var(--radius-lg);
}

.flow-step .icon {
  font-size: 20px;
  margin-bottom: 6px;
}

.flow-step .label {
  font-weight: 600;
}

.flow-step .description {
  font-size: 12px;
  color: var(--text-muted);
}

.flow-arrow {
  color: var(--text-muted);
}
```

---

## 17. Accessibility (A11y)

### 17.1 Keyboard Navigation

| Key | Action |
|-----|--------|
| Tab | Navigate through interactive elements |
| Enter/Space | Activate button/link |
| Escape | Close modal |
| Arrow Up/Down | Navigate document list (MVP) |

### 17.2 ARIA Attributes

```html
<!-- Document List -->
<ul role="list" aria-label="Документи для верифікації">
  <li role="listitem" tabindex="0" aria-label="Invoice_INV2026-0451.pdf, потребує верифікації">
    <!-- ... -->
  </li>
</ul>

<!-- Status Badge -->
<span 
  class="badge badge-warning" 
  role="status"
  aria-label="Статус: потребує верифікації"
>
  NEEDS VERIFICATION
</span>

<!-- Upload Modal -->
<div 
  role="dialog" 
  aria-modal="true" 
  aria-labelledby="upload-modal-title"
>
  <h3 id="upload-modal-title">Завантажити документ</h3>
  <!-- ... -->
</div>

<!-- Drop Zone -->
<div 
  role="button" 
  aria-label="Перетягніть файли сюди або натисніть для вибору"
  tabindex="0"
>
  <!-- ... -->
</div>
```

### 17.3 Focus Management

| Action | Focus Behavior |
|--------|----------------|
| Page load | Focus on first interactive element |
| Open Upload modal | Focus on drop zone |
| Open Verification modal | Focus on document preview |
| Close modal | Return focus to trigger button |
| After verify | Focus on next item in queue |

---

## 18. Anti-patterns (чого НЕ робити)

| ❌ Anti-pattern | Проблема | ✅ Рішення |
|-----------------|----------|------------|
| Показувати всі документи без групування | Важко знайти pending docs | Групування: Verification Queue + Recent |
| Приховувати AI confidence | Менеджер не знає пріоритет | Завжди показувати confidence % |
| Auto-verify без review | Можливі помилки AI | Human-in-the-loop для всіх docs |
| Складний multi-step upload | Friction | Single modal, minimal fields |
| Verification без document preview | Неможливо порівняти | Side-by-side: doc + AI data |
| No feedback on upload | User uncertainty | Progress bar + toast on complete |

---

## 19. Acceptance Criteria (Definition of Done)

### 19.1 PoC Acceptance

Documents v0 вважається успішним, якщо:

**Functional:**
- [ ] Менеджер бачить 4 KPI tiles з актуальними даними
- [ ] Менеджер бачить Verification Queue з pending документами
- [ ] Менеджер бачить Recently Uploaded таблицю
- [ ] Клік "View" → документ відкривається в модалі
- [ ] Badge показує правильний статус документа
- [ ] Case link веде до Case Cockpit

**UX:**
- [ ] Документи з низьким confidence (< 85%) візуально виділені
- [ ] Status badges мають правильні кольори
- [ ] Document flow visualization зрозуміла
- [ ] Responsive layout (desktop-first)

**Performance:**
- [ ] Page load < 1s
- [ ] Document list renders < 50 items without lag

### 19.2 MVP Acceptance (додатково)

- [ ] Upload modal з drag-and-drop
- [ ] Verification modal з side-by-side preview
- [ ] Quick Verify для high confidence docs
- [ ] Advanced filters (type, status, date)
- [ ] Search by filename/case

---

## 20. Roadmap: v0 → v1 → v2

### v0 (PoC)

| Компонент | Включено |
|-----------|----------|
| Stats tiles (4) | ✅ |
| Verification Queue | ✅ |
| Recently Uploaded table | ✅ |
| Document Type Reference | ✅ |
| Flow Visualization | ✅ |
| View document | ✅ |
| Upload modal | — |
| Verification modal | — |

### v1 (MVP)

| Компонент | Включено |
|-----------|----------|
| v0 features | ✅ |
| Upload modal | ✅ |
| Verification modal | ✅ |
| Quick Verify | ✅ |
| Advanced filters | ✅ |
| Search | ✅ |
| Bulk verify | — |

### v2 (Scale)

| Компонент | Включено |
|-----------|----------|
| v1 features | ✅ |
| Bulk verify | ✅ |
| OCR Preview | ✅ |
| AI Confidence Tuning | ✅ |
| Document Comparison | ✅ |
| Auto-categorization | ✅ |
| Export/Download | ✅ |

---

## 21. Technical Implementation Notes

### 21.1 Frontend Implementation

| Аспект | Рекомендація |
|--------|--------------|
| **Framework** | React + Supabase JS client |
| **State** | React Query for server state |
| **File Upload** | react-dropzone |
| **PDF Preview** | react-pdf or pdf.js |
| **Forms** | React Hook Form for modals |

### 21.2 AI Extraction Integration

```typescript
// n8n webhook trigger after upload
// POST /api/webhooks/document-uploaded
{
  "document_id": "uuid",
  "file_path": "s3://bucket/path",
  "doc_type": "INVOICE" // optional hint
}

// n8n workflow:
// 1. Download file from storage
// 2. Call OpenAI Vision API
// 3. Extract structured data
// 4. Update document record with ai_extraction
// 5. Set status = NEEDS_VERIFICATION or VERIFIED (if auto-verify enabled)
```

### 21.3 Real-time Updates

```typescript
// Supabase subscription for document status changes
supabase
  .channel('documents_changes')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'documents',
      filter: `org_id=eq.${orgId}`
    },
    (payload) => {
      // Refetch or update local state
      queryClient.invalidateQueries(['documents']);
    }
  )
  .subscribe();
```

### 21.4 Storage Integration

```typescript
// Supabase Storage for document files
const { data, error } = await supabase.storage
  .from('documents')
  .upload(`${orgId}/${caseId}/${fileName}`, file);

// Get signed URL for preview
const { signedURL } = await supabase.storage
  .from('documents')
  .createSignedUrl(filePath, 3600); // 1 hour expiry
```

---

## 22. Integration Points

### 22.1 Integration with Case Cockpit

| Взаємодія | Напрямок | Опис |
|-----------|----------|------|
| Case link | Documents → Cockpit | Navigate to `/cases/{id}#documents` |
| Upload from Cockpit | Cockpit → Documents | Same API, different entry point |
| Document sync | Both ways | Real-time updates |

### 22.2 Integration with n8n Workflows

| Workflow | Trigger | Action |
|----------|---------|--------|
| Document Processing | Document uploaded | AI extraction |
| Verification Notification | Low confidence detected | Notify manager |
| Case Progress | Document verified | Update case state |

### 22.3 Integration with Approvals

| Scenario | Flow |
|----------|------|
| Document-based approval | Verify document → Create approval request |
| Approval with attachment | Approval links to verified document |

---

## 23. End Note

Documents в IMCP — це **центральний хаб для Human-in-the-Loop верифікації**.

> Сторінка відповідає на головне питання: **"Які документи потребують моєї уваги?"**

Ключові принципи:
- AI екстрагує, людина верифікує (Human-in-the-Loop)
- Confidence-based пріоритезація (низький confidence = пріоритет)
- Швидкий доступ до верифікації (View + Verify)
- Clear status visualization (badges, colors)
- Document-Case linking (завжди є контекст кейса)

---

**Пов'язана документація:**
- [/docs/core/](../core/) — core документація IMCP
- [/docs/case_templates/](../case_templates/) — шаблони кейсів
- [ui_style_reference.md](./ui_style_reference.md) — дизайн-токени
- [case_list_spec.md](./case_list_spec.md) — Case List
- [personal_settings_spec.md](./personal_settings_spec.md) — персональні налаштування
