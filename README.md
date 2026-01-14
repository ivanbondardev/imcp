# IMCP — Iron Man Case Platform

> **AI generates, orchestrates and monitors.  
> Humans verify, decide and remain accountable.**

IMCP is a partial autonomy platform for managing complex business cases.

---

## Core Concept

The **Iron Man Suit** metaphor:
- **Human** — the pilot who verifies decisions and remains accountable
- **AI** — the exoskeleton that accelerates analysis and removes routine
- **System** — a suit, not an autopilot

---

## How to Read the Documentation

### 1. Start with Core Documentation

📁 **[docs/core/](./docs/core/)** — platform core

| Order | Document | What You'll Learn |
|-------|----------|-------------------|
| 1 | [Shared Mental Model](./docs/core/00_shared_mental_model.md) | How we think about the system |
| 2 | [Architecture Overview](./docs/core/01_architecture_overview.md) | Components and their boundaries |
| 3 | [Core Data Model](./docs/core/02_core_data_model.md) | Data structure (contract) |
| 4 | [Approval Pattern](./docs/core/03_approval_pattern.md) | Human-in-the-loop mechanism |
| 5 | [Case Cockpit UX](./docs/core/04_case_cockpit_ux.md) | UI/UX patterns |

### 2. Explore a Case Example

📁 **[docs/case_templates/f1_sea_import/](./docs/case_templates/f1_sea_import/)** — sea import from China

This is a complete case type example with:
- state machine (states and transitions)
- approval gates (verification points)
- automation rules
- UI spec (interface specification)
- test scenarios

---

## Tech Stack

| Component | Technology | Role |
|-----------|------------|------|
| **UI** | Next.js | Case Cockpit (pilot cabin) |
| **Database** | Supabase (Postgres) | Source of Truth |
| **Storage** | Supabase Storage | Documents |
| **Orchestration** | n8n | Workflow automation |
| **AI** | LLM (via n8n) | Extract / Generate / Verify |

---

## Architectural Principles

```
Supabase  — source of truth
n8n       — executor and orchestrator  
UI        — pilot cabin
AI        — a tool, not a decision-maker
```

---

## 🎮 Live Demo

**[▶️ Open Demo](https://ivbon.github.io/imcp/)** — інтерактивна демонстрація платформи

| Сторінка | Опис |
|----------|------|
| [Landing](https://ivbon.github.io/imcp/) | Головна сторінка платформи |
| [Case List](https://ivbon.github.io/imcp/demo/case-list.html) | Список справ з фільтрами |
| [Case Detail](https://ivbon.github.io/imcp/demo/case-detail.html) | Детальний огляд справи |
| [Timeline](https://ivbon.github.io/imcp/demo/timeline.html) | Хронологія подій |
| [Documents](https://ivbon.github.io/imcp/demo/documents.html) | Управління документами |
| [Approvals](https://ivbon.github.io/imcp/demo/approvals.html) | Погодження та верифікація |

---

## Quick Links

- 📖 [Core Documentation](./docs/core/README.md)
- 🚢 [F1 Sea Import Template](./docs/case_templates/f1_sea_import/)
- 📊 [Data Model Contract](./docs/core/02_core_data_model.md)

---

## License

This project is licensed under the [Apache License 2.0](./LICENSE).

Copyright © 2026 Ivan Bondar

---

## 🤝 Let's Collaborate

**Open for collaboration!**

If you're interested in the IMCP project and would like to:

- 🚀 **Implement IMCP** in your organization
- 💡 **Discuss customization** for your business processes
- 🤖 **Integrate AI solutions** for case management automation
- 📚 **Get consulting** on Human-in-the-Loop system architecture

**I'd love to connect!**

📧 Reach out to discuss partnership and implementation opportunities.

> *"AI empowers humans but doesn't replace their responsibility"* — this isn't just a slogan, it's a principle I can help bring to life in your business.
