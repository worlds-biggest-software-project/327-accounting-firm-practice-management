# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Accounting Firm Practice Management · Created: 2025-05-25

## Philosophy

This model treats the immutable event store as the single source of truth. Every state change — a task completed, an engagement letter signed, a time entry approved, a client health score updated — is appended as an event. Read models (materialised views prefixed `rm_`) are projected from events to serve operational queries. The CQRS pattern separates the write path (append to event store) from the read path (query read models).

For an accounting practice management platform, event sourcing is a natural fit: IRS Publication 4557 and the FTC Safeguards Rule demand continuous monitoring evidence and tamper-evident audit trails. AICPA SSARS/SSAE standards require documented sign-off chains. Rather than bolting an audit log onto a mutable relational model, this architecture makes the audit trail the foundation — every change is automatically captured because events are how changes happen.

The event-sourced approach also enables powerful AI features: the complete history of every engagement, billing decision, and client interaction is available for pattern analysis. AI write-up/write-down recommendations can reference comparable engagement histories. Client health scoring can analyse communication and payment event patterns. Deadline intelligence can replay past filing patterns to predict risk.

**Best for:** Firms with strong compliance requirements (public accounting, attestation engagements), platforms where AI analytics on historical patterns are a core value proposition, and architectures designed for long-term regulatory defensibility.

**Trade-offs:**
- (+) Complete, immutable audit trail satisfies IRS 4557, FTC Safeguards, and AICPA standards by construction
- (+) Temporal queries ("what was the engagement status on March 15?") are trivial — replay events to that point
- (+) AI analytics can process the full event history for pattern detection
- (+) Adding new projections (read models) requires no schema change to the source data
- (-) Read models must be maintained and can diverge from events if projection logic has bugs
- (-) Event replay for large streams requires snapshotting for performance
- (-) Developers must understand CQRS — simple CRUD operations become event-append + projection
- (-) Storage grows linearly with every state change; requires partition management

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 4557 | The event store IS the continuous monitoring evidence — every data access, change, and decision is an event |
| FTC Safeguards Rule | Tamper-evident event store satisfies documented security programme evidence requirements |
| AICPA SSAE 18 | Sign-off events (`engagement.sign_off.*`) create an immutable attestation trail with standard references |
| AICPA SSARS 21 | `engagement.service_level_set` events document the preparation/compilation/review determination |
| IRS MeF | `tax_return.e_filed`, `tax_return.accepted`, `tax_return.rejected` events track MeF lifecycle |
| PCAOB Independence | `independence.*` events create an immutable record of conflict checks, findings, and resolutions |
| SOC 2 | Event store satisfies all five Trust Services Criteria evidence requirements |
| GDPR / CCPA | `client.consent_given`, `client.data_deletion_requested` events; event replay enables data subject access reports |
| CloudEvents | Every event follows CloudEvents 1.0 attribute structure |

---

## Event Store Infrastructure

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- CloudEvents attributes
    ce_source TEXT NOT NULL,
    ce_type TEXT NOT NULL,
    ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    -- Actor tracking
    actor_type TEXT NOT NULL CHECK (actor_type IN ('staff', 'client', 'system', 'ai', 'integration')),
    actor_id UUID,
    actor_ip INET,
    -- Ordering
    sequence_number BIGINT NOT NULL,
    global_position BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);
CREATE INDEX idx_events_global ON event_store(global_position);
CREATE INDEX idx_events_actor ON event_store(actor_id, ce_time);
CREATE INDEX idx_events_ce_type ON event_store(ce_type);

-- Half-yearly partitions
CREATE TABLE event_store_2026h1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE event_store_2026h2 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    snapshot_data JSONB NOT NULL,
    last_sequence_number BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, last_sequence_number)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalogue

### Firm Events (`stream_type: 'firm'`)
- `firm.created` — firm registration
- `firm.settings_updated` — settings, offices, rates changed
- `firm.service_type_added`, `firm.service_type_updated` — service catalogue changes
- `firm.workflow_template_created`, `firm.workflow_template_updated` — template definitions
- `firm.integration_connected`, `firm.integration_disconnected`, `firm.integration_synced` — QBO, Xero, Gmail, M365

