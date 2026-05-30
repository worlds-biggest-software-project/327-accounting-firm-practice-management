# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Accounting Firm Practice Management · Created: 2025-05-25

## Philosophy

This model keeps core operational entities relational — firms, staff, clients, engagements — while collapsing variable-structure data into JSONB columns. Workflow steps live as a JSONB array on the engagement rather than in a separate table. Engagement letters, e-signatures, billing details, and AI recommendations are embedded as JSONB on the records they describe. Independence tracking and tax deadlines remain relational because they require cross-client querying and referential integrity.

The hybrid approach dramatically reduces table count (9 vs. 27) while preserving the ability to query structured data via PostgreSQL's JSONB containment operators. Adding a new field to an engagement letter or a new step type to a workflow template requires no schema migration — just a code change. This makes the platform fast to iterate during early development and flexible across different firm configurations.

The trade-off is that JSONB columns must be carefully indexed and validated at the application layer, and complex cross-entity queries (e.g., "all staff who signed off on engagements for clients with independence issues") require JSONB path expressions rather than simple JOINs.

**Best for:** Startups and solo-to-small firm deployments where rapid iteration, schema flexibility, and minimal migration overhead matter more than full relational integrity.

**Trade-offs:**
- (+) 9 tables — simple to understand, deploy, and back up
- (+) Adding new workflow step types, engagement letter fields, or billing line types requires no migration
- (+) JSONB containment queries (`@>`) with GIN indexes are fast for filtered lookups
- (+) Client custom fields are naturally expressed as JSONB without an EAV pattern
- (-) No foreign-key enforcement within JSONB structures
- (-) Complex cross-entity reporting requires JSONB path expressions
- (-) JSONB arrays can grow unbounded without application-layer size checks
- (-) Independence tracking queries across nested JSONB are less ergonomic than relational JOINs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 4557 | `audit_log` table remains relational for continuous monitoring evidence; tokens encrypted at rest |
| FTC Safeguards Rule | Access controls on `staff` table; `audit_log` provides GLBA evidence |
| AICPA SSAE 18 / SSARS 21 | `engagements.sign_offs` JSONB array records sign-off chain with standard references |
| IRS MeF | `engagements.tax_return` JSONB stores MeF submission tracking per engagement |
| PCAOB Independence | `independence_records` kept relational for cross-client querying |
| SOC 2 | `audit_log` satisfies Security criteria; role-based access via `staff.role` |
| GDPR / CCPA | `firms.data_residency`; consent tracking in `clients.portal_settings` JSONB |
| CloudEvents | `audit_log` follows CloudEvents attribute naming |

---

## Firms & Staff

```sql
CREATE TABLE firms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    ein TEXT,
    efin TEXT,
    is_public_accounting BOOLEAN DEFAULT FALSE,
    settings JSONB DEFAULT '{}',
    -- settings example:
    -- {
    --   "address": {"street": "...", "city": "...", "state": "...", "zip": "..."},
    --   "default_hourly_rate_cents": 25000,
    --   "fiscal_year_end_month": 12,
    --   "offices": [{"name": "Main", "address": {...}, "is_headquarters": true}],
    --   "payment_terms_days": 30,
    --   "service_types": [
    --     {"id": "uuid", "name": "Individual Tax Prep", "category": "tax_prep",
    --      "service_level": "preparation", "default_fee_type": "fixed",
    --      "default_fee_cents": 50000, "recurrence": "annual"}
    --   ],
    --   "workflow_templates": [
    --     {"id": "uuid", "name": "1040 Tax Prep", "service_type_id": "uuid",
    --      "steps": [
    --        {"step": 1, "name": "Collect source documents", "assignee_role": "client",
    --         "is_client_task": true, "estimated_minutes": 0},
    --        {"step": 2, "name": "Prepare return", "assignee_role": "staff",
    --         "estimated_minutes": 120, "depends_on": 1},
    --        {"step": 3, "name": "Review return", "assignee_role": "manager",
    --         "requires_sign_off": true, "depends_on": 2},
    --        {"step": 4, "name": "Send for signature", "assignee_role": "admin", "depends_on": 3}
    --      ]}
    --   ],
    --   "integrations": {
    --     "quickbooks_online": {"status": "connected", "realm_id": "...", "last_sync": "..."},
    --     "gmail": {"status": "connected"},
    --     "stripe": {"status": "connected"}
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    email TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('partner', 'manager', 'senior', 'staff', 'intern', 'admin', 'bookkeeper')),
    is_active BOOLEAN DEFAULT TRUE,
    hourly_rate_cents BIGINT,
    profile JSONB DEFAULT '{}',
    -- profile example:
    -- {
    --   "title": "Senior Tax Manager",
    --   "ptin": "P01234567",
    --   "cpa_licence": {"number": "CPA-12345", "state": "CA"},
    --   "office": "San Francisco",
    --   "target_billable_hours_weekly": 32.0,
    --   "can_sign_returns": true,
    --   "can_sign_engagements": false,
    --   "phone": "415-555-0100"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, email)
);
CREATE INDEX idx_staff_firm ON staff(firm_id);
CREATE INDEX idx_staff_profile ON staff USING GIN (profile);
```

