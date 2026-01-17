# Core Data Model  
## IMCP — Iron Man Case Platform (Supabase)

**Версія:** 1.1  
**Статус:** Core / Contract  
**Попередні документи:**  
- [00_shared_mental_model.md](./00_shared_mental_model.md) — ментальна модель  
- [01_architecture_overview.md](./01_architecture_overview.md) — архітектура  

**Наступні документи:**  
- [03_approval_pattern.md](./03_approval_pattern.md) — патерн підтверджень  

**Ціль:** описати **контрактну** модель даних IMCP — єдине джерело істини для реалізації.

---

## 1. Purpose — Навіщо цей документ

Цей документ є **специфікацією**, а не концептом. Він фіксує:

- **Канонічні таблиці** з точними полями та типами
- **State vs Status** — чітка термінологія
- **Event taxonomy** — повний перелік подій з metadata schema
- **DB invariants** — constraints, indexes, rules
- **JSONB contracts** — схеми для payload/computed з ownership
- **RLS model** — org/team/roles для доступу

---

## 2. Термінологія: State vs Status

> ⚠️ **Контракт:** у IMCP терміни `state` і `status` мають різне значення.

| Термін | Визначення | Приклад | Де зберігається |
|--------|------------|---------|-----------------|
| **state** | Бізнес-стан у state machine кейса | `WAITING_CLIENT_INFO`, `QUOTE_READY`, `BL_CONFIRMED` | `cases.state` |
| **status** | Агрегований технічний стан кейса | `OPEN`, `BLOCKED`, `DONE`, `ARCHIVED` | `cases.status` |

### 2.1 State (бізнес-стан)

- Визначається `case_type_config` для кожного типу кейса
- Переходи контролюються state machine
- Може бути багато станів (20+ для складних кейсів)
- Приклад для F1_SEA_IMPORT: `NEW` → `WAITING_CLIENT_INFO` → `QUOTE_READY` → ...

### 2.2 Status (технічний агрегат)

| Status | Опис | Коли |
|--------|------|------|
| `OPEN` | Кейс активний, є що робити | Є pending approvals або NBA |
| `BLOCKED` | Кейс заблоковано, потрібна зовнішня дія | Чекаємо клієнта/партнера |
| `DONE` | Кейс завершено успішно | Досягнуто фінальний state |
| `ARCHIVED` | Кейс закрито/скасовано | Ручне закриття або timeout |

### 2.3 Primary State vs Parallel Substates

```
cases.state        = "BL_REVIEW_PENDING"     -- Primary State (one at a time)
cases.substates    = ["INSURANCE_PENDING", "BROKER_DOCS_PENDING"]  -- JSONB array
```

> **Правило:** Primary State — завжди один. Parallel substates — для незалежних підпроцесів.

---

## 3. Core Tables — Канонічний набір

### 3.1 `cases` (P0)

```sql
CREATE TABLE cases (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Identity
  case_type       TEXT NOT NULL,           -- 'F1_SEA_IMPORT', 'AIR_EXPORT'
  case_number     TEXT UNIQUE,             -- Human-readable: 'F1-SEA-2026-00123'
  title           TEXT,                    -- Optional short description
  
  -- State Machine
  state           TEXT NOT NULL,           -- Business state from case_type_config
  status          TEXT NOT NULL DEFAULT 'OPEN',  -- OPEN|BLOCKED|DONE|ARCHIVED
  substates       JSONB DEFAULT '[]',      -- Parallel substates array
  
  -- Ownership & Access
  org_id          UUID NOT NULL,           -- Organization
  team_id         UUID,                    -- Team (optional)
  owner_user_id   UUID NOT NULL,           -- Primary owner
  
  -- Priority & SLA
  priority        INT DEFAULT 0,           -- 0=normal, 1=high, 2=urgent
  sla_deadline    TIMESTAMPTZ,             -- When case should be resolved
  
  -- Domain Data (JSONB)
  payload         JSONB NOT NULL DEFAULT '{}',   -- Domain fields (see schemas)
  computed        JSONB NOT NULL DEFAULT '{}',   -- AI/workflow computed values
  
  -- Timestamps
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at       TIMESTAMPTZ,
  
  -- Constraints
  CONSTRAINT valid_status CHECK (status IN ('OPEN', 'BLOCKED', 'DONE', 'ARCHIVED'))
);

-- Indexes
CREATE INDEX idx_cases_org_status ON cases(org_id, status);
CREATE INDEX idx_cases_owner ON cases(owner_user_id, status);
CREATE INDEX idx_cases_type_state ON cases(case_type, state);
CREATE INDEX idx_cases_sla ON cases(sla_deadline) WHERE status = 'OPEN';
```

