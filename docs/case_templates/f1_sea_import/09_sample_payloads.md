# F1 Sea Import — Sample Payloads

**Case Type:** `F1_SEA_IMPORT`  
**Пов'язані Core документи:**  
- [02_core_data_model.md](../../core/02_core_data_model.md) — Table schemas, JSONB Contracts

Цей документ містить приклади даних для PoC, узгоджені з Core Data Model.

---

## 1) Case Record (Supabase `cases` table)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "case_type": "F1_SEA_IMPORT",
  "case_number": "F1-SEA-2026-00123",
  "title": "ACME Ltd - Yiwu to Kyiv",
  
  "state": "QUOTE_APPROVAL_PENDING",
  "status": "OPEN",
  // Note: substates planned for P2, using computed.tasks[] for PoC
  
  "org_id": "org-001",
  "team_id": "team-logistics",
  "owner_user_id": "user-ivan",
  
  "priority": 1,
  "sla_deadline": "2026-01-21T12:00:00Z",
  
  "payload": {
    "client": {
      "name": "ACME Ltd",
      "contact_name": "John Smith",
      "contact_email": "john@acme.com",
      "contact_phone": "+380501234567"
    },
    "route": {
      "mode": "SEA",
      "incoterms": "FOB",
      "origin_country": "CN",
      "origin_warehouse": "YIWU",
      "destination_city": "Kyiv",
      "destination_terminal": "Terminal X",
      "unloading_point": "Kyiv, client warehouse"
    },
    "cargo": {
      "ready_date": "2026-02-10",
      "packages_count": 10,
      "dimensions": [
        {"length_cm": 60, "width_cm": 40, "height_cm": 50, "quantity": 10}
      ],
      "total_cbm": 1.2,
      "total_weight_kg": 350,
      "stackable": true,
      "dangerous_goods": false,
      "description": "Consumer electronics"
    },
    "broker": {
      "owner": "OUR",
      "export_declaration_required": "UNKNOWN"
    },
    "insurance": {
      "required": false
    },
    "integration": {
      "onec_request_id": "REQ-2026-00456",
      "onec_deal_number": null,
      "marking_code": null
    }
  },
  
  "computed": {
    "quote": {
      "version": 1,
      "calculated_at": "2026-01-14T10:30:00Z",
      "base_cost_usd": 1550,
      "margin_usd": 250,
      "total_usd": 1800,
      "validity_days": 7,
      "valid_until": "2026-01-21T10:30:00Z",
      "assumptions": ["stackable cargo", "no dangerous goods"],
      "risks": ["dimensions not verified by warehouse"]
    },
    "risks": [
      {
        "code": "DIMS_NOT_VERIFIED",
        "severity": "LOW",
        "message": "Warehouse has not verified dimensions yet",
        "detected_at": "2026-01-14T10:30:00Z",
        "detected_by": "SYSTEM"
      }
    ],
    "nba": {
      "action_code": "APPROVE_QUOTE",
      "title": "Підтвердити калькуляцію",
      "description": "Система підготувала розрахунок ціни. Перевірте та підтвердіть.",
      "approval_required": true,
      "approval_id": "appr-001"
    },
    "tasks": [
      {
        "code": "QUOTE_FOLLOWUP",
        "title": "Перевірити актуальність ціни",
        "status": "PENDING",
        "created_at": "2026-01-14T10:30:00Z"
      }
    ],
    "sla": {
      "deadline": "2026-01-21T12:00:00Z",
      "status": "OK",
      "hours_remaining": 168
    }
  },
  
  "created_at": "2026-01-14T09:00:00Z",
  "updated_at": "2026-01-14T10:35:00Z",
  "closed_at": null
}
```

---

## 2) Approval Record (Supabase `approvals` table)

### Example: QUOTE_APPROVAL (pending)

```json
{
  "id": "appr-001",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "approval_type": "QUOTE_APPROVAL",
  "status": "PENDING",
  
  "requested_by": "SYSTEM",
  "requested_at": "2026-01-14T10:30:00Z",
  
  "request_snapshot": {
    "quote_version": 1,
    "cost_breakdown": {
      "sea_freight_usd": 1200,
      "warehouse_handling_usd": 200,
      "customs_usd": 150
    },
    "margin_usd": 250,
    "total_usd": 1800,
    "validity_days": 7,
    "assumptions": ["stackable cargo", "no dangerous goods"],
    "risks": ["dimensions not verified by warehouse"]
  },
  
  "decided_by": null,
  "decided_at": null,
  "decision_snapshot": null,
  "decision_comment": null,
  
  "idempotency_key": "QUOTE_APPROVAL:550e8400:v1",
  
  "created_at": "2026-01-14T10:30:00Z",
  "updated_at": "2026-01-14T10:30:00Z"
}
```

### Example: QUOTE_APPROVAL (approved with edits)

```json
{
  "id": "appr-001",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "approval_type": "QUOTE_APPROVAL",
  "status": "APPROVED",
  
  "requested_by": "SYSTEM",
  "requested_at": "2026-01-14T10:30:00Z",
  
  "request_snapshot": {
    "quote_version": 1,
    "total_usd": 1800
  },
  
  "decided_by": "user-ivan",
  "decided_at": "2026-01-14T11:15:00Z",
  "decision_snapshot": {
    "quote_version": 1,
    "total_usd": 1750,
    "edited_fields": ["margin_usd"],
    "margin_usd": 200
  },
  "decision_comment": "Reduced margin for repeat customer",
  
  "idempotency_key": "QUOTE_APPROVAL:550e8400:v1",
  
  "created_at": "2026-01-14T10:30:00Z",
  "updated_at": "2026-01-14T11:15:00Z"
}
```

---

## 3) Case Event Record (Supabase `case_events` table)

### Example: STATE_CHANGED

```json
{
  "id": "evt-001",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "event_type": "STATE_CHANGED",
  "actor_type": "SYSTEM",
  "actor_id": "workflow:f1_calculate_quote",
  
  "metadata": {
    "from_state": "REQUEST_1C_CREATED",
    "to_state": "QUOTE_READY",
    "reason": "quote_calculation_completed"
  },
  
  "idempotency_key": "STATE:550e8400:QUOTE_READY:1",
  "created_at": "2026-01-14T10:25:00Z"
}
```

### Example: APPROVAL_APPROVED

```json
{
  "id": "evt-002",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "event_type": "APPROVAL_APPROVED",
  "actor_type": "HUMAN",
  "actor_id": "user-ivan",
  
  "metadata": {
    "approval_id": "appr-001",
    "approval_type": "QUOTE_APPROVAL",
    "has_edits": true
  },
  
  "idempotency_key": null,
  "created_at": "2026-01-14T11:15:00Z"
}
```

### Example: DOC_UPLOADED

```json
{
  "id": "evt-003",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "event_type": "DOC_UPLOADED",
  "actor_type": "INTEGRATION",
  "actor_id": "webhook:zed_bl_upload",
  
  "metadata": {
    "document_id": "doc-456",
    "doc_type": "BL_DRAFT",
    "source": "ZED"
  },
  
  "idempotency_key": "DOC:550e8400:BL_DRAFT:v1",
  "created_at": "2026-01-15T14:00:00Z"
}
```

### Example: INTEGRATION_SUCCESS

```json
{
  "id": "evt-004",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "event_type": "INTEGRATION_SUCCESS",
  "actor_type": "INTEGRATION",
  "actor_id": "n8n:1c_node",
  
  "metadata": {
    "integration_id": "int-001",
    "integration_type": "ONEC",
    "action": "create_deal",
    "external_id": "IMP-2026-00789"
  },
  
  "idempotency_key": "INT:550e8400:ONEC:create_deal",
  "created_at": "2026-01-14T12:00:00Z"
}
```

---

## 4) Document Record (Supabase `documents` table)

```json
{
  "id": "doc-456",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "doc_type": "BL_DRAFT",
  "version": 1,
  
  "storage_path": "cases/550e8400/docs/bl_draft_v1.pdf",
  "file_name": "BL_Draft_ACME_2026.pdf",
  "mime_type": "application/pdf",
  "size_bytes": 245678,
  "checksum": "sha256:abc123...",
  
  "status": "VERIFIED",
  "source": "ZED",
  
  "uploaded_by": "webhook:zed_upload",
  "uploaded_at": "2026-01-15T14:00:00Z",
  
  "extracted_data": {
    "shipper": "China Supplier Ltd",
    "consignee": "ACME Ltd",
    "cargo_description": "Consumer electronics",
    "packages": 10,
    "gross_weight_kg": 350
  },
  "extraction_confidence": 0.92,
  
  "metadata": {
    "verified_by": "user-ivan",
    "verified_at": "2026-01-15T15:30:00Z"
  }
}
```

---

## 5) AI Run Record (Supabase `ai_runs` table)

```json
{
  "id": "ai-001",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "run_type": "EXTRACT",
  "model": "gpt-4",
  "prompt_version": "intake_extract_v2",
  
  "input_refs": {
    "raw_text": "client_message_123",
    "case_id": "550e8400-e29b-41d4-a716-446655440000"
  },
  
  "output": {
    "incoterms": "FOB",
    "origin_warehouse": "YIWU",
    "cargo_ready_date": "2026-02-10",
    "packages_count": 10,
    "stackable": true,
    "dangerous_goods": false
  },
  
  "confidence": 0.89,
  "flags": [],
  
  "duration_ms": 2340,
  "tokens_used": 1250,
  
  "created_at": "2026-01-14T09:30:00Z"
}
```

### AI Run: VERIFY (BL Draft)

```json
{
  "id": "ai-002",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "run_type": "VERIFY",
  "model": "gpt-4",
  "prompt_version": "bl_verify_v1",
  
  "input_refs": {
    "bl_document_id": "doc-456",
    "contract_document_id": "doc-123",
    "invoice_document_id": "doc-124"
  },
  
  "output": {
    "overall_match": false,
    "issues": [
      {
        "field": "consignee",
        "expected": "ACME Ltd",
        "actual": "ACME LTD",
        "severity": "LOW",
        "type": "case_mismatch"
      }
    ],
    "verified_fields": ["shipper", "cargo_description", "packages"]
  },
  
  "confidence": 0.85,
  "flags": ["NEEDS_REVIEW"],
  
  "duration_ms": 3120,
  "tokens_used": 2100,
  
  "created_at": "2026-01-15T14:30:00Z"
}
```

---

## 6) Integration Record (Supabase `integrations` table)

```json
{
  "id": "int-001",
  "case_id": "550e8400-e29b-41d4-a716-446655440000",
  
  "integration_type": "ONEC",
  "external_id": "IMP-2026-00789",
  
  "status": "SYNCED",
  "last_error": null,
  "retry_count": 0,
  
  "request_payload": {
    "operation": "create_deal",
    "deal_type": "import",
    "client_name": "ACME Ltd",
    "incoterms": "FOB",
    "from_request_id": "REQ-2026-00456"
  },
  
  "response_payload": {
    "deal_number": "IMP-2026-00789",
    "created_at": "2026-01-14T12:00:00Z",
    "status": "active"
  },
  
  "correlation_key": "CREATE_1C_DEAL:550e8400",
  
  "created_at": "2026-01-14T11:59:00Z",
  "updated_at": "2026-01-14T12:00:05Z",
  "last_sync_at": "2026-01-14T12:00:05Z"
}
```

---

## 7) Payload Schema Summary

### cases.payload (F1_SEA_IMPORT)

```typescript
interface F1SeaImportPayload {
  client: {
    name: string;
    contact_name?: string;
    contact_email?: string;
    contact_phone?: string;
  };
  
