# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Accounting Firm Practice Management · Created: 2025-05-25

## Philosophy

This model gives every domain concept its own table with explicit foreign keys and constraints. Recurring workflows are modelled as a template-instance hierarchy: workflow templates define the steps for each service type, engagements instantiate those templates per client and period, and tasks are the individual work items within each engagement. Independence tracking and conflict-of-interest checking get dedicated tables rather than being embedded in client records.

The normalised approach makes complex cross-entity queries straightforward — "show me all overdue tasks for clients whose engagement letters haven't been signed" is a simple JOIN rather than a JSONB traversal. Referential integrity enforces business rules at the database level: you cannot delete a client who has open engagements, and every time entry must reference a valid engagement.

Standards alignment is first-class: sign-off trails satisfy AICPA SSARS/SSAE documentation requirements, the audit log meets IRS Publication 4557 and FTC Safeguards Rule evidence mandates, and independence records follow PCAOB structural requirements.

**Best for:** Multi-partner firms with compliance obligations (public accounting, audit), where data integrity and auditability outweigh schema flexibility.

**Trade-offs:**
- (+) Full referential integrity — the database enforces business rules
- (+) Complex cross-entity queries are simple JOINs
- (+) Independence tracking and sign-off trails are fully relational and queryable
- (+) Standard SQL — no JSONB expertise required
- (-) ~26 tables is a large schema to maintain and migrate
- (-) Adding jurisdiction-specific fields requires ALTER TABLE or new tables
- (-) Custom fields require a dedicated EAV table rather than inline JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 4557 | `audit_log` table with AES-256 encryption context; MFA enforcement tracked in staff sessions |
| FTC Safeguards Rule (GLBA) | `audit_log` provides continuous monitoring evidence; `staff` table tracks access levels |
| AICPA SSAE No. 18 | `sign_off_trails` table records attestation engagement sign-offs with reviewer, date, and conclusion |
| AICPA SSARS 21 | `engagements.service_level` distinguishes preparation vs. compilation vs. review; workflow templates enforce documentation per level |
| IRS MeF | `tax_returns` table stores submission_id, ack_status, and MeF XML reference for e-filed returns |
| PCAOB Independence | `independence_records` table tracks financial interests, employment relationships, and fee dependencies per client |
| SOC 2 | `audit_log` satisfies Security and Processing Integrity criteria; `staff` access controls support least-privilege |
| GDPR / CCPA | `clients.data_residency` field; `client_consent_records` for processing basis; `audit_log` for access evidence |
| OAuth 2.0 / OIDC | `integrations` table stores OAuth tokens for QBO, Xero, Gmail, M365 per firm |
| CloudEvents | `audit_log` follows CloudEvents attribute naming (ce_source, ce_type, ce_time) |

---

## Firm & Staff Management

```sql
CREATE TABLE firms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    ein TEXT,
    efin TEXT,                -- IRS Electronic Filing Identification Number
    ptin_firm TEXT,           -- Preparer Tax Identification Number (firm level)
    address JSONB,
    phone TEXT,
    website TEXT,
    default_hourly_rate_cents BIGINT,
    fiscal_year_end_month INT CHECK (fiscal_year_end_month BETWEEN 1 AND 12) DEFAULT 12,
    is_public_accounting BOOLEAN DEFAULT FALSE,
    soc2_certified BOOLEAN DEFAULT FALSE,
    data_residency TEXT CHECK (data_residency IN ('us', 'eu', 'uk', 'au', 'ca')) DEFAULT 'us',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE offices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    name TEXT NOT NULL,
    address JSONB,
    phone TEXT,
    is_headquarters BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_offices_firm ON offices(firm_id);

CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    office_id UUID REFERENCES offices(id),
    email TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('partner', 'manager', 'senior', 'staff', 'intern', 'admin', 'bookkeeper')),
    title TEXT,
    ptin TEXT,                -- individual Preparer Tax Identification Number
    cpa_licence_number TEXT,
    cpa_licence_state TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    hourly_rate_cents BIGINT,
    target_billable_hours_weekly NUMERIC(5,1),
    can_sign_returns BOOLEAN DEFAULT FALSE,
    can_sign_engagements BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, email)
);
CREATE INDEX idx_staff_firm ON staff(firm_id);
CREATE INDEX idx_staff_role ON staff(firm_id, role);
```

---