---

## Clients

```sql
CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_number TEXT,
    name TEXT NOT NULL,
    client_type TEXT NOT NULL CHECK (client_type IN (
        'individual', 'sole_proprietor', 'partnership', 'llc',
        's_corp', 'c_corp', 'non_profit', 'trust', 'estate', 'government'
    )),
    ein TEXT,
    ssn_last_four TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    health_score NUMERIC(5,2),
    health_score_updated_at TIMESTAMPTZ,
    assigned_partner_id UUID REFERENCES staff(id),
    assigned_manager_id UUID REFERENCES staff(id),
    tags TEXT[] DEFAULT '{}',
    contacts JSONB DEFAULT '[]',
    -- contacts example:
    -- [
    --   {"id": "uuid", "name": "Jane Smith", "email": "jane@example.com",
    --    "phone": "555-0100", "role": "owner", "is_primary": true,
    --    "is_portal_user": true, "portal_last_login": "2026-05-20T14:30:00Z"},
    --   {"id": "uuid", "name": "Bob Smith", "email": "bob@example.com",
    --    "role": "spouse", "is_portal_user": false}
    -- ]
    portal_settings JSONB DEFAULT '{}',
    -- portal_settings example:
    -- {
    --   "enabled": true,
    --   "notification_preferences": {"email": true, "sms": false},
    --   "consent_given_at": "2026-01-15T10:00:00Z",
    --   "consent_type": "engagement_letter"
    -- }
    profile JSONB DEFAULT '{}',
    -- profile example:
    -- {
    --   "address": {"street": "...", "city": "...", "state": "CA", "zip": "94105"},
    --   "fiscal_year_end_month": 12,
    --   "state_of_formation": "CA",
    --   "industry": "Technology",
    --   "referral_source": "Partner referral",
    --   "custom_fields": {"preferred_language": "Spanish", "tax_software": "Drake"},
    --   "notes": "Multi-state filer: CA, NY, WA"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clients_firm ON clients(firm_id);
CREATE INDEX idx_clients_type ON clients(firm_id, client_type);
CREATE INDEX idx_clients_tags ON clients USING GIN (tags);
CREATE INDEX idx_clients_contacts ON clients USING GIN (contacts);
CREATE INDEX idx_clients_active ON clients(firm_id, is_active);
```

---

## Engagements