---

### 3.2 `case_events` (P0)

```sql
CREATE TABLE case_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id         UUID NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
  
  -- Event Identity
  event_type      TEXT NOT NULL,           -- See Event Taxonomy below
  
  -- Actor
  actor_type      TEXT NOT NULL,           -- HUMAN|SYSTEM|AI|INTEGRATION
  actor_id        TEXT,                    -- user_id or integration name
  
  -- Payload
  metadata        JSONB NOT NULL DEFAULT '{}',  -- Event-specific data
  
  -- Idempotency (optional)
  idempotency_key TEXT,                    -- For deduplication
  
  -- Timestamp (immutable)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT valid_actor_type CHECK (actor_type IN ('HUMAN', 'SYSTEM', 'AI', 'INTEGRATION'))
);

-- Indexes
CREATE INDEX idx_events_case_time ON case_events(case_id, created_at DESC);
CREATE INDEX idx_events_type ON case_events(event_type);
CREATE UNIQUE INDEX idx_events_idempotency ON case_events(case_id, idempotency_key) 
  WHERE idempotency_key IS NOT NULL;
```

> ⚠️ **Invariant:** `case_events` is **append-only**. No UPDATE or DELETE allowed (enforce via trigger or RLS).

---

### 3.3 `approvals` (P0)

```sql
CREATE TABLE approvals (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id           UUID NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
  
  -- Type & Status
  approval_type     TEXT NOT NULL,           -- See Approval Types
  status            TEXT NOT NULL DEFAULT 'PENDING',
  
  -- Request (immutable after creation)
  requested_by      TEXT NOT NULL,           -- user_id or 'SYSTEM'
  requested_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  request_snapshot  JSONB NOT NULL,          -- What system proposes
  
  -- Decision (filled when decided)
  decided_by        UUID,                    -- user_id
  decided_at        TIMESTAMPTZ,
  decision_snapshot JSONB,                   -- What human approved (may differ)
  decision_comment  TEXT,
  
  -- Idempotency
  idempotency_key   TEXT,
  
  -- Timestamps
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT valid_approval_status CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED', 'CANCELLED')),
  CONSTRAINT unique_idempotency UNIQUE (idempotency_key)
);

-- Indexes
CREATE INDEX idx_approvals_case ON approvals(case_id, status);
CREATE INDEX idx_approvals_pending ON approvals(status, requested_at) WHERE status = 'PENDING';
CREATE INDEX idx_approvals_decider ON approvals(decided_by) WHERE decided_by IS NOT NULL;
```

> ⚠️ **Invariant:** Decision update allowed **only if `status = 'PENDING'`** (optimistic locking).

```sql
-- Trigger to enforce immutable decision
CREATE OR REPLACE FUNCTION prevent_decision_change()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.status != 'PENDING' AND NEW.status != OLD.status THEN
    RAISE EXCEPTION 'Cannot change decision on non-PENDING approval';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_approval_immutable
  BEFORE UPDATE ON approvals
  FOR EACH ROW EXECUTE FUNCTION prevent_decision_change();
```

---

### 3.4 `documents` (P0/P1)

