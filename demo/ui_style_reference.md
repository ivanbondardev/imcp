Ось єдиний Markdown-документ, який об’єднує Light Modern (VSCode Built-in) + Cursor Panels/Windows UI reference 👇

⸻

🎨 UI Style Reference — Light Modern (VSCode) + Cursor Panels/Windows

1) Загальний вайб (Base Style)

Light Modern (VSCode Built-in)
	•	Світлий фон, нейтральний “clean”
	•	Мінімум візуального шуму
	•	Контраст середній/високий
	•	Акценти стримані, але читабельні

Cursor UI
	•	Основа як у VSCode, але більш “продуктово”
	•	Панелі тримаються на відступах + hover, а не на рамках
	•	Акцент тонкий, не агресивний
	•	AI UI виглядає як окремий модуль (chat + actions + code blocks)

⸻

2) Базова палітра (Colors)

Backgrounds
	•	Editor background: #FFFFFF
	•	Sidebar background: #F3F3F3
	•	Panel background: #F3F3F3
	•	Hover background: #E8E8E8
	•	Active line highlight: #F5F5F5

Borders / Dividers
	•	Border: #D0D0D0
	•	Subtle divider: #E0E0E0

Text
	•	Primary text: #1E1E1E
	•	Secondary text: #5A5A5A
	•	Muted text: #808080

Accent / Selection
	•	Accent blue: #007ACC
	•	Accent hover: #006BB3
	•	Selection bg: #ADD6FF
	•	Focus outline: #007ACC

⸻

3) Типографіка (Typography)

Fonts
	•	UI: system-ui, -apple-system, Segoe UI, Roboto, sans-serif
	•	Code: Consolas, 'Courier New', monospace

Sizes
	•	Base text: 13–14px
	•	Small labels: 12px
	•	Titles: 16–20px
	•	Line-height: 1.4–1.6

⸻

4) Компонентні стани (UI States)

Buttons

Primary
	•	bg: #007ACC
	•	text: #FFFFFF
	•	hover: #006BB3

Secondary
	•	bg: #F3F3F3
	•	border: #D0D0D0
	•	hover: #E8E8E8

Inputs
	•	bg: #FFFFFF
	•	border: #D0D0D0
	•	placeholder: #808080
	•	focus border: #007ACC

Selection / Highlight
	•	text selection: #ADD6FF
	•	hover row bg (light): rgba(0,0,0,0.04)

⸻

5) Тіні / Глибина (Shadows)

Light Modern / Cursor уникають “важких” тіней:
	•	Card shadow: 0 1px 2px rgba(0,0,0,0.08)
	•	Dropdown shadow: 0 4px 12px rgba(0,0,0,0.12)
	•	Popup shadow (Command Palette): 0 8px 24px rgba(0,0,0,0.18)

⸻

6) Cursor Layout Reference (Panels & Windows)

Основні області інтерфейсу
	•	Title Bar / Top Bar
	•	Activity Bar (ліва вертикальна ікон-панель)
	•	Side Bar (ліва панель: explorer/search/git)
	•	Editor Area (центр)
	•	Bottom Panel (terminal/output/problems)
	•	Right Panel (AI Chat / Composer / Diff / Outline)

⸻

7) Панелі (Sidebars / Panels)

Left Sidebar (Explorer / Search / SCM)

Стиль
	•	фон трохи “сіріший” за редактор
	•	мінімум рамок
	•	hover виділення рядка
	•	активний елемент має тонкий accent

Референс
	•	Row height: 22–26px
	•	Padding left: 10–14px
	•	Hover radius: 6px
	•	Hover bg: rgba(0,0,0,0.04)

⸻

Right Sidebar (Cursor AI Chat / Composer)

Стиль
	•	виглядає як окремий модуль
	•	чат-формат, інпут знизу (sticky)
	•	code blocks в окремих контейнерах
	•	actions під відповідями: Copy / Insert / Apply / Diff

Референс
	•	Width: 360–480px
	•	Message padding: 10–12px
	•	Code block bg: #F3F3F3
	•	Border: #D0D0D0

⸻

8) Editor Tabs / Windows

Editor Tabs
	•	пласкі таби, мінімум рамок
	•	активний таб контрастніший
	•	hover підсвітка на табах

Референс
	•	Tab height: 32–36px
	•	Tab padding: 10–14px
	•	Active tab bg: #FFFFFF
	•	Inactive tab bg: #F3F3F3
	•	Hover bg: #E8E8E8

Split Editors
	•	divider тонкий, майже непомітний
Divider: 1px solid #E0E0E0

⸻

9) Bottom Panel (Terminal / Output / Problems)

Стиль
	•	відділений top-border
	•	вкладки компактні
	•	фон як у sidebar/panel

Референс
	•	Default height: 240–320px
	•	Tab row height: 28–32px
	•	Border top: #D0D0D0

⸻

10) Floating Windows / Popups

Command Palette (Ctrl/Cmd+P)
	•	floating modal по центру
	•	легкий backdrop
	•	rounded corners
	•	середня тінь
	•	список результатів під інпутом

Референс
	•	Width: 600–720px
	•	Radius: 10–12px
	•	Shadow: 0 8px 24px rgba(0,0,0,0.18)
	•	Item height: 34–38px
	•	Active item bg: #ADD6FF

Context Menu / Dropdown
	•	bg: #FFFFFF
	•	border: #D0D0D0
	•	shadow: 0 6px 18px rgba(0,0,0,0.12)
	•	radius: 8px
	•	item height: 28–32px

⸻

11) Cards / Blocks (Cursor AI Responses)

Стиль
	•	“документ в документі”
	•	border + padding
	•	code section окремо

Референс
	•	card bg: #FAFAFA
	•	card border: #E0E0E0
	•	radius: 10px
	•	padding: 12px

⸻

12) Syntax Highlight Reference (VSCode Light)
	•	Keyword: #0000FF
	•	String: #A31515
	•	Number: #098658
	•	Function: #795E26
	•	Comment: #008000
	•	Type: #267F99
	•	Variable: #001080

⸻

13) Design Tokens (готові CSS змінні)

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


⸻

14) Cursor-like Checklist (швидка перевірка)

✅ Плоскі панелі + мінімум рамок
✅ Hover замість “обводок всюди”
✅ Акцент тонкий (не кричущий)
✅ Floating palette з хорошою тінню
✅ AI panel: чат + actions + code blocks
✅ Невеликі кнопки editor-style