```sql
CREATE TABLE engagements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    name TEXT NOT NULL,
    service_category TEXT NOT NULL CHECK (service_category IN (
        'tax_prep', 'bookkeeping', 'payroll', 'audit', 'review',
        'compilation', 'advisory', 'consulting', 'other'
    )),
    service_level TEXT CHECK (service_level IN ('preparation', 'compilation', 'review', 'audit', 'attestation')),
    status TEXT NOT NULL CHECK (status IN (
        'planned', 'pending_engagement_letter', 'in_progress',
        'in_review', 'awaiting_client', 'completed', 'cancelled'
    )) DEFAULT 'planned',
    period_label TEXT,
    period_start DATE,
    period_end DATE,
    due_date DATE,
    extended_due_date DATE,
    fee_type TEXT CHECK (fee_type IN ('fixed', 'hourly', 'value_based')),
    fee_amount_cents BIGINT,
    budget_hours NUMERIC(7,2),
    actual_hours NUMERIC(7,2) DEFAULT 0,
    assigned_partner_id UUID REFERENCES staff(id),
    assigned_manager_id UUID REFERENCES staff(id),
    assigned_preparer_id UUID REFERENCES staff(id),
    is_recurring BOOLEAN DEFAULT FALSE,
    recurrence_rule TEXT,
    parent_engagement_id UUID REFERENCES engagements(id),
    priority INT CHECK (priority BETWEEN 1 AND 5) DEFAULT 3,
    tags TEXT[] DEFAULT '{}',
    tasks JSONB DEFAULT '[]',
    -- tasks example:
    -- [
    --   {"id": "uuid", "step": 1, "name": "Collect source documents",
    --    "status": "completed", "assigned_to": "uuid", "is_client_task": true,
    --    "assigned_client_contact_id": "uuid", "due_date": "2026-02-15",
    --    "completed_at": "2026-02-10T09:00:00Z", "completed_by": "uuid"},
    --   {"id": "uuid", "step": 2, "name": "Prepare return",
    --    "status": "in_progress", "assigned_to": "uuid",
    --    "estimated_minutes": 120, "due_date": "2026-03-01"},
    --   {"id": "uuid", "step": 3, "name": "Manager review",
    --    "status": "not_started", "assigned_to": "uuid",
    --    "requires_sign_off": true, "depends_on": 2}
    -- ]
    engagement_letter JSONB,
    -- engagement_letter example:
    -- {
    --   "id": "uuid", "version": 1, "status": "signed",
    --   "service_scope": "Preparation of 2025 Form 1040...",
    --   "fee_description": "Fixed fee of $500",
    --   "fee_amount_cents": 50000,
    --   "service_level": "preparation",
    --   "ai_generated": true,
    --   "sent_at": "2026-01-10T10:00:00Z",
    --   "document_id": "uuid",
    --   "signatures": [
    --     {"signer_name": "Jane Smith", "signer_email": "jane@example.com",
    --      "signer_type": "client", "status": "signed",
    --      "signed_at": "2026-01-11T14:30:00Z", "ip_address": "203.0.113.1",
    --      "signature_hash": "sha256:abc123..."},
    --     {"signer_name": "John Partner", "signer_type": "firm_partner",
    --      "status": "signed", "signed_at": "2026-01-10T16:00:00Z"}
    --   ]
    -- }
    sign_offs JSONB DEFAULT '[]',
    -- sign_offs example (SSAE 18 / SSARS 21 compliance):
    -- [
    --   {"type": "preparation_complete", "standard_reference": "SSARS 21 AR-C 70",
    --    "signed_by": "uuid", "signed_at": "2026-03-15T11:00:00Z",
    --    "conclusion": "Return prepared in accordance with applicable standards"},
    --   {"type": "partner_approval", "signed_by": "uuid",
    --    "signed_at": "2026-03-16T09:00:00Z"}
    -- ]
    tax_return JSONB,
    -- tax_return example:
    -- {
    --   "form_type": "1040", "tax_year": 2025, "jurisdiction": "federal",
    --   "status": "accepted",
    --   "mef_submission_id": "20260315-ABC-123456",
    --   "mef_ack_status": "accepted",
    --   "mef_ack_timestamp": "2026-03-15T22:00:00Z",
    --   "preparer_id": "uuid", "reviewer_id": "uuid", "signer_id": "uuid",
    --   "filed_at": "2026-03-15T20:00:00Z"
    -- }
    billing JSONB DEFAULT '{}',
    -- billing example:
    -- {
    --   "ai_write_up_cents": 5000,
    --   "ai_write_down_cents": 0,
    --   "ai_billing_notes": "Time tracked 2.1h vs 2.0h budget; recommend billing at fixed fee",
    --   "invoice_id": "uuid",
    --   "invoiced_at": "2026-04-01T10:00:00Z"
    -- }
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_engagements_firm ON engagements(firm_id);
CREATE INDEX idx_engagements_client ON engagements(client_id);
CREATE INDEX idx_engagements_status ON engagements(firm_id, status);
CREATE INDEX idx_engagements_due ON engagements(firm_id, due_date);
CREATE INDEX idx_engagements_tags ON engagements USING GIN (tags);
CREATE INDEX idx_engagements_tasks ON engagements USING GIN (tasks);
CREATE INDEX idx_engagements_sign_offs ON engagements USING GIN (sign_offs);
```