```sql
CREATE TABLE documents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id         UUID NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
  
  -- Document Identity
  doc_type        TEXT NOT NULL,             -- CONTRACT, INVOICE, BL_DRAFT, POA
  version         INT NOT NULL DEFAULT 1,
  
  -- Storage
  storage_path    TEXT NOT NULL,             -- Path in Supabase Storage
  file_name       TEXT NOT NULL,
  mime_type       TEXT NOT NULL,
  size_bytes      BIGINT,
  checksum        TEXT,                      -- SHA256 for integrity
  
  -- Status & Source
  status          TEXT NOT NULL DEFAULT 'UPLOADED',
  source          TEXT NOT NULL,             -- CLIENT|ZED|BROKER|SYSTEM|AI
  
  -- Upload Info
  uploaded_by     TEXT NOT NULL,
  uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Extracted Data
  extracted_data  JSONB DEFAULT '{}',        -- AI-extracted fields
  extraction_confidence NUMERIC,
  
  -- Metadata
  metadata        JSONB DEFAULT '{}',
  
  -- Constraints
  CONSTRAINT valid_doc_status CHECK (status IN ('UPLOADED', 'PROCESSING', 'VERIFIED', 'REPLACED', 'ARCHIVED')),
  CONSTRAINT valid_source CHECK (source IN ('CLIENT', 'ZED', 'BROKER', 'SYSTEM', 'AI'))
);

-- Indexes
CREATE INDEX idx_docs_case ON documents(case_id, doc_type);
CREATE INDEX idx_docs_case_version ON documents(case_id, doc_type, version DESC);
CREATE UNIQUE INDEX idx_docs_storage ON documents(storage_path);
```

---

### 3.5 `ai_runs` (P1)

```sql
CREATE TABLE ai_runs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id         UUID NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
  
  -- Run Type
  run_type        TEXT NOT NULL,             -- EXTRACT|GENERATE|VERIFY|CLASSIFY
  model           TEXT,                      -- gpt-4, claude-3, etc.
  prompt_version  TEXT,
  
  -- Input/Output
  input_refs      JSONB NOT NULL,            -- References to docs/events
  output          JSONB NOT NULL,            -- Structured result
  
  -- Quality
  confidence      NUMERIC,
  flags           JSONB DEFAULT '[]',        -- ['LOW_CONFIDENCE', 'NEEDS_REVIEW']
  
  -- Performance
  duration_ms     INT,
  tokens_used     INT,
  
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT valid_run_type CHECK (run_type IN ('EXTRACT', 'GENERATE', 'VERIFY', 'CLASSIFY'))
);

CREATE INDEX idx_ai_runs_case ON ai_runs(case_id, created_at DESC);
```

---

### 3.6 `integrations` (P1)

```sql
CREATE TABLE integrations (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id           UUID NOT NULL REFERENCES cases(id) ON DELETE CASCADE,
  
  -- Integration Identity
  integration_type  TEXT NOT NULL,           -- ONEC|SHEET|EMAIL|BROKER_API
  external_id       TEXT,                    -- ID in external system
  
  -- Status
  status            TEXT NOT NULL DEFAULT 'PENDING',
  last_error        TEXT,
  retry_count       INT DEFAULT 0,
  
  -- Data
  request_payload   JSONB,
  response_payload  JSONB,
  
  -- Correlation
  correlation_key   TEXT,                    -- For matching callbacks
  
  -- Timestamps
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_sync_at      TIMESTAMPTZ,
  
  CONSTRAINT valid_integration_status CHECK (status IN ('PENDING', 'SYNCED', 'FAILED', 'RETRYING'))
);

CREATE INDEX idx_integrations_case ON integrations(case_id, integration_type);
CREATE INDEX idx_integrations_external ON integrations(integration_type, external_id);
```

---

## 4. Event Taxonomy — Канонічні типи подій

> ⚠️ **Контракт:** Усі події мають використовувати типи з цієї таблиці.

### 4.1 Core Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `CASE_CREATED` | Кейс створено | HUMAN/SYSTEM | `{initial_state, source}` |
| `STATE_CHANGED` | Змінено бізнес-стан | HUMAN/SYSTEM | `{from_state, to_state, reason?}` |
| `STATUS_CHANGED` | Змінено технічний статус | SYSTEM | `{from_status, to_status, reason}` |
| `SUBSTATE_ADDED` | Додано паралельний підстан | SYSTEM | `{substate}` |
| `SUBSTATE_REMOVED` | Завершено паралельний підстан | SYSTEM | `{substate, resolution}` |