### Staff Events (`stream_type: 'staff'`)
- `staff.created`, `staff.updated`, `staff.deactivated`
- `staff.role_changed` — role promotions/changes
- `staff.licence_updated` — CPA licence, PTIN changes

### Client Events (`stream_type: 'client'`)
- `client.created`, `client.updated`, `client.deactivated`
- `client.contact_added`, `client.contact_updated`, `client.contact_removed`
- `client.portal_enabled`, `client.portal_login`
- `client.health_score_computed` — AI health score update
- `client.consent_given`, `client.consent_revoked` — GDPR/CCPA
- `client.data_deletion_requested`, `client.data_deletion_completed`
- `client.tag_added`, `client.tag_removed`

### Engagement Events (`stream_type: 'engagement'`)
- `engagement.created`, `engagement.updated`, `engagement.cancelled`
- `engagement.status_changed` — planned → in_progress → completed lifecycle
- `engagement.staff_assigned` — partner, manager, preparer assignments
- `engagement.service_level_set` — SSARS 21 determination
- `engagement.recurring_generated` — auto-generated from recurrence rule
- **Task events:**
  - `engagement.task_created`, `engagement.task_assigned`
  - `engagement.task_started`, `engagement.task_completed`, `engagement.task_blocked`
  - `engagement.client_task_sent`, `engagement.client_task_completed`
  - `engagement.client_task_reminder_sent` — automated follow-up
- **Engagement letter events:**
  - `engagement.letter_drafted`, `engagement.letter_ai_drafted`
  - `engagement.letter_sent`, `engagement.letter_viewed`
  - `engagement.letter_signed`, `engagement.letter_declined`, `engagement.letter_expired`
- **Sign-off events (AICPA compliance):**
  - `engagement.sign_off.preparation_complete`
  - `engagement.sign_off.review_complete`
  - `engagement.sign_off.partner_approval`
  - `engagement.sign_off.concurring_review`
  - `engagement.sign_off.engagement_acceptance`

### Tax Return Events (`stream_type: 'tax_return'`)
- `tax_return.created`, `tax_return.preparation_started`
- `tax_return.prepared`, `tax_return.reviewed`, `tax_return.approved`
- `tax_return.e_filed` — MeF submission with submission_id
- `tax_return.accepted` — MeF acknowledgment received
- `tax_return.rejected` — MeF rejection with rejection_code
- `tax_return.extension_filed`
- `tax_return.delivered_to_client`
- `tax_return.form_8879_signed` — e-signature captured

### Time Entry Events (`stream_type: 'time_entry'`)
- `time_entry.logged` — manual, timer, or calendar import
- `time_entry.ai_drafted` — AI-suggested time entry
- `time_entry.ai_draft_accepted`, `time_entry.ai_draft_modified`, `time_entry.ai_draft_rejected`
- `time_entry.approved`, `time_entry.written_off`
- `time_entry.invoiced` — linked to invoice

### Invoice Events (`stream_type: 'invoice'`)
- `invoice.created`, `invoice.ai_drafted`
- `invoice.ai_write_up_recommended`, `invoice.ai_write_down_recommended`
- `invoice.write_up_applied`, `invoice.write_down_applied`
- `invoice.line_item_added`, `invoice.line_item_removed`
- `invoice.reviewed`, `invoice.sent`, `invoice.viewed`
- `invoice.payment_received` — with method, amount, reference
- `invoice.partially_paid`, `invoice.paid`
- `invoice.overdue`, `invoice.reminder_sent`
- `invoice.written_off`, `invoice.voided`

### Document Events (`stream_type: 'document'`)
- `document.uploaded` — with uploader type (staff/client), category, tax_year
- `document.version_created`
- `document.accessed`, `document.downloaded`
- `document.shared_with_client`, `document.client_visibility_revoked`
- `document.e_signature_requested`, `document.e_signature_completed`
- `document.deleted`