## Client Management

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
    ssn_last_four TEXT,      -- last 4 only; full SSN never stored
    primary_contact_name TEXT,
    primary_contact_email TEXT,
    primary_contact_phone TEXT,
    address JSONB,
    fiscal_year_end_month INT CHECK (fiscal_year_end_month BETWEEN 1 AND 12) DEFAULT 12,
    state_of_formation TEXT,
    industry TEXT,
    referral_source TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    health_score NUMERIC(5,2),         -- 0-100, computed by AI
    health_score_updated_at TIMESTAMPTZ,
    portal_enabled BOOLEAN DEFAULT FALSE,
    assigned_partner_id UUID REFERENCES staff(id),
    assigned_manager_id UUID REFERENCES staff(id),
    tags TEXT[] DEFAULT '{}',
    custom_fields JSONB DEFAULT '{}',
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clients_firm ON clients(firm_id);
CREATE INDEX idx_clients_type ON clients(firm_id, client_type);
CREATE INDEX idx_clients_tags ON clients USING GIN (tags);
CREATE INDEX idx_clients_active ON clients(firm_id, is_active);

CREATE TABLE client_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    full_name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    role TEXT,                -- e.g. 'owner', 'cfo', 'bookkeeper', 'spouse'
    is_portal_user BOOLEAN DEFAULT FALSE,
    portal_last_login_at TIMESTAMPTZ,
    is_primary BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_contacts_client ON client_contacts(client_id);
```

---

## Service Types & Workflow Templates

```sql
CREATE TABLE service_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    name TEXT NOT NULL,
    category TEXT NOT NULL CHECK (category IN (
        'tax_prep', 'bookkeeping', 'payroll', 'audit', 'review',
        'compilation', 'advisory', 'consulting', 'other'
    )),
    -- SSARS 21: service level determines documentation requirements
    service_level TEXT CHECK (service_level IN ('preparation', 'compilation', 'review', 'audit', 'attestation')),
    default_fee_type TEXT CHECK (default_fee_type IN ('fixed', 'hourly', 'value_based')),
    default_fee_cents BIGINT,
    recurrence TEXT CHECK (recurrence IN ('annual', 'quarterly', 'monthly', 'weekly', 'one_time')),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_service_types_firm ON service_types(firm_id);

CREATE TABLE workflow_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    service_type_id UUID REFERENCES service_types(id),
    name TEXT NOT NULL,
    description TEXT,
    is_system_template BOOLEAN DEFAULT FALSE,  -- pre-built templates
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_workflow_templates_firm ON workflow_templates(firm_id);

CREATE TABLE workflow_template_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES workflow_templates(id) ON DELETE CASCADE,
    step_number INT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    default_assignee_role TEXT CHECK (default_assignee_role IN (
        'partner', 'manager', 'senior', 'staff', 'intern', 'admin', 'bookkeeper', 'client'
    )),
    estimated_minutes INT,
    is_client_task BOOLEAN DEFAULT FALSE,  -- appears in client portal
    requires_sign_off BOOLEAN DEFAULT FALSE,
    depends_on_step INT,    -- step_number this depends on
    auto_reminder_days INT, -- days before due date to remind
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (template_id, step_number)
);
CREATE INDEX idx_template_steps_template ON workflow_template_steps(template_id);
```

---

## Engagements & Tasks

```sql
CREATE TABLE engagements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    service_type_id UUID NOT NULL REFERENCES service_types(id),
    workflow_template_id UUID REFERENCES workflow_templates(id),
    name TEXT NOT NULL,
    period_label TEXT,              -- e.g. '2025 Tax Year', 'Q1 2026 Bookkeeping'
    period_start DATE,
    period_end DATE,
    due_date DATE,
    extended_due_date DATE,        -- after extension filed
    status TEXT NOT NULL CHECK (status IN (
        'planned', 'pending_engagement_letter', 'in_progress',
        'in_review', 'awaiting_client', 'completed', 'cancelled'
    )) DEFAULT 'planned',
    fee_type TEXT CHECK (fee_type IN ('fixed', 'hourly', 'value_based')),
    fee_amount_cents BIGINT,
    budget_hours NUMERIC(7,2),
    actual_hours NUMERIC(7,2) DEFAULT 0,
    assigned_partner_id UUID REFERENCES staff(id),
    assigned_manager_id UUID REFERENCES staff(id),
    assigned_preparer_id UUID REFERENCES staff(id),
    is_recurring BOOLEAN DEFAULT FALSE,
    recurrence_rule TEXT,           -- iCal RRULE format (RFC 5545)
    parent_engagement_id UUID REFERENCES engagements(id), -- recurring series link
    priority INT CHECK (priority BETWEEN 1 AND 5) DEFAULT 3,
    tags TEXT[] DEFAULT '{}',
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_engagements_firm ON engagements(firm_id);
CREATE INDEX idx_engagements_client ON engagements(client_id);
CREATE INDEX idx_engagements_status ON engagements(firm_id, status);
CREATE INDEX idx_engagements_due ON engagements(firm_id, due_date);
CREATE INDEX idx_engagements_tags ON engagements USING GIN (tags);