### 4.2 Approval Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `APPROVAL_CREATED` | Створено запит на підтвердження | SYSTEM | `{approval_id, approval_type, ai_generated?: bool}` |
| `APPROVAL_APPROVED` | Затверджено | HUMAN | `{approval_id, has_edits: bool, edited_fields?: string[]}` |
| `APPROVAL_REJECTED` | Відхилено | HUMAN | `{approval_id, reason}` |
| `APPROVAL_CANCELLED` | Скасовано | SYSTEM | `{approval_id, reason}` |

> **Примітка:** `ai_generated` вказує, чи approval був створений на основі AI-чернетки. `edited_fields` (optional) — список полів, які змінив користувач (для Quality Loop аналізу).

### 4.3 Document Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `DOC_UPLOADED` | Завантажено документ | HUMAN/INTEGRATION | `{document_id, doc_type, source}` |
| `DOC_EXTRACTED` | AI витягнув дані | AI | `{document_id, ai_run_id, fields_count}` |
| `DOC_VERIFIED` | Документ верифіковано людиною | HUMAN | `{document_id, changes_made: bool}` |
| `DOC_REPLACED` | Документ замінено новою версією | HUMAN | `{document_id, old_version, new_version}` |

### 4.4 Integration Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `INTEGRATION_STARTED` | Розпочато інтеграційну дію | SYSTEM | `{integration_id, type, action}` |
| `INTEGRATION_SUCCESS` | Інтеграція успішна | INTEGRATION | `{integration_id, external_id}` |
| `INTEGRATION_FAILED` | Інтеграція провалилась | INTEGRATION | `{integration_id, error, error_code?, retry_count, retryable?: bool}` |

> **Примітка:** `error_code` — нормалізований код помилки (TIMEOUT, AUTH, VALIDATION, RATE_LIMIT, UNKNOWN). `retryable` — чи можна автоматично повторити.

### 4.5 AI Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `AI_RUN_COMPLETED` | AI виклик завершено | AI | `{ai_run_id, run_type, confidence}` |
| `AI_SUGGESTION_CREATED` | AI створив пропозицію | AI | `{field, suggested_value, confidence}` |

### 4.6 Communication Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `EMAIL_SENT` | Відправлено email | SYSTEM | `{to, subject, template?}` |
| `NOTIFICATION_SENT` | Відправлено нотифікацію | SYSTEM | `{channel, recipient, type}` |
| `COMMENT_ADDED` | Додано коментар | HUMAN | `{text, mentions?: []}` |

### 4.7 Business Milestone Events

| event_type | Опис | actor_type | metadata schema |
|------------|------|------------|-----------------|
| `QUOTE_SENT` | Ціна/пропозиція відправлена клієнту | SYSTEM/HUMAN | `{quote_id?, sent_to, channel?}` |
| `CLIENT_CONFIRMED` | Клієнт підтвердив угоду | HUMAN/INTEGRATION | `{confirmation_ref?, channel}` |
| `SHIPMENT_DEPARTED` | Відправлення виїхало | INTEGRATION | `{tracking_id?, carrier}` |
| `SHIPMENT_ARRIVED` | Відправлення прибуло | INTEGRATION | `{arrival_date, location}` |

> **Примітка:** Business milestone events фіксують ключові бізнес-події, які використовуються для метрик (Time-to-quote, SLA compliance).

### 4.8 Metadata Schema Examples

```typescript
// STATE_CHANGED
{
  "from_state": "WAITING_CLIENT_INFO",
  "to_state": "CLIENT_INFO_COLLECTED",
  "reason": "all_required_fields_received"  // optional
}

// APPROVAL_APPROVED
{
  "approval_id": "uuid",
  "has_edits": true,  // human modified the draft
  "edited_fields": ["cargo.total_cbm", "quote.margin_usd"]  // optional, for Quality Loop
}

// QUOTE_SENT
{
  "quote_id": "uuid",
  "sent_to": "client@example.com",
  "channel": "EMAIL"
}

// INTEGRATION_FAILED
{
  "integration_id": "uuid",
  "error": "Connection timeout",
  "error_code": "TIMEOUT",  // normalized: TIMEOUT|AUTH|VALIDATION|RATE_LIMIT|UNKNOWN
  "retry_count": 2,
  "retryable": true
}
```

