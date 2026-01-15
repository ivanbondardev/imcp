# UI Style Reference
## Light Modern (VSCode) + Cursor Panels/Windows

**Версія:** 1.0  
**Статус:** Core  
**Тип документа:** Guide  
**Аудиторія:** UX/UI, Frontend, Product  

---

## 1. Загальний вайб (Base Style)

### Light Modern (VSCode Built-in)
- Світлий фон, нейтральний "clean"
- Мінімум візуального шуму
- Контраст середній/високий
- Акценти стримані, але читабельні

### Cursor UI
- Основа як у VSCode, але більш "продуктово"
- Панелі тримаються на відступах + hover, а не на рамках
- Акцент тонкий, не агресивний
- AI UI виглядає як окремий модуль (chat + actions + code blocks)

---

## 2. Базова палітра (Colors)

### Backgrounds

| Token | Value | Usage |
|-------|-------|-------|
| `--bg-editor` | `#FFFFFF` | Editor background |
| `--bg-sidebar` | `#F3F3F3` | Sidebar background |
| `--bg-panel` | `#F3F3F3` | Panel background |
| `--bg-hover` | `rgba(0,0,0,0.04)` | Hover background |
| Active line | `#F5F5F5` | Active line highlight |

### Borders / Dividers

| Token | Value | Usage |
|-------|-------|-------|
| `--border` | `#D0D0D0` | Border |
| `--divider` | `#E0E0E0` | Subtle divider |

### Text

| Token | Value | Usage |
|-------|-------|-------|
| `--text` | `#1E1E1E` | Primary text |
| `--text-secondary` | `#5A5A5A` | Secondary text |
| `--text-muted` | `#808080` | Muted text |

### Accent / Selection

| Token | Value | Usage |
|-------|-------|-------|
| `--accent` | `#007ACC` | Accent blue |
| `--accent-hover` | `#006BB3` | Accent hover |
| `--selection` | `#ADD6FF` | Selection bg |
| Focus outline | `#007ACC` | Focus outline |

---

## 3. Типографіка (Typography)

### Fonts

| Type | Font Stack |
|------|------------|
| **UI** | `system-ui, -apple-system, Segoe UI, Roboto, sans-serif` |
| **Code** | `Consolas, 'Courier New', monospace` |

### Sizes

| Element | Size |
|---------|------|
| Base text | 13–14px |
| Small labels | 12px |
| Titles | 16–20px |
| Line-height | 1.4–1.6 |

---

## 4. Компонентні стани (UI States)

### Buttons

**Primary:**
| Property | Value |
|----------|-------|
| bg | `#007ACC` |
| text | `#FFFFFF` |
| hover | `#006BB3` |

**Secondary:**
| Property | Value |
|----------|-------|
| bg | `#F3F3F3` |
| border | `#D0D0D0` |
| hover | `#E8E8E8` |

### Inputs

| Property | Value |
|----------|-------|
| bg | `#FFFFFF` |
| border | `#D0D0D0` |
| placeholder | `#808080` |
| focus border | `#007ACC` |

### Selection / Highlight

| Property | Value |
|----------|-------|
| text selection | `#ADD6FF` |
| hover row bg | `rgba(0,0,0,0.04)` |

---

## 5. Тіні / Глибина (Shadows)

Light Modern / Cursor уникають "важких" тіней:

| Token | Value | Usage |
|-------|-------|-------|
| `--shadow-sm` | `0 1px 2px rgba(0,0,0,0.08)` | Card shadow |
| `--shadow-md` | `0 4px 12px rgba(0,0,0,0.12)` | Dropdown shadow |
| `--shadow-popup` | `0 8px 24px rgba(0,0,0,0.18)` | Popup shadow |

---

## 6. Border Radius

| Token | Value | Usage |
|-------|-------|-------|
| `--radius-sm` | `6px` | Small elements, hover states |
| `--radius-md` | `10px` | Cards, panels |
| `--radius-lg` | `12px` | Modals, popups |

---

## 7. Layout Reference (Panels & Windows)

### Основні області інтерфейсу