---

## Time Entries

```sql
CREATE TABLE time_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    staff_id UUID NOT NULL REFERENCES staff(id),
    engagement_id UUID REFERENCES engagements(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    date DATE NOT NULL,
    duration_minutes INT NOT NULL CHECK (duration_minutes > 0),
    description TEXT,
    is_billable BOOLEAN DEFAULT TRUE,
    source TEXT NOT NULL CHECK (source IN ('manual', 'timer', 'ai_draft', 'calendar_import')) DEFAULT 'manual',
    rate_cents BIGINT,
    amount_cents BIGINT,
    status TEXT NOT NULL CHECK (status IN ('draft', 'approved', 'invoiced', 'written_off')) DEFAULT 'draft',
    ai_adjustment JSONB,
    -- ai_adjustment example:
    -- {
    --   "suggested_minutes": 90,
    --   "suggested_amount_cents": 37500,
    --   "adjustment_type": "write_down",
    --   "reason": "Similar returns averaged 1.5h; 2h tracked may include non-billable research",
    --   "accepted": true
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_time_entries_firm_date ON time_entries(firm_id, date);
CREATE INDEX idx_time_entries_staff ON time_entries(staff_id, date);
CREATE INDEX idx_time_entries_engagement ON time_entries(engagement_id);
CREATE INDEX idx_time_entries_status ON time_entries(firm_id, status);
```

---

## Invoices & Payments

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    invoice_number TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN (
        'draft', 'ai_drafted', 'pending_review', 'sent', 'viewed',
        'partially_paid', 'paid', 'overdue', 'written_off', 'void'
    )) DEFAULT 'draft',
    issue_date DATE,
    due_date DATE,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT DEFAULT 0,
    balance_due_cents BIGINT NOT NULL DEFAULT 0,
    line_items JSONB NOT NULL DEFAULT '[]',
    -- line_items example:
    -- [
    --   {"engagement_id": "uuid", "description": "2025 Individual Tax Preparation",
    --    "line_type": "fixed_fee", "quantity": 1, "unit_price_cents": 50000,
    --    "amount_cents": 50000},
    --   {"engagement_id": "uuid", "description": "State return - CA",
    --    "line_type": "fixed_fee", "quantity": 1, "unit_price_cents": 15000,
    --    "amount_cents": 15000},
    --   {"description": "Write-down: complexity below estimate",
    --    "line_type": "write_down", "amount_cents": -5000}
    -- ]
    ai_analysis JSONB,
    -- ai_analysis example:
    -- {
    --   "write_up_cents": 0, "write_down_cents": 5000,
    --   "notes": "Actual time 1.5h vs 2.0h budget; recommend $50 write-down",
    --   "comparable_engagements": ["uuid1", "uuid2"],
    --   "generated_at": "2026-04-01T08:00:00Z"
    -- }
    payments JSONB DEFAULT '[]',
    -- payments example:
    -- [
    --   {"id": "uuid", "amount_cents": 60000, "method": "credit_card",
    --    "date": "2026-04-15", "stripe_payment_id": "pi_abc123",
    --    "reference": "Online payment"}
    -- ]
    stripe_invoice_id TEXT,
    notes TEXT,
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, invoice_number)
);
CREATE INDEX idx_invoices_firm ON invoices(firm_id);
CREATE INDEX idx_invoices_client ON invoices(client_id);
CREATE INDEX idx_invoices_status ON invoices(firm_id, status);
CREATE INDEX idx_invoices_line_items ON invoices USING GIN (line_items);
```

---

## Documents & Email

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID REFERENCES clients(id),
    engagement_id UUID REFERENCES engagements(id),
    name TEXT NOT NULL,
    file_path TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type TEXT,
    category TEXT CHECK (category IN (
        'engagement_letter', 'tax_return', 'financial_statement',
        'source_document', 'workpaper', 'correspondence',
        'form_8879', 'k1', 'w2', 'w9', '1099', 'other'
    )),
    tax_year INT,
    version INT DEFAULT 1,
    parent_document_id UUID REFERENCES documents(id),
    is_client_visible BOOLEAN DEFAULT FALSE,
    uploaded_by_type TEXT CHECK (uploaded_by_type IN ('staff', 'client')),
    uploaded_by_id UUID,
    checksum_sha256 TEXT,
    e_signatures JSONB DEFAULT '[]',
    -- e_signatures example:
    -- [
    --   {"signer_name": "Jane Smith", "signer_email": "jane@example.com",
    --    "signer_type": "client", "form_type": "8879",
    --    "status": "signed", "signed_at": "2026-03-10T14:00:00Z",
    --    "ip_address": "203.0.113.1", "signature_hash": "sha256:..."}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_documents_firm ON documents(firm_id);
CREATE INDEX idx_documents_client ON documents(client_id);
CREATE INDEX idx_documents_engagement ON documents(engagement_id);
CREATE INDEX idx_documents_category ON documents(firm_id, category);
CREATE INDEX idx_documents_tax_year ON documents(client_id, tax_year);
```