---

## 5. JSONB Contracts — Payload & Computed

### 5.1 Ownership Rules

> ⚠️ **Контракт:** Хто має право писати в які поля JSONB.

| JSONB Field | UI writes | n8n writes | AI writes | Notes |
|-------------|-----------|------------|-----------|-------|
| `payload.client` | ✅ | ❌ | ❌ | Клієнтські дані |
| `payload.cargo` | ✅ | ✅ (warehouse dims) | ✅ (extraction) | Дані вантажу |
| `payload.route` | ✅ | ❌ | ❌ | Маршрут |
| `payload.broker` | ✅ | ❌ | ❌ | Вибір брокера |
| `computed.quote` | ❌ | ✅ | ❌ | Розрахунок ціни |
| `computed.risks` | ❌ | ✅ | ✅ | Виявлені ризики |
| `computed.nba` | ❌ | ✅ | ❌ | Next Best Action |
| `computed.sla` | ❌ | ✅ | ❌ | Розрахунок дедлайнів |
| `computed.ai_extractions` | ❌ | ❌ | ✅ | Витягнуті дані |

### 5.2 Payload Schema per Case Type

```typescript
// F1_SEA_IMPORT payload schema
interface F1SeaImportPayload {
  client: {
    name: string;
    contact_name?: string;
    contact_email?: string;
    contact_phone?: string;
  };
  
  route: {
    mode: 'SEA';
    origin_country: string;
    origin_warehouse: 'YIWU' | 'SHENZHEN' | 'OTHER';
    destination_city: string;
    destination_terminal?: string;
    unloading_point?: string;
  };
  
  cargo: {
    ready_date: string;              // ISO date
    packages_count: number;
    dimensions: Array<{
      length_cm: number;
      width_cm: number;
      height_cm: number;
      quantity: number;
    }>;
    total_cbm?: number;              // calculated
    total_weight_kg?: number;
    stackable: boolean;
    dangerous_goods: boolean;
    dangerous_class?: string;
    description: string;
    
    // Warehouse-verified (n8n can write)
    warehouse_verified?: boolean;
    warehouse_dimensions?: Array<{...}>;
    warehouse_verified_at?: string;
  };
  
  broker: {
    owner: 'OUR' | 'CLIENT';
    name?: string;
    export_declaration_required: 'YES' | 'NO' | 'UNKNOWN';
  };
  
  insurance?: {
    required: boolean;
    type?: 'STANDARD' | 'FULL';
    value_usd?: number;
  };
}
```

### 5.3 Computed Schema

```typescript
interface CaseComputed {
  // Quote calculation (n8n writes after QUOTE_READY)
  quote?: {
    version: number;
    calculated_at: string;
    base_cost_usd: number;
    margin_usd: number;
    total_usd: number;
    validity_days: number;
    assumptions: string[];
    risks: string[];
  };
  
  // Risk flags (AI + n8n)
  risks?: Array<{
    code: string;            // 'DIMS_MISMATCH', 'DANGEROUS_GOODS', 'MISSING_EXPORT_DEC'
    severity: 'LOW' | 'MEDIUM' | 'HIGH';
    message: string;
    detected_at: string;
    detected_by: 'AI' | 'SYSTEM' | 'HUMAN';
  }>;
  
  // Next Best Action (n8n calculates)
  nba?: {
    action_code: string;     // 'APPROVE_QUOTE', 'UPLOAD_BL', 'VERIFY_DIMS'
    title: string;
    description: string;
    approval_required: boolean;
    approval_id?: string;
  };
  
  // SLA tracking (n8n calculates)
  sla?: {
    deadline: string;
    status: 'OK' | 'AT_RISK' | 'OVERDUE';
    hours_remaining?: number;
  };
  
  // AI extractions (AI writes)
  ai_extractions?: {
    [doc_type: string]: {
      ai_run_id: string;
      extracted_at: string;
      confidence: number;
      fields: Record<string, any>;
      needs_verification: boolean;
    };
  };
}
```