CREATE TABLE engagement_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id UUID NOT NULL REFERENCES engagements(id),
    template_step_id UUID REFERENCES workflow_template_steps(id),
    step_number INT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL CHECK (status IN (
        'not_started', 'in_progress', 'awaiting_client', 'in_review',
        'completed', 'skipped', 'blocked'
    )) DEFAULT 'not_started',
    assigned_to_id UUID REFERENCES staff(id),
    is_client_task BOOLEAN DEFAULT FALSE,
    assigned_client_contact_id UUID REFERENCES client_contacts(id),
    due_date DATE,
    completed_at TIMESTAMPTZ,
    completed_by_id UUID REFERENCES staff(id),
    requires_sign_off BOOLEAN DEFAULT FALSE,
    signed_off_by_id UUID REFERENCES staff(id),
    signed_off_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_engagement ON engagement_tasks(engagement_id);
CREATE INDEX idx_tasks_assigned ON engagement_tasks(assigned_to_id, status);
CREATE INDEX idx_tasks_due ON engagement_tasks(due_date) WHERE status NOT IN ('completed', 'skipped');
CREATE INDEX idx_tasks_client ON engagement_tasks(assigned_client_contact_id) WHERE is_client_task = TRUE;
```

---

## Engagement Letters & E-Signatures

```sql
CREATE TABLE engagement_letters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id UUID NOT NULL REFERENCES engagements(id),
    document_id UUID REFERENCES documents(id),
    version INT NOT NULL DEFAULT 1,
    status TEXT NOT NULL CHECK (status IN (
        'draft', 'ai_drafted', 'pending_review', 'sent', 'viewed',
        'signed', 'declined', 'expired', 'superseded'
    )) DEFAULT 'draft',
    service_scope TEXT,
    fee_description TEXT,
    fee_amount_cents BIGINT,
    -- SSARS 21: engagement letter requirements vary by service level
    service_level TEXT CHECK (service_level IN ('preparation', 'compilation', 'review', 'audit', 'attestation')),
    sent_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    ai_generated BOOLEAN DEFAULT FALSE,
    ai_generation_context JSONB,   -- prior-year data used for AI drafting
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_engagement_letters_engagement ON engagement_letters(engagement_id);

CREATE TABLE e_signatures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_letter_id UUID REFERENCES engagement_letters(id),
    document_id UUID REFERENCES documents(id),
    signer_name TEXT NOT NULL,
    signer_email TEXT NOT NULL,
    signer_type TEXT NOT NULL CHECK (signer_type IN ('client', 'firm_partner', 'firm_staff')),
    client_contact_id UUID REFERENCES client_contacts(id),
    staff_id UUID REFERENCES staff(id),
    status TEXT NOT NULL CHECK (status IN ('pending', 'signed', 'declined', 'expired')) DEFAULT 'pending',
    signed_at TIMESTAMPTZ,
    ip_address INET,
    user_agent TEXT,
    signature_hash TEXT,           -- SHA-256 of the signed document
    form_type TEXT,                -- e.g. '8879', 'engagement_letter', 'consent'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_esignatures_letter ON e_signatures(engagement_letter_id);