---

## Independence & Compliance

```sql
CREATE TABLE independence_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    staff_id UUID REFERENCES staff(id),
    record_type TEXT NOT NULL CHECK (record_type IN (
        'financial_interest', 'employment_relationship', 'family_relationship',
        'business_relationship', 'fee_dependency', 'non_audit_service', 'other'
    )),
    description TEXT NOT NULL,
    threat_level TEXT CHECK (threat_level IN ('none', 'low', 'moderate', 'significant', 'prohibited')),
    safeguards_applied TEXT,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_active BOOLEAN DEFAULT TRUE,
    conflict_checks JSONB DEFAULT '[]',
    -- conflict_checks example:
    -- [
    --   {"check_date": "2026-01-05", "checked_by": "uuid",
    --    "result": "clear", "notes": "No conflicts identified"},
    --   {"check_date": "2026-06-01", "checked_by": "uuid",
    --    "result": "requires_review",
    --    "conflicts_found": ["Staff member's spouse employed by client"],
    --    "resolution": "Removed staff from engagement team",
    --    "approved_by": "uuid", "approved_at": "2026-06-02T10:00:00Z"}
    -- ]
    reviewed_by_id UUID REFERENCES staff(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_independence_firm ON independence_records(firm_id);
CREATE INDEX idx_independence_client ON independence_records(client_id);
CREATE INDEX idx_independence_active ON independence_records(firm_id, is_active);
CREATE INDEX idx_independence_checks ON independence_records USING GIN (conflict_checks);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    ce_source TEXT NOT NULL,
    ce_type TEXT NOT NULL,
    ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    actor_type TEXT NOT NULL CHECK (actor_type IN ('staff', 'client', 'system', 'integration')),
    actor_id UUID,
    actor_ip INET,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    action TEXT NOT NULL,
    detail JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_audit_firm_time ON audit_log(firm_id, ce_time);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, ce_time);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_type ON audit_log(ce_type);
```

---

## Example Queries

### Find all engagements with unsigned engagement letters

```sql
SELECT e.id, e.name, c.name AS client_name, e.engagement_letter->>'status' AS letter_status
FROM engagements e
JOIN clients c ON c.id = e.client_id
WHERE e.firm_id = $1
  AND e.engagement_letter IS NOT NULL
  AND e.engagement_letter->>'status' NOT IN ('signed')
  AND e.status = 'pending_engagement_letter'
ORDER BY e.due_date;
```