### Email Events (`stream_type: 'email'`)
- `email.synced` — from Gmail/M365
- `email.triaged` — assigned to engagement/client
- `email.ai_summarised` — AI summary generated
- `email.ai_response_drafted` — AI draft response created
- `email.response_sent`

### Independence Events (`stream_type: 'independence'`)
- `independence.record_created` — financial interest, relationship, etc.
- `independence.record_updated`, `independence.record_expired`
- `independence.conflict_check_initiated`
- `independence.conflict_check_clear`
- `independence.conflict_check_conflict_found`
- `independence.conflict_resolved` — with safeguards applied
- `independence.annual_review_completed`

### Deadline Events (`stream_type: 'deadline'`)
- `deadline.calendar_updated` — IRS/state calendar change detected
- `deadline.client_deadline_created` — per-client deadline instantiated
- `deadline.client_deadline_cascaded` — deadline changed due to calendar update
- `deadline.at_risk` — approaching deadline with incomplete work
- `deadline.extension_filed`
- `deadline.met`, `deadline.missed`

### Portal Events (`stream_type: 'portal'`)
- `portal.message_sent` — staff → client
- `portal.message_received` — client → staff
- `portal.message_read`
- `portal.task_completed_by_client`
- `portal.document_uploaded_by_client`

### AI Events (`stream_type: 'ai'`)
- `ai.workflow_generated` — AI created workflow from natural language
- `ai.engagement_letter_drafted` — AI drafted from prior-year data
- `ai.invoice_analysed` — write-up/write-down recommendation
- `ai.time_entry_suggested` — missed time entry detected
- `ai.client_health_computed` — health score with contributing factors
- `ai.deadline_risk_assessed` — deadline risk prediction
- `ai.anomaly_detected` — prior-year comparison flag
- `ai.email_summarised`, `ai.email_response_drafted`

---

## Read Models

### rm_engagements — Engagement Dashboard

```sql
CREATE TABLE rm_engagements (
    engagement_id UUID PRIMARY KEY,
    firm_id UUID NOT NULL,
    client_id UUID NOT NULL,
    client_name TEXT NOT NULL,
    name TEXT NOT NULL,
    service_category TEXT NOT NULL,
    service_level TEXT,
    status TEXT NOT NULL,
    period_label TEXT,
    due_date DATE,
    extended_due_date DATE,
    fee_type TEXT,
    fee_amount_cents BIGINT,
    budget_hours NUMERIC(7,2),
    actual_hours NUMERIC(7,2),
    assigned_partner TEXT,
    assigned_manager TEXT,
    assigned_preparer TEXT,
    priority INT,
    tags TEXT[],
    task_summary JSONB,
    -- task_summary example:
    -- {
    --   "total": 8, "completed": 5, "in_progress": 2, "blocked": 0,
    --   "awaiting_client": 1,
    --   "next_task": {"name": "Manager review", "assigned_to": "Jane Doe", "due_date": "2026-03-10"},
    --   "overdue_count": 0
    -- }
    engagement_letter_status TEXT,
    sign_off_chain JSONB DEFAULT '[]',
    -- sign_off_chain example:
    -- [
    --   {"type": "preparation_complete", "standard": "SSARS 21 AR-C 70",
    --    "signed_by": "Staff Name", "signed_at": "2026-03-15T11:00:00Z"},
    --   {"type": "partner_approval", "signed_by": "Partner Name",
    --    "signed_at": "2026-03-16T09:00:00Z"}
    -- ]
    tax_return_status TEXT,
    mef_status TEXT,
    billing_status TEXT,
    last_activity_at TIMESTAMPTZ,
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_eng_firm ON rm_engagements(firm_id);
CREATE INDEX idx_rm_eng_client ON rm_engagements(client_id);
CREATE INDEX idx_rm_eng_status ON rm_engagements(firm_id, status);
CREATE INDEX idx_rm_eng_due ON rm_engagements(firm_id, due_date);
```

### rm_clients — Client Overview