```
┌──────────────────────────────────────────────────────────────┐
│ Title Bar / Top Bar                                          │
├────┬─────────────────────────────────────────────┬───────────┤
│    │                                             │           │
│ A  │                                             │   Right   │
│ c  │           Editor Area                       │   Panel   │
│ t  │                                             │   (AI)    │
│ i  │                                             │           │
│ v  ├─────────────────────────────────────────────┤           │
│ i  │                                             │           │
│ t  │ Side Bar (Explorer/Search/Git)              │           │
│ y  │                                             │           │
├────┴─────────────────────────────────────────────┴───────────┤
│ Bottom Panel (Terminal / Output / Problems)                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Панелі (Sidebars / Panels)

### Left Sidebar (Explorer / Search / SCM)

| Property | Value |
|----------|-------|
| Row height | 22–26px |
| Padding left | 10–14px |
| Hover radius | 6px |
| Hover bg | `rgba(0,0,0,0.04)` |

**Стиль:**
- Фон трохи "сіріший" за редактор
- Мінімум рамок
- Hover виділення рядка
- Активний елемент має тонкий accent

### Right Sidebar (AI Chat / Composer)

| Property | Value |
|----------|-------|
| Width | 360–480px |
| Message padding | 10–12px |
| Code block bg | `#F3F3F3` |
| Border | `#D0D0D0` |

**Стиль:**
- Виглядає як окремий модуль
- Чат-формат, інпут знизу (sticky)
- Code blocks в окремих контейнерах
- Actions під відповідями: Copy / Insert / Apply / Diff

---

## 9. Editor Tabs / Windows

### Editor Tabs

| Property | Value |
|----------|-------|
| Tab height | 32–36px |
| Tab padding | 10–14px |
| Active tab bg | `#FFFFFF` |
| Inactive tab bg | `#F3F3F3` |
| Hover bg | `#E8E8E8` |

**Стиль:**
- Пласкі таби, мінімум рамок
- Активний таб контрастніший
- Hover підсвітка на табах

### Split Editors

| Property | Value |
|----------|-------|
| Divider | `1px solid #E0E0E0` |

---

## 10. Bottom Panel (Terminal / Output / Problems)

| Property | Value |
|----------|-------|
| Default height | 240–320px |
| Tab row height | 28–32px |
| Border top | `#D0D0D0` |

**Стиль:**
- Відділений top-border
- Вкладки компактні
- Фон як у sidebar/panel

---

## 11. Floating Windows / Popups

### Command Palette (Ctrl/Cmd+P)

| Property | Value |
|----------|-------|
| Width | 600–720px |
| Radius | 10–12px |
| Shadow | `0 8px 24px rgba(0,0,0,0.18)` |
| Item height | 34–38px |
| Active item bg | `#ADD6FF` |

**Стиль:**
- Floating modal по центру
- Легкий backdrop
- Rounded corners
- Середня тінь
- Список результатів під інпутом

### Context Menu / Dropdown

| Property | Value |
|----------|-------|
| bg | `#FFFFFF` |
| border | `#D0D0D0` |
| shadow | `0 6px 18px rgba(0,0,0,0.12)` |
| radius | `8px` |
| item height | 28–32px |

---

## 12. Cards / Blocks (AI Responses)

| Property | Value |
|----------|-------|
| card bg | `#FAFAFA` |
| card border | `#E0E0E0` |
| radius | `10px` |
| padding | `12px` |

**Стиль:**
- "Документ в документі"
- Border + padding
- Code section окремо

---

## 13. Syntax Highlight Reference (VSCode Light)

| Element | Color |
|---------|-------|
| Keyword | `#0000FF` |
| String | `#A31515` |
| Number | `#098658` |
| Function | `#795E26` |
| Comment | `#008000` |
| Type | `#267F99` |
| Variable | `#001080` |

---

## 14. Design Tokens (CSS Variables)

```css
:root {
  /* Backgrounds */
  --bg-editor: #ffffff;
  --bg-sidebar: #f3f3f3;
  --bg-panel: #f3f3f3;
  --bg-hover: rgba(0,0,0,0.04);

  /* Borders / dividers */
  --border: #d0d0d0;
  --divider: #e0e0e0;

  /* Text */
  --text: #1e1e1e;
  --text-secondary: #5a5a5a;
  --text-muted: #808080;

  /* Accent */
  --accent: #007acc;
  --accent-hover: #006bb3;
  --selection: #add6ff;

  /* Radius */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 12px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.08);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.12);
  --shadow-popup: 0 8px 24px rgba(0,0,0,0.18);
}
```

---

## 15. Quick Checklist

При створенні UI компонентів перевіряйте:

- [ ] Плоскі панелі + мінімум рамок
- [ ] Hover замість "обводок всюди"
- [ ] Акцент тонкий (не кричущий)
- [ ] Floating palette з хорошою тінню
- [ ] AI panel: чат + actions + code blocks
- [ ] Невеликі кнопки editor-style
- [ ] Використано CSS variables з секції 14