  route: {
    mode: 'SEA';
    incoterms: string;
    origin_country: string;
    origin_warehouse: 'YIWU' | 'SHENZHEN' | 'OTHER';
    destination_city: string;
    destination_terminal?: string;
    unloading_point?: string;
  };
  
  cargo: {
    ready_date: string;  // ISO date
    packages_count: number;
    dimensions: Array<{
      length_cm: number;
      width_cm: number;
      height_cm: number;
      quantity: number;
    }>;
    total_cbm?: number;
    total_weight_kg?: number;
    stackable: boolean;
    dangerous_goods: boolean;
    dangerous_class?: string;
    description: string;
    
    // Warehouse-verified (n8n writes)
    warehouse_packages_count?: number;
    warehouse_dimensions?: Array<{...}>;
    warehouse_weight_kg?: number;
    warehouse_stackable?: boolean;
    warehouse_verified_at?: string;
    final_dimensions?: Array<{...}>;  // After approval
  };
  
  broker: {
    owner: 'OUR' | 'CLIENT';
    name?: string;
    contact?: string;
    export_declaration_required: 'YES' | 'NO' | 'UNKNOWN';
  };
  
  insurance?: {
    required: boolean;
    type?: 'STANDARD' | 'FULL';
    value_usd?: number;
  };
  
  integration: {
    onec_request_id?: string;
    onec_deal_number?: string;
    marking_code?: string;  // ALEX-<deal_number>
    vp_table_row_id?: string;
  };
}
```

### cases.computed (F1_SEA_IMPORT)

```typescript
interface F1SeaImportComputed {
  quote?: {
    version: number;
    calculated_at: string;
    base_cost_usd: number;
    margin_usd: number;
    total_usd: number;
    validity_days: number;
    valid_until: string;
    assumptions: string[];
    risks: string[];
  };
  
  risks?: Array<{
    code: string;
    severity: 'LOW' | 'MEDIUM' | 'HIGH';
    message: string;
    detected_at: string;
    detected_by: 'AI' | 'SYSTEM' | 'HUMAN';
  }>;
  
  nba?: {
    action_code: string;
    title: string;
    description: string;
    approval_required: boolean;
    approval_id?: string;
  };
  
  sla?: {
    deadline: string;
    status: 'OK' | 'AT_RISK' | 'OVERDUE';
    hours_remaining?: number;
  };
  
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
