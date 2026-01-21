# UI Patterns Catalog (IMCP)

Каталог повторюваних патернів, які мають бути **однаковими** на всіх екранах.  
Ціль: швидке продуктове ревʼю без дублювання деталей у кожній специфікації.

---

## 1) Badges: `status` vs `state`

### `cases.status` (агрегат, P0)
- **OPEN**: є наступна дія / є pending approval / кейс активний
- **BLOCKED**: очікуємо зовнішній інпут (клієнт/партнер); UI приглушує “дії”
- **DONE**: завершено, read-only за замовчуванням
- **ARCHIVED**: закрито/скасовано, read-only

### `cases.state` (бізнес-стан, P0)
- Показуємо як “де ми зараз у процесі” (людською назвою + технічний ключ у tooltip, якщо потрібно)
- Не плутати зі статусом: **state багато, status мало**

---

## 2) NBA (Next Best Action) — канонічний формат (P0)

NBA відповідає на: **“що робити далі?”** і завжди містить:
- **Title** (1 рядок)
- **Why now** (1–2 рядки: причина/ризик/SLA)
- **Context bullets** (2–3 маркери максимум)
- **Primary CTA** (одна дія)
- **Secondary CTA** (0–2 дії)

Правила:
- NBA не перетворюється на “меню всього”
- Якщо потрібен approval — NBA веде в approval‑дію/деталь (а не напряму “міняє state”)

---

## 3) Risk flags (P0)

### Призначення
Ризики — це “що може піти не так” + “де перевірити”, а не просто червоні лейбли.

### Відображення
- 3 рівні: **OK / Warn / Critical**
- На картках/рядках: показуємо **1 головний ризик**, решту — по кліку

### Anti-noise
- не дублюємо один і той самий ризик у 3 місцях без потреби
- prefer progressive disclosure

---

## 4) Confidence & Uncertainty (P0)

### Як показуємо confidence
- Показуємо **по полях/аспектах**, коли це критично для рішення
- Рівні: High / Medium / Low (без надлишкової точності)

### Progressive disclosure для reasoning
- Reasoning доступний завжди, але **не займає** основний екран
- “Показати як розраховано” → відкриває панель/модалку

---

## 5) Approval UX (P0)

Approval — керований мікро‑процес:
1. **Summary**: що це і чому важливо
2. **Request snapshot**: що пропонує система
3. **Risks/flags**: що перевірити
4. **Decision**: Approve / Edit & Approve / Reject + comment
5. **Audit**: рішення відображається в Timeline

Принципи:
- мінімізуємо context switching (все на картці/деталі approval)
- reject завжди просить причину (коротко, але обов’язково)

---

## 6) Documents: “needs review” як derived flag (P0)

- `documents.status` — лише канонічні статуси (UPLOADED/PROCESSING/VERIFIED/REPLACED/ARCHIVED)
- “needs human review” — derived flag (confidence/ризики), **не статус**

Рекомендований UX:
- Queue показує “needs review first”
- верифікація: preview + extracted data + правка + підтвердження

---

## 7) Empty / Loading / Error states (P0)

### Empty
- пояснює “чому пусто”
- дає next action (CTA)

### Loading
- skeleton для списків/карток
- “last updated” для фонового refresh

### Error
- короткий меседж + `correlation_id` (якщо є)
- CTA: Retry / Back

---

## 8) Deep links (P0)

Усі ключові артефакти мають прямі посилання:
- Case (`case_number`)
- Approval (`approval_id`)
- Document (`document_id`)
- Timeline highlight (`case_event_id`)