```sql
CREATE TABLE rm_clients (
    client_id UUID PRIMARY KEY,
    firm_id UUID NOT NULL,
    client_number TEXT,
    name TEXT NOT NULL,
    client_type TEXT NOT NULL,
    is_active BOOLEAN,
    assigned_partner TEXT,
    assigned_manager TEXT,
    tags TEXT[],
    contact_count INT DEFAULT 0,
    portal_enabled BOOLEAN DEFAULT FALSE,
    portal_last_login TIMESTAMPTZ,
    -- Health scoring
    health_score NUMERIC(5,2),
    health_factors JSONB,
    -- health_factors example:
    -- {
    --   "communication_responsiveness": 85,
    --   "document_timeliness": 72,
    --   "ar_aging_score": 90,
    --   "engagement_completion_rate": 95,
    --   "risk_level": "low",
    --   "at_risk_reason": null
    -- }
    -- Engagement summary
    active_engagements INT DEFAULT 0,
    total_engagements INT DEFAULT 0,
    open_balance_cents BIGINT DEFAULT 0,
    total_revenue_cents BIGINT DEFAULT 0,
    last_engagement_completed_at TIMESTAMPTZ,
    last_communication_at TIMESTAMPTZ,
    -- Independence
    has_independence_issues BOOLEAN DEFAULT FALSE,
    independence_threat_level TEXT,
    last_conflict_check_at TIMESTAMPTZ,
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_clients_firm ON rm_clients(firm_id);
CREATE INDEX idx_rm_clients_health ON rm_clients(firm_id, health_score);
CREATE INDEX idx_rm_clients_active ON rm_clients(firm_id, is_active);
CREATE INDEX idx_rm_clients_tags ON rm_clients USING GIN (tags);
```

### rm_staff_utilisation — Staff Workload & Capacity

```sql
CREATE TABLE rm_staff_utilisation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id UUID NOT NULL,
    firm_id UUID NOT NULL,
    week_starting DATE NOT NULL,  -- Monday of the week
    full_name TEXT NOT NULL,
    role TEXT NOT NULL,
    target_hours NUMERIC(5,1),
    billable_hours NUMERIC(7,2) DEFAULT 0,
    non_billable_hours NUMERIC(7,2) DEFAULT 0,
    total_hours NUMERIC(7,2) DEFAULT 0,
    utilisation_pct NUMERIC(5,1) DEFAULT 0,
    -- Breakdown by source
    manual_hours NUMERIC(7,2) DEFAULT 0,
    timer_hours NUMERIC(7,2) DEFAULT 0,
    ai_draft_hours NUMERIC(7,2) DEFAULT 0,
    calendar_import_hours NUMERIC(7,2) DEFAULT 0,
    -- AI insights
    ai_suggested_hours NUMERIC(7,2) DEFAULT 0,  -- AI-detected missed entries
    -- Engagement assignments
    active_engagements INT DEFAULT 0,
    overdue_tasks INT DEFAULT 0,
    entries JSONB DEFAULT '[]',
    -- entries example:
    -- [
    --   {"date": "2026-03-18", "engagement": "Smith 1040", "minutes": 90,
    --    "source": "manual", "status": "approved"},
    --   {"date": "2026-03-19", "engagement": "Jones Bookkeeping", "minutes": 60,
    --    "source": "ai_draft", "ai_adjustment": {"suggested": 45, "accepted": true}}
    -- ]
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (staff_id, week_starting)
);
CREATE INDEX idx_rm_util_firm ON rm_staff_utilisation(firm_id, week_starting);
CREATE INDEX idx_rm_util_staff ON rm_staff_utilisation(staff_id, week_starting);
```

### rm_invoices — Billing Dashboard