### 5.4 Versioning Computed

```typescript
// Кожен computed блок має версію для відстеження змін
computed.quote.version = 1;  // increment on recalculation

// При перерахунку:
// 1. Increment version
// 2. Log case_event: COMPUTED_UPDATED
// 3. If approval exists for old version → cancel it
```

---

## 6. DB Invariants — Constraints & Rules

### 6.1 Summary of Constraints

| Table | Constraint | Type | Purpose |
|-------|------------|------|---------|
| `cases` | `valid_status` | CHECK | Enum validation |
| `approvals` | `unique_idempotency` | UNIQUE | Prevent duplicates |
| `approvals` | `check_approval_immutable` | TRIGGER | No decision change after PENDING |
| `case_events` | `idx_events_idempotency` | UNIQUE | Event deduplication |
| `documents` | `idx_docs_storage` | UNIQUE | One record per file |

### 6.2 Indexes for Performance

```sql
-- Cases: operational queries
CREATE INDEX idx_cases_org_status ON cases(org_id, status);
CREATE INDEX idx_cases_owner ON cases(owner_user_id, status) WHERE status = 'OPEN';
CREATE INDEX idx_cases_sla ON cases(sla_deadline) WHERE status = 'OPEN';

-- Events: timeline queries
CREATE INDEX idx_events_case_time ON case_events(case_id, created_at DESC);

-- Approvals: inbox queries
CREATE INDEX idx_approvals_pending ON approvals(status, requested_at) 
  WHERE status = 'PENDING';
CREATE INDEX idx_approvals_user ON approvals(decided_by, decided_at DESC)
  WHERE decided_by IS NOT NULL;

-- Documents: case document list
CREATE INDEX idx_docs_case ON documents(case_id, doc_type, version DESC);
```

### 6.3 Optimistic Locking for Approvals

```sql
-- Safe approval decision update
UPDATE approvals 
SET 
  status = 'APPROVED',
  decided_by = $user_id,
  decided_at = now(),
  decision_snapshot = $snapshot,
  updated_at = now()
WHERE 
  id = $approval_id 
  AND status = 'PENDING'  -- Optimistic lock
RETURNING *;

-- If 0 rows returned → approval already decided, show error to user
```

### 6.4 Event Idempotency Pattern

```sql
-- Safe event creation with idempotency
INSERT INTO case_events (case_id, event_type, actor_type, actor_id, metadata, idempotency_key)
VALUES ($case_id, 'STATE_CHANGED', 'SYSTEM', 'workflow_123', $metadata, $idemp_key)
ON CONFLICT (case_id, idempotency_key) DO NOTHING
RETURNING *;

-- If NULL returned → event already exists (deduplicated)
```

---

## 7. RLS Model — Access Control

### 7.1 Access Model

```
Organization (org_id)
  └── Team (team_id) [optional]
       └── User (user_id)
            └── Role (role)
```

### 7.2 Roles & Permissions

| Role | Cases | Approvals | Documents | Events | Dashboard Access |
|------|-------|-----------|-----------|--------|------------------|
| `VIEWER` | Read own org | Read | Read | Read | — |
| `OPERATOR` | Read/Write own | Approve assigned types | Upload | Write | — |
| `MANAGER` | Read/Write team | Approve all types | All | Write | Owner Dashboard (read) |
| `ADMIN` | All in org | All | All | All | Owner + Engineering Dashboards |
| `OPS_LEAD` | Read/Write team | Approve escalations | All | Write | Owner Dashboard (full) |
| `ENGINEER` | Read all (tech) | Read | Read | Read | Engineering Dashboard (full) |

> **Примітка:** `OPS_LEAD` та `ENGINEER` — спеціалізовані ролі для дашбордів. `ENGINEER` має доступ до технічних метрик, але не до бізнес-рішень.

### 7.3 RLS Policies (Supabase)

