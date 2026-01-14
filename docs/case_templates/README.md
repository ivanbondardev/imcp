# Case Templates
## IMCP — Шаблони типів кейсів

**Призначення:** Шаблони для різних типів бізнес-кейсів у платформі IMCP

---

## 🛠️ Як створити новий Case Template

📄 **[IMPLEMENTATION_FRAMEWORK.md](./IMPLEMENTATION_FRAMEWORK.md)** — повний покроковий гайд

Фреймворк включає:
- 10 фаз імплементації
- Питання для інтерв'ю з SME та адміністраторами систем (1С, email, партнери)
- Naming conventions та glossary
- Чеклісти якості та Definition of Done

---

## 📁 Існуючі Case Templates

| Case Type | Опис | Статус |
|-----------|------|--------|
| [f1_sea_import](./f1_sea_import/) | Морський імпорт з Китаю (Yiwu/Shenzhen → Україна) | ✅ Reference |

---

## 📂 Структура Case Template

Кожен case_template складається з 10 документів:

```
{case_type}/
├── 00_case_overview.md           # Огляд кейса, Definition of Done
├── 01_states_and_flow.md         # State machine, переходи
├── 02_required_data_and_documents.md  # Поля payload, документи
├── 03_approval_gates.md          # Human-in-the-loop точки
├── 04_automation_and_ai_rules.md # Автоматизація, AI функції
├── 05_sequences.md               # Mermaid sequence diagrams
├── 06_ui_case_cockpit_spec.md    # UI специфікація
├── 07_n8n_workflows_map.md       # Карта n8n workflows
├── 08_test_scenarios.md          # Тестові сценарії
└── 09_sample_payloads.md         # Приклади JSON
```

---

## 🔗 Пов'язана документація

- 📖 [Core Documentation](../core/README.md) — ядро платформи
- 📊 [Data Model Contract](../core/02_core_data_model.md) — модель даних
- 🎯 [Approval Pattern](../core/03_approval_pattern.md) — патерн підтверджень