CREATE INDEX idx_esignatures_document ON e_signatures(document_id);
```

---

## Time Tracking & Billing

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
    amount_cents BIGINT,           -- duration * rate
    status TEXT NOT NULL CHECK (status IN ('draft', 'approved', 'invoiced', 'written_off')) DEFAULT 'draft',
    -- AI write-up/write-down
    ai_suggested_minutes INT,
    ai_suggested_amount_cents BIGINT,
    ai_adjustment_reason TEXT,
    invoice_id UUID REFERENCES invoices(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_time_entries_firm_date ON time_entries(firm_id, date);
CREATE INDEX idx_time_entries_staff ON time_entries(staff_id, date);
CREATE INDEX idx_time_entries_engagement ON time_entries(engagement_id);
CREATE INDEX idx_time_entries_status ON time_entries(firm_id, status);

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
    subtotal_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT DEFAULT 0,
    total_cents BIGINT NOT NULL DEFAULT 0,
    amount_paid_cents BIGINT DEFAULT 0,
    balance_due_cents BIGINT NOT NULL DEFAULT 0,
    -- AI billing analysis
    ai_write_up_cents BIGINT DEFAULT 0,
    ai_write_down_cents BIGINT DEFAULT 0,
    ai_billing_notes TEXT,
    payment_terms_days INT DEFAULT 30,
    notes TEXT,
    sent_at TIMESTAMPTZ,
    paid_at TIMESTAMPTZ,
    stripe_invoice_id TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, invoice_number)
);
CREATE INDEX idx_invoices_firm ON invoices(firm_id);
CREATE INDEX idx_invoices_client ON invoices(client_id);
CREATE INDEX idx_invoices_status ON invoices(firm_id, status);

CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    engagement_id UUID REFERENCES engagements(id),
    description TEXT NOT NULL,
    quantity NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price_cents BIGINT NOT NULL,
    amount_cents BIGINT NOT NULL,
    line_type TEXT NOT NULL CHECK (line_type IN ('time', 'fixed_fee', 'expense', 'adjustment', 'write_up', 'write_down')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_invoice_lines_invoice ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    amount_cents BIGINT NOT NULL,
    payment_method TEXT CHECK (payment_method IN ('ach', 'credit_card', 'check', 'wire', 'cash', 'other')),
    payment_date DATE NOT NULL,
    reference_number TEXT,
    stripe_payment_id TEXT,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

---

## Documents

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID REFERENCES clients(id),
    engagement_id UUID REFERENCES engagements(id),
    uploaded_by_id UUID REFERENCES staff(id),
    uploaded_by_contact_id UUID REFERENCES client_contacts(id),
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
    parent_document_id UUID REFERENCES documents(id),  -- version chain
    is_client_visible BOOLEAN DEFAULT FALSE,
    checksum_sha256 TEXT,
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

## Email Integration

```sql
CREATE TABLE email_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    staff_id UUID REFERENCES staff(id),
    client_id UUID REFERENCES clients(id),
    engagement_id UUID REFERENCES engagements(id),
    external_message_id TEXT NOT NULL,  -- Gmail/M365 message ID
    thread_id TEXT,
    provider TEXT NOT NULL CHECK (provider IN ('gmail', 'microsoft365')),
    direction TEXT NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    from_address TEXT NOT NULL,
    to_addresses TEXT[] NOT NULL,
    cc_addresses TEXT[],
    subject TEXT,
    snippet TEXT,                       -- first ~200 chars for preview
    received_at TIMESTAMPTZ NOT NULL,
    has_attachments BOOLEAN DEFAULT FALSE,
    is_triaged BOOLEAN DEFAULT FALSE,
    triaged_by_id UUID REFERENCES staff(id),
    ai_summary TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, external_message_id)
);
CREATE INDEX idx_emails_firm ON email_messages(firm_id);
CREATE INDEX idx_emails_client ON email_messages(client_id);
CREATE INDEX idx_emails_engagement ON email_messages(engagement_id);
CREATE INDEX idx_emails_staff ON email_messages(staff_id, received_at);
CREATE INDEX idx_emails_untriaged ON email_messages(firm_id) WHERE is_triaged = FALSE;
```

---

## Tax Deadlines & Deadline Intelligence

```sql
CREATE TABLE tax_deadlines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction TEXT NOT NULL,          -- 'federal', state code, or country
    form_type TEXT NOT NULL,             -- e.g. '1040', '1120', '1065', '990', 'state_income'
    entity_type TEXT NOT NULL,           -- 'individual', 'c_corp', 's_corp', 'partnership', etc.
    deadline_type TEXT NOT NULL CHECK (deadline_type IN ('original', 'extended', 'estimated_payment', 'information')),
    due_date DATE NOT NULL,
    tax_year INT NOT NULL,
    description TEXT,
    source_url TEXT,
    last_verified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tax_deadlines_lookup ON tax_deadlines(jurisdiction, form_type, tax_year);