### Find all overdue client tasks across engagements

```sql
SELECT e.id AS engagement_id, e.name, c.name AS client_name,
       task->>'name' AS task_name, task->>'due_date' AS due_date
FROM engagements e
JOIN clients c ON c.id = e.client_id,
     jsonb_array_elements(e.tasks) AS task
WHERE e.firm_id = $1
  AND task->>'is_client_task' = 'true'
  AND task->>'status' NOT IN ('completed', 'skipped')
  AND (task->>'due_date')::date < CURRENT_DATE
ORDER BY (task->>'due_date')::date;
```

### Staff utilisation for the current week

```sql
SELECT s.full_name, s.role,
       COALESCE(SUM(te.duration_minutes), 0) / 60.0 AS hours_logged,
       (s.profile->>'target_billable_hours_weekly')::numeric AS target_hours
FROM staff s
LEFT JOIN time_entries te ON te.staff_id = s.id
  AND te.date >= date_trunc('week', CURRENT_DATE)
  AND te.date < date_trunc('week', CURRENT_DATE) + interval '7 days'
  AND te.is_billable = TRUE
WHERE s.firm_id = $1 AND s.is_active = TRUE
GROUP BY s.id, s.full_name, s.role, s.profile
ORDER BY hours_logged DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Firm & Staff | 2 | firms (offices/service_types/templates in JSONB), staff (profile in JSONB) |
| Client Management | 1 | clients (contacts/portal_settings/custom_fields in JSONB) |
| Engagements | 1 | engagements (tasks/engagement_letter/sign_offs/tax_return/billing in JSONB) |
| Time Tracking | 1 | time_entries (AI adjustment in JSONB) |
| Invoices | 1 | invoices (line_items/payments/AI analysis in JSONB) |
| Documents | 1 | documents (e_signatures in JSONB) |
| Independence | 1 | independence_records (conflict_checks in JSONB) |
| Audit | 1 | audit_log (partitioned, CloudEvents) |
| **Total** | **9** | |

---

## Key Design Decisions

1. **Workflow templates on the firm, task instances on the engagement** — `firms.settings.workflow_templates` stores the process definitions; `engagements.tasks` stores the runtime instances. Template edits cascade to future engagements by regenerating the tasks array from the template at engagement creation time.

2. **Engagement letter and e-signatures embedded on the engagement** — an engagement letter exists in the context of one engagement, so it lives as a JSONB column rather than a separate table. Signatures are nested within the letter object, keeping the complete signature audit trail in one place.

3. **Sign-offs as a JSONB array on engagements** — AICPA SSARS/SSAE sign-off chains are append-only and always queried in the context of a specific engagement. The JSONB array preserves insertion order and includes standard references.

4. **Independence records kept relational** — unlike most entities that collapse into JSONB, independence tracking requires cross-client queries ("does any staff member have a conflict with any audit client?") that are impractical to express across nested JSONB. The `conflict_checks` JSONB on each record captures the check history per relationship.

5. **Tax return tracking as engagement JSONB** — each engagement produces at most one tax return, making a separate table unnecessary. MeF submission tracking (submission_id, ack_status, rejection_code) lives inline.

6. **Payments embedded on invoices** — payments always belong to a specific invoice. The JSONB array keeps payment history alongside the invoice for simple ledger-style views. Cross-invoice payment reporting uses `jsonb_array_elements` with aggregation.

7. **Audit log remains fully relational and partitioned** — this is the one area where JSONB embedding would be inappropriate. The audit log must support time-range queries across all entities for IRS 4557 and FTC Safeguards compliance. Partitioning by time enables efficient retention management.

8. **Client contacts as JSONB array** — contacts are always accessed in the context of their client. Portal user status and last-login tracking are embedded per contact. The GIN index on `contacts` enables containment queries for finding clients by contact email.

9. **AI analysis separated from business data** — `time_entries.ai_adjustment` and `invoices.ai_analysis` keep AI recommendations alongside but structurally separate from the authoritative business values. The `accepted` flag on AI adjustments tracks whether the recommendation was adopted.