```sql
CREATE TABLE rm_invoices (
    invoice_id UUID PRIMARY KEY,
    firm_id UUID NOT NULL,
    client_id UUID NOT NULL,
    client_name TEXT NOT NULL,
    invoice_number TEXT NOT NULL,
    status TEXT NOT NULL,
    issue_date DATE,
    due_date DATE,
    total_cents BIGINT,
    amount_paid_cents BIGINT,
    balance_due_cents BIGINT,
    line_item_count INT DEFAULT 0,
    -- AI billing analysis
    ai_write_up_cents BIGINT DEFAULT 0,
    ai_write_down_cents BIGINT DEFAULT 0,
    ai_billing_notes TEXT,
    -- Payment timeline
    timeline JSONB DEFAULT '[]',
    -- timeline example:
    -- [
    --   {"event": "created", "at": "2026-04-01T08:00:00Z", "by": "System"},
    --   {"event": "ai_analysed", "at": "2026-04-01T08:01:00Z",
    --    "notes": "Recommend $50 write-down based on comparable engagements"},
    --   {"event": "write_down_applied", "at": "2026-04-01T10:00:00Z",
    --    "amount_cents": 5000, "by": "Partner Name"},
    --   {"event": "sent", "at": "2026-04-02T09:00:00Z", "by": "Admin Name"},
    --   {"event": "viewed", "at": "2026-04-02T14:00:00Z"},
    --   {"event": "payment_received", "at": "2026-04-15T10:00:00Z",
    --    "amount_cents": 55000, "method": "credit_card"}
    -- ]
    days_outstanding INT,
    is_overdue BOOLEAN DEFAULT FALSE,
    stripe_invoice_id TEXT,
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_inv_firm ON rm_invoices(firm_id);
CREATE INDEX idx_rm_inv_client ON rm_invoices(client_id);
CREATE INDEX idx_rm_inv_status ON rm_invoices(firm_id, status);
CREATE INDEX idx_rm_inv_overdue ON rm_invoices(firm_id) WHERE is_overdue = TRUE;
```

### rm_deadline_tracker — Deadline Intelligence

```sql
CREATE TABLE rm_deadline_tracker (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    client_id UUID NOT NULL,
    client_name TEXT NOT NULL,
    engagement_id UUID,
    jurisdiction TEXT NOT NULL,
    form_type TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    tax_year INT NOT NULL,
    deadline_type TEXT NOT NULL,
    due_date DATE NOT NULL,
    status TEXT NOT NULL,
    -- Risk assessment
    days_remaining INT,
    risk_level TEXT CHECK (risk_level IN ('on_track', 'at_risk', 'critical', 'overdue', 'completed')),
    engagement_status TEXT,
    engagement_completion_pct NUMERIC(5,1),
    -- Extension tracking
    extension_filed BOOLEAN DEFAULT FALSE,
    extension_filed_at TIMESTAMPTZ,
    extended_due_date DATE,
    -- AI prediction
    ai_risk_score NUMERIC(5,2),
    ai_risk_factors JSONB,
    -- ai_risk_factors example:
    -- {
    --   "missing_documents": ["W-2 from employer", "1099-DIV from broker"],
    --   "prior_year_filed_days_before": 3,
    --   "engagement_behind_schedule": true,
    --   "prediction": "High risk of missing deadline based on prior-year pattern"
    -- }
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (client_id, jurisdiction, form_type, tax_year, deadline_type)
);
CREATE INDEX idx_rm_deadlines_firm ON rm_deadline_tracker(firm_id);
CREATE INDEX idx_rm_deadlines_due ON rm_deadline_tracker(firm_id, due_date);
CREATE INDEX idx_rm_deadlines_risk ON rm_deadline_tracker(firm_id, risk_level);
CREATE INDEX idx_rm_deadlines_client ON rm_deadline_tracker(client_id);
```

### rm_firm_dashboard — Firm-Wide KPIs