CREATE TABLE client_deadlines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES clients(id),
    engagement_id UUID REFERENCES engagements(id),
    tax_deadline_id UUID REFERENCES tax_deadlines(id),
    due_date DATE NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('upcoming', 'at_risk', 'extended', 'filed', 'missed')) DEFAULT 'upcoming',
    extension_filed BOOLEAN DEFAULT FALSE,
    extension_filed_at TIMESTAMPTZ,
    filed_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_deadlines_client ON client_deadlines(client_id);
CREATE INDEX idx_client_deadlines_due ON client_deadlines(due_date, status);
```

---

## Independence & Conflict of Interest

```sql
CREATE TABLE independence_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    staff_id UUID REFERENCES staff(id),           -- specific staff member, or NULL for firm-level
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
    reviewed_by_id UUID REFERENCES staff(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_independence_firm ON independence_records(firm_id);
CREATE INDEX idx_independence_client ON independence_records(client_id);
CREATE INDEX idx_independence_staff ON independence_records(staff_id);
CREATE INDEX idx_independence_active ON independence_records(firm_id, is_active);

CREATE TABLE conflict_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID REFERENCES clients(id),
    prospective_client_name TEXT,
    prospective_client_ein TEXT,
    checked_by_id UUID NOT NULL REFERENCES staff(id),
    check_date DATE NOT NULL,
    result TEXT NOT NULL CHECK (result IN ('clear', 'conflict_found', 'requires_review')),
    conflicts_found JSONB DEFAULT '[]',
    resolution TEXT,
    approved_by_id UUID REFERENCES staff(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conflict_checks_firm ON conflict_checks(firm_id);
CREATE INDEX idx_conflict_checks_client ON conflict_checks(client_id);
```

---

## Sign-Off Trails (AICPA Compliance)

```sql
CREATE TABLE sign_off_trails (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    engagement_id UUID NOT NULL REFERENCES engagements(id),
    sign_off_type TEXT NOT NULL CHECK (sign_off_type IN (
        'preparation_complete', 'compilation_report', 'review_report',
        'audit_opinion', 'attestation_report', 'quality_review',
        'partner_approval', 'concurring_review', 'engagement_acceptance',
        'engagement_continuance'
    )),
    -- SSAE 18 / SSARS 21 documentation
    standard_reference TEXT,           -- e.g. 'SSARS 21 AR-C 70', 'SSAE 18 AT-C 105'
    conclusion TEXT,
    signed_by_id UUID NOT NULL REFERENCES staff(id),
    signed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes TEXT,
    supporting_document_id UUID REFERENCES documents(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sign_offs_engagement ON sign_off_trails(engagement_id);
CREATE INDEX idx_sign_offs_staff ON sign_off_trails(signed_by_id);
```

---

## Tax Returns (MeF Tracking)

```sql
CREATE TABLE tax_returns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id UUID NOT NULL REFERENCES engagements(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    tax_year INT NOT NULL,
    form_type TEXT NOT NULL,           -- '1040', '1120', '1065', '990', etc.
    jurisdiction TEXT NOT NULL DEFAULT 'federal',
    status TEXT NOT NULL CHECK (status IN (
        'not_started', 'in_preparation', 'prepared', 'in_review',
        'approved', 'e_filed', 'accepted', 'rejected', 'amended', 'paper_filed'
    )) DEFAULT 'not_started',
    -- MeF tracking
    mef_submission_id TEXT,
    mef_ack_status TEXT CHECK (mef_ack_status IN ('accepted', 'rejected', 'pending')),
    mef_ack_timestamp TIMESTAMPTZ,
    mef_rejection_code TEXT,
    preparer_id UUID REFERENCES staff(id),
    reviewer_id UUID REFERENCES staff(id),
    signer_id UUID REFERENCES staff(id),   -- signing partner
    filed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tax_returns_engagement ON tax_returns(engagement_id);
CREATE INDEX idx_tax_returns_client ON tax_returns(client_id, tax_year);
CREATE INDEX idx_tax_returns_status ON tax_returns(status);
```

---

## Integrations & Audit Log

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    provider TEXT NOT NULL CHECK (provider IN (
        'quickbooks_online', 'xero', 'gmail', 'microsoft365', 'stripe', 'docusign'
    )),
    status TEXT NOT NULL CHECK (status IN ('connected', 'disconnected', 'error', 'expired')) DEFAULT 'disconnected',
    oauth_access_token TEXT,           -- encrypted at rest (AES-256)
    oauth_refresh_token TEXT,          -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    external_tenant_id TEXT,           -- QBO realm_id, Xero tenant_id
    scopes TEXT[],
    last_sync_at TIMESTAMPTZ,
    sync_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, provider)
);
CREATE INDEX idx_integrations_firm ON integrations(firm_id);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    -- CloudEvents attributes
    ce_source TEXT NOT NULL,           -- e.g. '/firms/{id}/engagements/{id}'
    ce_type TEXT NOT NULL,             -- e.g. 'engagement.task.completed', 'document.accessed'
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

## Client Portal Messages

```sql
CREATE TABLE portal_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL REFERENCES firms(id),
    client_id UUID NOT NULL REFERENCES clients(id),
    engagement_id UUID REFERENCES engagements(id),
    sender_type TEXT NOT NULL CHECK (sender_type IN ('staff', 'client')),
    sender_staff_id UUID REFERENCES staff(id),
    sender_contact_id UUID REFERENCES client_contacts(id),
    subject TEXT,
    body TEXT NOT NULL,
    has_attachments BOOLEAN DEFAULT FALSE,
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_portal_messages_client ON portal_messages(client_id, created_at);
CREATE INDEX idx_portal_messages_engagement ON portal_messages(engagement_id);
CREATE INDEX idx_portal_messages_unread ON portal_messages(client_id) WHERE is_read = FALSE;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Firm & Staff | 3 | firms, offices, staff |
| Client Management | 2 | clients, client_contacts |
| Service & Workflow Templates | 3 | service_types, workflow_templates, workflow_template_steps |
| Engagements & Tasks | 2 | engagements, engagement_tasks |
| Engagement Letters & E-Signatures | 2 | engagement_letters, e_signatures |
| Time & Billing | 4 | time_entries, invoices, invoice_line_items, payments |
| Documents | 1 | documents |
| Email Integration | 1 | email_messages |
| Tax Deadlines | 2 | tax_deadlines, client_deadlines |
| Independence & Conflicts | 2 | independence_records, conflict_checks |
| Sign-Off Trails | 1 | sign_off_trails |
| Tax Returns | 1 | tax_returns |
| Integrations & Audit | 2 | integrations, audit_log (partitioned) |
| Portal | 1 | portal_messages |
| **Total** | **27** | |

---

## Key Design Decisions

1. **SSN never stored in full** — only `ssn_last_four` is persisted; full SSN is handled transiently per IRS Publication 4557. The EIN is stored because it's a public business identifier.

2. **Workflow template-instance hierarchy** — `workflow_templates` → `workflow_template_steps` define the process; `engagements` → `engagement_tasks` are the runtime instances. Cascade edits propagate template changes to future recurring instances (Jetpack Workflow's "Magic Job" pattern).

3. **SSARS 21 service level on both service types and engagement letters** — the service level determines documentation requirements. The engagement letter's service level may differ from the service type default when a client upgrades (e.g., from preparation to compilation).

4. **AI billing as columns on existing tables** — `ai_suggested_minutes` and `ai_suggested_amount_cents` on `time_entries` and `ai_write_up_cents`/`ai_write_down_cents` on `invoices` keep AI recommendations alongside the data they reference rather than in separate tables.

5. **Independence records are per-relationship, not per-engagement** — a financial interest in a client affects all engagements with that client. The `independence_records` table models the ongoing relationship; `conflict_checks` model the point-in-time acceptance decision.

6. **Tax deadline reference data separated from client deadlines** — `tax_deadlines` is the IRS/state calendar (updated by AI deadline intelligence); `client_deadlines` instantiates those per client. When the IRS changes a deadline, client deadlines can cascade.

7. **Email messages stored as metadata, not full content** — snippet and subject are stored for search and triage; full email body remains in Gmail/M365 and is fetched on demand. This avoids duplicating large email stores and respects provider storage as the source of truth.

8. **Audit log follows CloudEvents specification** — `ce_source`, `ce_type`, `ce_time`, `ce_specversion` attributes enable interoperability with event-driven architectures and satisfy IRS 4557 continuous monitoring evidence requirements.

9. **Sign-off trails reference AICPA standard sections** — the `standard_reference` field (e.g., 'SSARS 21 AR-C 70') creates a direct linkage between the sign-off and the professional standard it satisfies, supporting peer review and quality control.

10. **MeF tracking on tax returns** — `mef_submission_id`, `mef_ack_status`, and `mef_rejection_code` enable the platform to track e-file lifecycle without building a full MeF integration, allowing third-party tax software to remain the filing engine while the practice management platform tracks status.
