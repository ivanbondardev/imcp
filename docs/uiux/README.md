# IMCP UI/UX Documentation
## Специфікації інтерфейсів платформи

**Версія:** 1.0  
**Статус:** Active  
**Призначення:** Документація UI/UX компонентів IMCP

---

## Структура документації

| Документ | Тип | Аудиторія | Опис |
|----------|-----|-----------|------|
| [ui_style_reference.md](./ui_style_reference.md) | Guide | UX/UI, Frontend | Дизайн-токени, палітра, компоненти |
| [owner_dashboard_spec.md](./owner_dashboard_spec.md) | Spec | Business Owner, Ops Lead, Product, UX | Специфікація Owner Dashboard |
| [personal_settings_spec.md](./personal_settings_spec.md) | Spec | Product, UX/UI, Frontend, Architecture | Персональні налаштування менеджера |

---

## Зв'язок з Core документацією

UI/UX документи базуються на core специфікаціях:

| Core документ | Що визначає для UI |
|---------------|-------------------|
| [00_shared_mental_model.md](../core/00_shared_mental_model.md) | UX-принципи, розподіл людина/AI |
| [01_architecture_overview.md](../core/01_architecture_overview.md) | UI Contract — що UI може/не може |
| [02_core_data_model.md](../core/02_core_data_model.md) | Структура даних, JSONB schemas |
| [03_approval_pattern.md](../core/03_approval_pattern.md) | Approval UI patterns |
| [04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md) | Case Cockpit — основний менеджерський UI |

---

## UI Style Reference

Усі UI компоненти мають відповідати дизайн-токенам:

- [ui_style_reference.md](./ui_style_reference.md) — палітра, типографіка, компоненти

---

## Типи документів

| Тип | Опис |
|-----|------|
| **Spec** | Детальна специфікація екрану/компоненту |
| **Pattern** | Переиспользуваний UI патерн |
| **Guide** | Гайд для імплементації |

---

## Roadmap документації

### Completed

- [x] `personal_settings_spec.md` — персональні налаштування та система нотифікацій

### Planned

- [ ] `mobile_patterns.md` — адаптація для mobile
- [ ] `accessibility_guide.md` — a11y вимоги

---

## Порядок читання

1. **[Core README](../core/README.md)** — загальне розуміння платформи
2. **[ui_style_reference.md](./ui_style_reference.md)** — дизайн-система
3. **[04_case_cockpit_ux.md](../core/04_case_cockpit_ux.md)** — основний UI для менеджерів
4. **[owner_dashboard_spec.md](./owner_dashboard_spec.md)** — UI для бізнес-овнерів
5. **[personal_settings_spec.md](./personal_settings_spec.md)** — персональні налаштування