```sql
-- Enable RLS
ALTER TABLE cases ENABLE ROW LEVEL SECURITY;
ALTER TABLE approvals ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE case_events ENABLE ROW LEVEL SECURITY;

-- Cases: org-based access
CREATE POLICY "cases_org_access" ON cases
  FOR ALL
  USING (org_id = auth.jwt() ->> 'org_id');

-- Approvals: same org + can decide only if permitted
CREATE POLICY "approvals_read" ON approvals
  FOR SELECT
  USING (
    case_id IN (SELECT id FROM cases WHERE org_id = auth.jwt() ->> 'org_id')
  );

CREATE POLICY "approvals_decide" ON approvals
  FOR UPDATE
  USING (
    status = 'PENDING'
    AND case_id IN (SELECT id FROM cases WHERE org_id = auth.jwt() ->> 'org_id')
    AND can_approve(auth.uid(), approval_type)  -- custom function
  );

-- Documents: via case access
CREATE POLICY "documents_access" ON documents
  FOR ALL
  USING (
    case_id IN (SELECT id FROM cases WHERE org_id = auth.jwt() ->> 'org_id')
  );

-- Events: read via case, write only by system/triggers
CREATE POLICY "events_read" ON case_events
  FOR SELECT
  USING (
    case_id IN (SELECT id FROM cases WHERE org_id = auth.jwt() ->> 'org_id')
  );

CREATE POLICY "events_insert" ON case_events
  FOR INSERT
  WITH CHECK (
    case_id IN (SELECT id FROM cases WHERE org_id = auth.jwt() ->> 'org_id')
  );
```

### 7.4 Approval Permissions Table

```sql
CREATE TABLE approval_permissions (
  role          TEXT NOT NULL,
  approval_type TEXT NOT NULL,
  can_approve   BOOLEAN DEFAULT true,
  PRIMARY KEY (role, approval_type)
);

-- Example data
INSERT INTO approval_permissions VALUES
  ('OPERATOR', 'QUOTE_APPROVAL', true),
  ('OPERATOR', 'BL_DRAFT_APPROVAL', false),  -- only MANAGER
  ('MANAGER', 'QUOTE_APPROVAL', true),
  ('MANAGER', 'BL_DRAFT_APPROVAL', true),
  ('MANAGER', 'ONEC_DEAL_CREATE_APPROVAL', true);

-- Helper function
CREATE FUNCTION can_approve(user_id UUID, approval_type TEXT) RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM user_roles ur
    JOIN approval_permissions ap ON ur.role = ap.role
    WHERE ur.user_id = $1 
      AND ap.approval_type = $2 
      AND ap.can_approve = true
  );
$$ LANGUAGE sql SECURITY DEFINER;
```

---

## 8. Case Type Configuration

> ⚠️ **Рішення для PoC/MVP:** Config-as-data у таблиці `case_type_configs`.

### 8.1 Config Table