```sql
CREATE TABLE rm_firm_dashboard (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    snapshot_date DATE NOT NULL,
    -- Engagement metrics
    active_engagements INT DEFAULT 0,
    engagements_completed_this_month INT DEFAULT 0,
    engagements_overdue INT DEFAULT 0,
    avg_engagement_cycle_days NUMERIC(7,1),
    -- Billing metrics
    wip_cents BIGINT DEFAULT 0,           -- unbilled time
    invoiced_this_month_cents BIGINT DEFAULT 0,
    collected_this_month_cents BIGINT DEFAULT 0,
    ar_outstanding_cents BIGINT DEFAULT 0,
    ar_over_90_days_cents BIGINT DEFAULT 0,
    write_up_this_month_cents BIGINT DEFAULT 0,
    write_down_this_month_cents BIGINT DEFAULT 0,
    realisation_rate_pct NUMERIC(5,1),    -- collected / standard billing
    -- Staff metrics
    avg_utilisation_pct NUMERIC(5,1),
    total_billable_hours_this_week NUMERIC(7,1),
    -- Client metrics
    active_clients INT DEFAULT 0,
    clients_at_risk INT DEFAULT 0,        -- health score below threshold
    avg_client_health_score NUMERIC(5,1),
    -- Compliance metrics
    unsigned_engagement_letters INT DEFAULT 0,
    pending_sign_offs INT DEFAULT 0,
    upcoming_deadlines_7_days INT DEFAULT 0,
    overdue_deadlines INT DEFAULT 0,
    independence_issues_open INT DEFAULT 0,
    -- Tax season metrics
    returns_in_progress INT DEFAULT 0,
    returns_filed INT DEFAULT 0,
    returns_accepted INT DEFAULT 0,
    returns_rejected INT DEFAULT 0,
    extensions_filed INT DEFAULT 0,
    last_event_id UUID,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, snapshot_date)
);
CREATE INDEX idx_rm_dash_firm ON rm_firm_dashboard(firm_id, snapshot_date);
```

### rm_ai_insights — AI Activity Feed

```sql
CREATE TABLE rm_ai_insights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id UUID NOT NULL,
    insight_type TEXT NOT NULL CHECK (insight_type IN (
        'workflow_generated', 'engagement_letter_drafted', 'invoice_analysed',
        'time_entry_suggested', 'health_score_computed', 'deadline_risk',
        'anomaly_detected', 'email_summarised', 'response_drafted'
    )),
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    client_id UUID,
    staff_id UUID,
    title TEXT NOT NULL,
    detail JSONB NOT NULL,
    -- detail example (invoice_analysed):
    -- {
    --   "engagement": "Smith 2025 1040",
    --   "budget_hours": 2.0, "actual_hours": 1.5,
    --   "standard_amount_cents": 50000,
    --   "recommended_amount_cents": 45000,
    --   "adjustment_type": "write_down",
    --   "comparable_engagements": 12,
    --   "confidence": 0.87,
    --   "reason": "Actual time 25% below budget; 12 comparable returns averaged $420"
    -- }
    status TEXT NOT NULL CHECK (status IN ('pending', 'accepted', 'modified', 'rejected', 'expired')) DEFAULT 'pending',
    acted_on_by UUID,
    acted_on_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ
);
CREATE INDEX idx_rm_ai_firm ON rm_ai_insights(firm_id, created_at);
CREATE INDEX idx_rm_ai_status ON rm_ai_insights(firm_id, status);
CREATE INDEX idx_rm_ai_type ON rm_ai_insights(firm_id, insight_type);
CREATE INDEX idx_rm_ai_entity ON rm_ai_insights(entity_type, entity_id);
```

---

## Event-Driven Automation Patterns

### Auto-Generated Recurring Engagements
When `engagement.completed` fires for a recurring engagement, the system generates the next instance by emitting `engagement.recurring_generated` with tasks copied from the workflow template. Cascade template edits propagate automatically.

### Client Task Follow-Up
When `engagement.client_task_sent` fires and no `engagement.client_task_completed` event arrives within the configured reminder window, the system emits `engagement.client_task_reminder_sent` and sends the automated follow-up.

### AI Write-Up/Write-Down
When `invoice.created` fires, the AI projection analyses `time_entry.logged` and `time_entry.approved` events for the engagement, compares against historical patterns, and emits `invoice.ai_write_up_recommended` or `invoice.ai_write_down_recommended`.