```sql
CREATE TABLE case_type_configs (
  case_type       TEXT PRIMARY KEY,
  display_name    TEXT NOT NULL,
  
  -- State Machine
  states          JSONB NOT NULL,          -- All possible states
  initial_state   TEXT NOT NULL,
  final_states    JSONB NOT NULL,          -- ['CLOSED', 'CANCELLED']
  transitions     JSONB NOT NULL,          -- Allowed transitions
  
  -- Approvals
  approval_map    JSONB NOT NULL,          -- State → required approvals
  
  -- Fields
  required_fields JSONB NOT NULL,          -- Per-state required payload fields
  
  -- UI
  nba_rules       JSONB,                   -- Rules for computing NBA
  
  -- Metadata
  version         INT DEFAULT 1,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

### 8.2 Example: F1_SEA_IMPORT Config

```json
{
  "case_type": "F1_SEA_IMPORT",
  "display_name": "Морський імпорт (Китай → Україна)",
  
  "states": [
    "NEW",
    "WAITING_CLIENT_INFO",
    "CLIENT_INFO_COLLECTED",
    "QUOTE_READY",
    "QUOTE_APPROVAL_PENDING",
    "QUOTE_SENT",
    "WAITING_CLIENT_CONFIRMATION",
    "CONFIRMED",
    "DEAL_IMPORT_PENDING",
    "DEAL_IMPORT_CREATED",
    "WAREHOUSE_INSTRUCTIONS_SENT",
    "WAITING_WAREHOUSE_DIMS",
    "DIMS_APPROVAL_PENDING",
    "DIMS_CONFIRMED",
    "BL_REVIEW_PENDING",
    "BL_CONFIRMED",
    "READY_FOR_CLEARANCE",
    "CLOSED",
    "CANCELLED"
  ],
  
  "initial_state": "NEW",
  "final_states": ["CLOSED", "CANCELLED"],
  
  "transitions": {
    "NEW": ["WAITING_CLIENT_INFO", "CANCELLED"],
    "WAITING_CLIENT_INFO": ["CLIENT_INFO_COLLECTED", "CANCELLED"],
    "CLIENT_INFO_COLLECTED": ["QUOTE_READY"],
    "QUOTE_READY": ["QUOTE_APPROVAL_PENDING"],
    "QUOTE_APPROVAL_PENDING": ["QUOTE_SENT", "QUOTE_READY"],
    "QUOTE_SENT": ["WAITING_CLIENT_CONFIRMATION"],
    "WAITING_CLIENT_CONFIRMATION": ["CONFIRMED", "CANCELLED"],
    "CONFIRMED": ["DEAL_IMPORT_PENDING"],
    "...": "..."
  },
  
  "approval_map": {
    "QUOTE_APPROVAL_PENDING": {
      "approval_type": "QUOTE_APPROVAL",
      "on_approve": "QUOTE_SENT",
      "on_reject": "QUOTE_READY"
    },
    "DIMS_APPROVAL_PENDING": {
      "approval_type": "DIMS_CHANGE_APPROVAL",
      "on_approve": "DIMS_CONFIRMED",
      "on_reject": "WAITING_WAREHOUSE_DIMS"
    },
    "BL_REVIEW_PENDING": {
      "approval_type": "BL_DRAFT_APPROVAL",
      "on_approve": "BL_CONFIRMED",
      "on_reject": "BL_REVIEW_PENDING"
    }
  },
  
  "required_fields": {
    "CLIENT_INFO_COLLECTED": [
      "payload.client.name",
      "payload.route.origin_warehouse",
      "payload.cargo.ready_date",
      "payload.cargo.packages_count",
      "payload.cargo.stackable",
      "payload.cargo.dangerous_goods",
      "payload.broker.owner"
    ],
    "QUOTE_READY": [
      "computed.quote.total_usd"
    ]
  },
  
  "nba_rules": {
    "QUOTE_APPROVAL_PENDING": {
      "action_code": "APPROVE_QUOTE",
      "title": "Підтвердити калькуляцію",
      "approval_required": true
    },
    "WAITING_CLIENT_INFO": {
      "action_code": "COLLECT_CLIENT_INFO",
      "title": "Зібрати дані від клієнта",
      "approval_required": false
    }
  }
}
```

---

## 9. Minimal PoC Data Model

Для швидкого старту достатньо **4 таблиці**:

```
cases          -- кейси + state machine
case_events    -- audit log
approvals      -- human decisions
documents      -- file metadata
```

**Можна відкласти:**
- `ai_runs` — логувати в case_events.metadata
- `integrations` — зберігати в cases.payload тимчасово
- `case_type_configs` — hardcode в n8n спочатку

---

## 10. Closing Notes

Цей документ є **контрактом** для реалізації IMCP:

- **State vs Status** — зафіксована термінологія
- **Event Taxonomy** — канонічні типи подій
- **DB Invariants** — constraints та indexes
- **JSONB Contracts** — схеми та ownership
- **RLS Model** — org/team/roles

---

**Наступний документ:** [03_approval_pattern.md](./03_approval_pattern.md) — детальний опис патерну підтверджень.

**Пов'язана документація:**  
- [/docs/case_templates/](../case_templates/) — приклади конкретних кейсів