### Deadline Cascade
When `deadline.calendar_updated` fires (IRS changed a deadline), the system identifies all affected `client_deadline` streams and emits `deadline.client_deadline_cascaded` events, updating the `rm_deadline_tracker` read model and triggering notifications.

### Client Health Scoring
A periodic projection processes `portal.message_received`, `portal.document_uploaded_by_client`, `invoice.payment_received`, `engagement.client_task_completed`, and `invoice.overdue` events to compute a composite health score, emitting `client.health_score_computed`.

### Independence Monitoring
When `engagement.created` fires for a client marked as an attestation client, the system cross-references `independence.*` events and emits `independence.conflict_check_initiated` if no recent check exists.

### MeF Acknowledgment Tracking
When `tax_return.e_filed` fires, the system begins polling for MeF acknowledgment. Upon receipt, `tax_return.accepted` or `tax_return.rejected` is emitted, updating the `rm_engagements` and `rm_deadline_tracker` read models.

### Sign-Off Chain Validation
The engagement projection enforces sign-off sequencing: `engagement.sign_off.partner_approval` cannot be projected unless `engagement.sign_off.preparation_complete` or `engagement.sign_off.review_complete` already exists in the stream.

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Model: Engagements | 1 | rm_engagements (tasks, sign-offs, tax return status) |
| Read Model: Clients | 1 | rm_clients (health scoring, independence status, revenue) |
| Read Model: Staff | 1 | rm_staff_utilisation (weekly, with AI insights) |
| Read Model: Invoices | 1 | rm_invoices (timeline, AI analysis) |
| Read Model: Deadlines | 1 | rm_deadline_tracker (risk assessment, AI prediction) |
| Read Model: Firm KPIs | 1 | rm_firm_dashboard (daily snapshot) |
| Read Model: AI | 1 | rm_ai_insights (actionable AI recommendations) |
| **Total** | **10** | 3 infrastructure + 7 read models |

---

## Key Design Decisions

1. **The event store IS the audit trail** — IRS Publication 4557 requires continuous monitoring evidence. Rather than building an audit log alongside mutable tables, every state change is an immutable event. The audit trail is a by-product of the architecture, not an afterthought.

2. **Actor type includes 'ai'** — AI-generated events (draft time entries, engagement letters, billing recommendations) are attributed to the AI actor, creating a clear provenance chain. When a staff member accepts or modifies an AI suggestion, that's a separate event from a human actor.

3. **Sign-off events encode AICPA standard references** — `engagement.sign_off.preparation_complete` events include the standard reference (e.g., 'SSARS 21 AR-C 70') in event_data, making the sign-off trail directly auditable against professional standards.

4. **Deadline risk as an AI-driven read model** — `rm_deadline_tracker` combines calendar data with engagement progress and historical filing patterns. The `ai_risk_factors` JSONB captures the reasoning behind risk predictions, making AI decisions transparent and explainable.

5. **rm_ai_insights as an actionable feed** — rather than hiding AI recommendations inside other read models, a dedicated `rm_ai_insights` table creates a centralised action queue. Staff can review, accept, modify, or reject each insight with a tracked status.

6. **Client health scoring from event patterns** — health scores are computed by replaying communication, payment, and engagement events rather than from static field values. This means the score automatically reflects changes without explicit recalculation triggers.

7. **Recurring engagement generation as events** — when an engagement completes and is part of a recurring series, the system generates the next instance through an event rather than a cron job. This makes the generation auditable and replayable.

8. **Independence events as a separate stream type** — independence tracking is cross-engagement (a conflict affects all engagements with a client), so it gets its own stream indexed by client rather than being nested under engagement events.

9. **rm_firm_dashboard as a daily snapshot** — firm-wide KPIs are projected once per day into a single row, giving partners instant access to the metrics they check every morning: WIP, AR, utilisation, deadline status, and tax season progress.

10. **Half-yearly event store partitions** — accounting work follows annual cycles with predictable volume spikes during tax season (January–April, September–October). Half-yearly partitions balance partition size with management overhead.
