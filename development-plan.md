# Accounting Firm Practice Management — Phased Development Plan

> Project: 327-accounting-firm-practice-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (the entity-centric normalized relational model, chosen for its first-class compliance, auditability, and cross-entity queryability) into concrete technology decisions, architecture, and a phased build sequence. The product is an AI-native, open-source, multi-tenant SaaS practice-management platform for accounting firms, self-hostable and optionally managed.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **TypeScript** (Node.js 22 LTS) | The product is API + web-portal heavy with many OAuth integrations (QBO, Xero, Graph, Google, Stripe, DocuSign) that all ship first-class TS SDKs. One language across API, background workers, and frontend reduces context-switching and lets domain types (engagement, task, invoice) be shared end-to-end. |
| API framework | **NestJS** | Module/provider DI structure maps cleanly onto the 14 domain areas of data-model-1; built-in OpenAPI 3.1 generation (required to publish a documented public API — the explicit gap among incumbents per research.md); guards/interceptors give a clean place to enforce multi-tenant isolation (OWASP A01) and audit logging. |
| Database | **PostgreSQL 16** | Data-model-1 relies on UUIDs, `JSONB`, `TEXT[]`, `GIN` indexes, `CHECK` constraints, `INET`, and range-partitioned `audit_log` — all native to Postgres. Referential integrity enforces business rules (cannot delete a client with open engagements) at the DB layer. |
| ORM / migrations | **Drizzle ORM + drizzle-kit** | SQL-first ORM that expresses the normalized DDL faithfully (partial indexes, CHECK constraints, partitioning hooks) without hiding it behind abstractions; type-safe queries derived from schema; explicit, reviewable SQL migrations needed for a compliance product. |
| Task queue | **BullMQ on Redis** | Async workloads dominate: webhook fan-out, email sync, LLM calls (engagement-letter drafting, billing analysis), deadline cascades, health-score recompute. BullMQ gives retries, scheduling (repeatable jobs for recurring-engagement generation and deadline polling), and dead-letter handling. |
| Cache / sessions / rate-limit | **Redis** | Shared with BullMQ; backs session/refresh-token revocation lists, integration-token short-TTL cache, and API rate limiting. |
| LLM access | **Vercel AI SDK** with a provider-abstraction layer (Anthropic default; OpenAI/Azure OpenAI pluggable) | AI is the core differentiator and must be swappable for self-hosters (some firms mandate Azure OpenAI for data-residency). The AI SDK gives structured-output (`generateObject`) for typed extraction (engagement scope, write-up/down recommendations) and streaming for drafting UIs. |
| Object storage | **S3-compatible (AWS S3 / MinIO for self-host)** | Documents (`documents.file_path`) must be AES-256 encrypted at rest (IRS Pub 4557); S3 SSE-KMS for managed, MinIO for self-host. DB stores only metadata + checksum. |
| Frontend (firm app) | **Next.js 16 (App Router) + React + shadcn/ui + Tailwind** | Server Components for the firm dashboard (Kanban, capacity views); WCAG 2.2 AA achievable with shadcn/Radix primitives. |
| Frontend (client portal) | Same Next.js app, separate route group + auth realm | Reuses component library; strict tenant/role separation between staff and client_contact realms. |
| AuthN/AuthZ | **OIDC + OAuth 2.0** (self-issued JWT access tokens; optional enterprise SSO via Okta/Entra/Google) | NIST 800-63B AAL2 + MFA mandated by IRS Pub 4557; OIDC for the 10+-staff firms that expect SSO (standards.md). |
| Secrets / token encryption | **Application-layer AES-256-GCM** via a KMS-backed data key | `integrations.oauth_access_token`/`oauth_refresh_token` and any sensitive column must be encrypted at rest beyond disk encryption (IRS 4557, FTC Safeguards). |
| Containerisation | **Docker + docker-compose** (Helm chart later) | Self-hostable is a stated deployment mode; compose bundles api, worker, web, postgres, redis, minio. |
| Testing | **Vitest** (unit/integration) + **Supertest** (HTTP) + **Playwright** (E2E) + **Testcontainers** (real Postgres/Redis) | Compliance logic (sign-off trails, independence, audit) needs real-DB integration tests, not just mocks. |
| Code quality | **ESLint + Prettier + TypeScript strict** + **tsc --noEmit** in CI | Standard TS ecosystem; strict mode catches null/tenant-id mistakes early. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | Shares `@app/domain` types and `@app/db` schema across api, worker, and web packages. |
| Webhooks (outbound) | **CloudEvents 1.0 envelope, HMAC-signed** | standards.md recommends a standardised webhook model; CloudEvents matches the `audit_log` attribute naming already in data-model-1. |
| API spec | **OpenAPI 3.1.0** auto-generated, published | Direct response to the documented-API gap; mirrors Karbon's OpenAPI 3.1 reference. |
| Observability | **OpenTelemetry → OTLP**; structured JSON logs (pino) | SOC 2 Availability + continuous-monitoring evidence. |

### Project Structure

```
accounting-practice-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml           # api, worker, web, postgres, redis, minio
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── db/                      # Drizzle schema + migrations (single source of DDL)
│   │   ├── src/
│   │   │   ├── schema/          # one file per domain area (firms.ts, clients.ts, ...)
│   │   │   ├── index.ts         # drizzle client factory (tenant-scoped helper)
│   │   │   └── seed/            # system workflow templates, tax_deadlines reference data
│   │   ├── drizzle.config.ts
│   │   └── migrations/
│   ├── domain/                  # framework-free domain types, enums, value objects, Zod schemas
│   │   └── src/
│   │       ├── enums.ts         # status enums mirroring DB CHECK constraints
│   │       ├── events.ts        # CloudEvents type registry (ce_type catalogue)
│   │       └── schemas/         # Zod request/response schemas, shared with web
│   ├── ai/                      # LLM provider abstraction + prompt templates + structured extractors
│   │   └── src/
│   │       ├── provider.ts      # AiProvider interface; AnthropicProvider, OpenAiProvider
│   │       ├── prompts/         # engagement-letter.ts, billing-analysis.ts, workflow-gen.ts, ...
│   │       └── tasks/           # typed task fns: draftEngagementLetter(), analyzeBilling(), ...
│   └── config/                  # shared eslint/tsconfig/prettier
├── apps/
│   ├── api/                     # NestJS HTTP API + public OpenAPI
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/          # tenant guard, roles guard, audit interceptor, crypto, errors
│   │       ├── auth/            # OIDC, JWT, MFA, sessions
│   │       └── modules/         # firms, staff, clients, services, workflows, engagements,
│   │                            # tasks, engagement-letters, esign, time, billing, invoices,
│   │                            # documents, email, deadlines, independence, signoffs,
│   │                            # tax-returns, integrations, portal, webhooks, ai
│   ├── worker/                  # BullMQ processors (email sync, llm jobs, deadlines, recurrence)
│   │   └── src/
│   │       ├── main.ts
│   │       └── processors/
│   └── web/                     # Next.js 16 firm app + client portal
│       └── src/app/
│           ├── (firm)/          # staff-authenticated dashboard
│           └── (portal)/        # client_contact-authenticated portal
└── tests/
    ├── integration/             # Testcontainers Postgres+Redis suites
    └── e2e/                     # Playwright
```

The structure is additive: each phase fills in `packages/db/src/schema/*`, an `apps/api/src/modules/*` module, optional `apps/worker/src/processors/*`, and `apps/web` routes without restructuring earlier work.

---

## Phase 1: Foundation, Tenancy & Schema Core

### Purpose
Establish the monorepo, database, and the cross-cutting primitives every later phase depends on: multi-tenant isolation, the audit log, encryption, and the firm/staff/client core of the schema. Nothing domain-specific ships yet, but after this phase a firm and its staff and clients exist, every request is tenant-scoped and audited, and the compliance backbone (`audit_log`, encryption) is in place.

### Tasks

#### 1.1 — Monorepo, tooling, and container scaffolding

**What**: Stand up the pnpm/Turborepo workspace with `db`, `domain`, `config`, `api`, `worker`, `web` packages and a docker-compose dev stack.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build`, `lint`, `test`, `typecheck` with dependency graph (`db`→`domain`→apps).
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, `api`, `worker`, `web`. Healthchecks on postgres/redis; api/worker `depends_on` healthy.
- `.env.example` with: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`/`S3_BUCKET`/`S3_ACCESS_KEY`/`S3_SECRET_KEY`, `JWT_SIGNING_KEY`, `DATA_ENCRYPTION_KEY` (base64 32 bytes), `AI_PROVIDER`, `ANTHROPIC_API_KEY`, `APP_BASE_URL`, `PORT` (default 3001), `WEB_PORT` (default 3000).
- TypeScript `strict: true`, `noUncheckedIndexedAccess: true` in shared `config/tsconfig.base.json`.

**Testing**:
- `Unit: turbo build` — all packages compile, `tsc --noEmit` passes.
- `Integration: docker compose up` — postgres/redis/minio reach healthy; `GET /health` on api returns `200 {status:"ok"}`.
- `Unit: lint` — ESLint+Prettier pass on empty scaffold.

#### 1.2 — Database client, migration tooling, and partitioned audit log

**What**: Drizzle schema package with the firm/staff core tables plus the range-partitioned `audit_log`, and a tenant-scoped DB client helper.

**Design**:
- Schema files port DDL from data-model-suggestion-1 verbatim for: `firms`, `offices`, `staff` (with `UNIQUE(firm_id, email)`, role CHECK), and `audit_log` (`PARTITION BY RANGE (ce_time)`).
- `audit_log` migration creates monthly partitions for the current and next month; a `ensureAuditPartitions()` maintenance function (called by a worker cron in Phase 9) creates upcoming partitions.
- Tenant-scoped client helper:
  ```ts
  interface TenantContext { firmId: string; actorId: string | null; actorType: 'staff'|'client'|'system'|'integration'; actorIp?: string; }
  function forTenant(ctx: TenantContext): Database // every query auto-filters firm_id where column exists
  ```
- `crypto.ts`: `encryptColumn(plaintext): string` / `decryptColumn(ciphertext): string` using AES-256-GCM with `DATA_ENCRYPTION_KEY`; output format `v1:<iv>:<tag>:<ciphertext>` (base64). Used by `integrations` tokens later.
- Enum constants in `packages/domain/src/enums.ts` mirror every CHECK constraint (single source; DB and Zod both reference these).

**Testing**:
- `Integration (Testcontainers): drizzle migrate` — all tables created; `\d audit_log` shows partitioned table with 2 partitions.
- `Unit: encryptColumn → decryptColumn` round-trips; tampered ciphertext (flip one tag byte) → throws `DecryptionError`.
- `Unit: enum arrays match DB CHECK lists` — test asserts `STAFF_ROLES` equals the SQL CHECK set (fixture comparison) to prevent drift.
- `Integration: insert audit_log row dated next month` lands in the correct partition.

#### 1.3 — Tenant isolation guard, roles guard, and audit interceptor

**What**: NestJS cross-cutting middleware enforcing OWASP A01 (multi-tenant isolation) and writing CloudEvents audit rows for every mutating request.

**Design**:
- `TenantGuard`: extracts `firmId` from the authenticated principal (JWT claim, Phase 2), attaches `TenantContext` to the request. Rejects any route param `:firmId` that does not match the principal's firm → `403 TENANT_MISMATCH`.
- `RolesGuard` + `@Roles('partner','manager',...)` decorator: checks `staff.role`.
- `AuditInterceptor`: after a successful mutating handler (POST/PATCH/PUT/DELETE), writes an `audit_log` row:
  ```ts
  { firm_id, ce_source:`/firms/${firmId}${req.path}`, ce_type:`${resourceType}.${action}`,
    ce_time:now, ce_specversion:'1.0', actor_type, actor_id, actor_ip,
    resource_type, resource_id, action, detail:{changedFields} }
  ```
- Resource type/action derived from a per-controller `@Audit({resourceType:'engagement', action:'updated'})` decorator.
- Standard error envelope: `{ error: { code, message, details? } }`; global `HttpExceptionFilter`.

**Testing**:
- `Integration (mocked auth): GET /firms/{otherFirmId}/clients` with firm A token → `403 TENANT_MISMATCH`, no rows returned.
- `Integration: PATCH a firm record` → exactly one `audit_log` row with `ce_type='firm.updated'`, correct `actor_id`, `detail.changedFields`.
- `Unit: RolesGuard` — staff role on a partner-only route → `403`.
- `Integration: failed handler (validation error)` → no audit row written.

#### 1.4 — Firms, offices, staff CRUD + firm bootstrap

**What**: First domain module: create a firm with its first partner (bootstrap), manage offices and staff.

**Design**:
- Endpoints (all under `/v1`):
  - `POST /firms` (public bootstrap) → `{ firm, partner }`; creates firm + first `staff` with role `partner`, `can_sign_engagements=true`.
  - `GET/PATCH /firms/{firmId}`
  - `POST/GET/PATCH/DELETE /firms/{firmId}/offices`
  - `POST/GET/PATCH /firms/{firmId}/staff` (DELETE → soft `is_active=false`; staff with assigned engagements cannot be hard-deleted — DB FK).
- Zod schemas in `packages/domain` validate request bodies; `default_hourly_rate_cents`, `fiscal_year_end_month` defaults applied.
- `data_residency`, `is_public_accounting` settable on firm; `is_public_accounting` gates the Phase 8 independence module.

**Testing**:
- `Integration: POST /firms` → firm + partner persisted; `GET` returns them.
- `Integration: create staff with duplicate email in same firm` → `409 DUPLICATE_STAFF`.
- `Integration: same email in two different firms` → both succeed (tenant scoping).
- `Unit: invalid fiscal_year_end_month=13` → `400` with field name.
- `Integration: soft-delete staff` → `is_active=false`, still returned with `?includeInactive=true`.

---

## Phase 2: Authentication, MFA & Sessions

### Purpose
Make the system usable securely. Implements OIDC-capable login, self-issued JWTs, mandatory MFA (IRS Pub 4557 / NIST 800-63B AAL2), session/refresh management, and a separate authentication realm for client-portal contacts. After this phase, staff and client contacts can authenticate and all guards from Phase 1 have a real principal.

### Tasks

#### 2.1 — Staff authentication, JWT issuance, refresh & revocation

**What**: Password + JWT auth for staff, with refresh tokens and Redis-backed revocation.

**Design**:
- `staff` gains auth columns (migration): `password_hash TEXT`, `mfa_secret TEXT` (encrypted), `mfa_enabled BOOLEAN`, `last_login_at TIMESTAMPTZ`.
- Argon2id for password hashing.
- `POST /auth/login` `{email,password}` → if MFA enabled returns `{ mfaRequired:true, mfaToken }`; else `{ accessToken, refreshToken }`.
- Access JWT (15 min): claims `{ sub:staffId, firmId, role, type:'staff' }`, signed with `JWT_SIGNING_KEY` (RS256, rotateable). Refresh token (30 d) stored hashed in Redis `refresh:{jti}` → revocable.
- `POST /auth/refresh`, `POST /auth/logout` (revokes jti).
- Account lockout: 5 failed attempts → 15-min lock (Redis counter).

**Testing**:
- `Integration: login with correct creds (no MFA)` → valid JWT decodable with expected claims.
- `Integration: 5 wrong passwords` → 6th returns `423 LOCKED` even with correct password.
- `Integration: refresh after logout` → `401 TOKEN_REVOKED`.
- `Unit: expired access token` → `401`.

#### 2.2 — MFA (TOTP) enrolment and verification

**What**: TOTP enrolment/verification satisfying AAL2.

**Design**:
- `POST /auth/mfa/enrol` (authenticated) → `{ secret, otpauthUrl, qrSvg }`; stores encrypted `mfa_secret`, `mfa_enabled=false` until confirmed.
- `POST /auth/mfa/confirm` `{code}` → sets `mfa_enabled=true`, returns 8 one-time recovery codes (hashed).
- `POST /auth/mfa/verify` `{mfaToken, code}` → issues access/refresh tokens.
- Firm setting `require_mfa` (default true) blocks token issuance to staff without MFA on protected routes.

**Testing**:
- `Unit: TOTP verify` with `otplib` deterministic time → valid code passes, off-by-one window passes, stale code fails.
- `Integration: full enrol→confirm→verify` flow issues tokens.
- `Integration: recovery code` consumes once; reuse → `401`.

#### 2.3 — Enterprise SSO (OIDC) + portal contact auth realm

**What**: OIDC login for staff via external IdP, and a separate magic-link/password realm for `client_contacts`.

**Design**:
- OIDC: `.well-known` discovery, Authorization Code + PKCE. `firms.settings.sso = { issuer, clientId, clientSecret(enc), allowedDomains[] }`. On callback, match `email` to a `staff` row (auto-provision optional). Standard ID-token (JWT) validation.
- Portal realm: `POST /portal/auth/request-link` `{email}` → emails a signed, single-use magic link (15-min TTL). Token claims `{ sub:clientContactId, clientId, firmId, type:'client' }`. Optional password set after first login.
- `RolesGuard` extended: `type:'client'` principals can reach only `/portal/*` routes (enforced in guard).

**Testing**:
- `Integration (mocked IdP): OIDC callback` with valid code → staff token issued; unknown email + auto-provision off → `403`.
- `Integration: portal magic link` single-use; second use → `401`.
- `Integration: client token on /v1/clients` (staff route) → `403`.

---

## Phase 3: Clients, Service Types & Workflow Templates

### Purpose
Build the client CRM and the template side of the template-instance hierarchy. After this phase a firm can model its clients (with custom fields, tags, assigned partner/manager) and define reusable workflow templates per service type — the prerequisite for instantiating engagements in Phase 4.

### Tasks

#### 3.1 — Client CRM and contacts

**What**: CRUD for `clients` and `client_contacts` with tags, custom fields, and assignment.

**Design**:
- Port `clients` and `client_contacts` DDL from data-model-1. Endpoints:
  - `POST/GET/PATCH /firms/{firmId}/clients` (`GET` supports filters: `client_type`, `tag`, `is_active`, `assigned_partner_id`, full-text `q` on name; pagination `?cursor&limit`).
  - `DELETE` blocked if open engagements exist → `409 CLIENT_HAS_OPEN_ENGAGEMENTS` (rely on FK + explicit check).
  - `POST/GET/PATCH/DELETE /firms/{firmId}/clients/{clientId}/contacts`.
- `ssn_last_four` validated to exactly 4 digits; full SSN rejected (`400` if a 9-digit value posted to that field).
- `custom_fields JSONB` validated against a per-firm `firms.settings.client_custom_field_defs` schema (Zod-built dynamically).

**Testing**:
- `Integration: create client with tags + custom_fields` → GIN tag filter `?tag=vip` returns it.
- `Integration: post 9-digit ssn_last_four` → `400`.
- `Integration: delete client with open engagement` → `409`.
- `Integration: custom field not in firm defs` → `400 UNKNOWN_CUSTOM_FIELD`.

#### 3.2 — Service types and workflow templates (+ system seed templates)

**What**: `service_types`, `workflow_templates`, `workflow_template_steps`, plus a seeded library of system templates.

**Design**:
- Port DDL. `service_types.service_level` (SSARS 21) drives engagement-letter documentation later.
- Template steps carry `default_assignee_role`, `is_client_task`, `requires_sign_off`, `depends_on_step`, `auto_reminder_days`, `estimated_minutes`.
- Seed (`packages/db/src/seed`): system templates (`is_system_template=true`, copied into a firm on demand) for **1040 tax prep**, **monthly bookkeeping**, **payroll**, **audit**, **review**, **compilation** — each with ordered steps mirroring the cross-cutting table-stakes flows from features.md.
- `POST /firms/{firmId}/workflow-templates/from-system/{systemTemplateId}` clones a system template into the firm (editable copy).
- Endpoints for CRUD of templates and steps; reorder via `PATCH .../steps` with full step array (validates contiguous `step_number`, valid `depends_on_step`).

**Testing**:
- `Integration: clone 1040 system template` → firm gets template + steps with correct order and roles.
- `Unit: step with depends_on_step pointing to non-existent step` → `400 INVALID_DEPENDENCY`.
- `Unit: duplicate step_number` → `400`.
- `Integration: deactivate template in use` allowed; new engagements cannot select it (`is_active=false` filtered).

#### 3.3 — AI-assisted workflow generation from natural language

**What**: MVP AI feature — generate a workflow template (steps) from a natural-language service description (Financial Cents / Client Hub "Magic Workflow" parity).

**Design**:
- `POST /firms/{firmId}/workflow-templates/ai-generate` `{ description, serviceTypeId? }` → returns a **draft** (not persisted) template + steps for review; firm confirms via the normal create endpoint.
- Implemented in `packages/ai/src/tasks/generateWorkflow.ts` using `generateObject` with a Zod output schema:
  ```ts
  const WorkflowDraft = z.object({
    name: z.string(),
    steps: z.array(z.object({
      step_number: z.number().int(),
      name: z.string(),
      description: z.string(),
      default_assignee_role: z.enum(STAFF_ROLES_PLUS_CLIENT),
      is_client_task: z.boolean(),
      requires_sign_off: z.boolean(),
      estimated_minutes: z.number().int().optional(),
      depends_on_step: z.number().int().nullable(),
      auto_reminder_days: z.number().int().nullable(),
    }))
  });
  ```
- System prompt (in `packages/ai/src/prompts/workflow-gen.ts`) instructs: produce ordered accounting-firm steps; mark client-supplied-document steps `is_client_task=true`; mark partner sign-off steps `requires_sign_off=true`; never invent regulatory deadlines.
- Runs as a synchronous request (short) but routed through the AI provider abstraction with a 30 s timeout; long generations fall back to `202` + a BullMQ job id (Phase 9 pattern).

**Testing**:
- `Integration (mocked AI provider): "monthly bookkeeping for a restaurant client"` → draft has ≥4 ordered steps, contiguous `step_number`, at least one `is_client_task=true`.
- `Unit: provider returns malformed object` → `502 AI_OUTPUT_INVALID` (Zod parse failure surfaced cleanly).
- `Integration: generated draft confirmed via create endpoint` → persisted correctly.

---

## Phase 4: Engagements, Tasks & Recurrence (Core Value)

### Purpose
The heart of the product. Instantiate workflow templates as engagements and tasks per client and period, drive the task lifecycle, and generate recurring engagement series with cascade edits (Jetpack "Magic Job"). After this phase the platform delivers its primary table-stakes value: recurring workflow and task management with a firm-wide work view.

### Tasks

#### 4.1 — Engagement creation from template + task generation

**What**: Create an engagement for a client from a workflow template, materialising `engagement_tasks` from `workflow_template_steps`.

**Design**:
- Port `engagements` and `engagement_tasks` DDL. `POST /firms/{firmId}/engagements` `{ clientId, serviceTypeId, workflowTemplateId?, name, periodLabel, periodStart, periodEnd, dueDate, feeType, feeAmountCents, budgetHours, assignedPartnerId, assignedManagerId, assignedPreparerId }`.
- On create, copy each active template step into `engagement_tasks` with: `step_number`, `name`, `description`, `requires_sign_off`, `is_client_task`, `assigned_to_id` resolved from `default_assignee_role` (→ engagement's partner/manager/preparer, or null for `client`/`staff`), `due_date` derived from engagement `due_date` minus offset (see 4.2), status `not_started`.
- Engagement status state machine: `planned → pending_engagement_letter → in_progress → in_review → awaiting_client → completed`, plus `cancelled` from any non-terminal state. Transitions validated server-side; illegal transition → `409 INVALID_STATUS_TRANSITION`.
- `actual_hours` maintained as a derived sum from `time_entries` (Phase 6) via trigger or recompute-on-write.

**Testing**:
- `Integration: create engagement from cloned 1040 template` → N tasks created matching N active steps, statuses `not_started`, client-task assignees null.
- `Integration: illegal transition completed→in_progress` → `409`.
- `Unit: assignee resolution` — `default_assignee_role='manager'` → `assigned_to_id = engagement.assigned_manager_id`.

#### 4.2 — Task lifecycle, dependencies, reminders & client tasks

**What**: Task status transitions with dependency gating, due-date scheduling, and client-task exposure.

**Design**:
- Task state machine: `not_started → in_progress → (awaiting_client | in_review) → completed`; `skipped`, `blocked` reachable; `requires_sign_off=true` tasks cannot reach `completed` until a sign-off (Phase 8) is recorded → `409 SIGN_OFF_REQUIRED`.
- Dependency gating: a task whose `depends_on` (resolved from template `depends_on_step`) is not `completed`/`skipped` cannot leave `not_started` → `409 BLOCKED_BY_DEPENDENCY`.
- Due-date offset: template step due offset = engagement `due_date` minus cumulative step ordering (simple model: each step `due_date = engagement.due_date - (totalSteps - step_number) * defaultStepSpacingDays`, firm-configurable; `auto_reminder_days` drives Phase 9 reminder jobs).
- `PATCH /firms/{firmId}/engagements/{id}/tasks/{taskId}` for status/assignee/notes; client tasks updatable via portal endpoint (Phase 5) by the assigned `client_contact`.
- Endpoints: `GET /firms/{firmId}/tasks?assignee&status&dueBefore` (firm-wide work view feed for the Kanban/agenda dashboard).

**Testing**:
- `Integration: complete task with requires_sign_off and no sign-off` → `409 SIGN_OFF_REQUIRED`.
- `Integration: start task whose dependency is not_started` → `409 BLOCKED_BY_DEPENDENCY`.
- `Integration: complete dependency then start dependent` → succeeds.
- `Integration: completing a task` sets `completed_at`, `completed_by_id`, writes audit row `engagement.task.completed`.

#### 4.3 — Recurring engagement series + cascade edits ("Magic Job")

**What**: Recurring engagements via RRULE, with edits to the series cascading to future instances only.

**Design**:
- `is_recurring`, `recurrence_rule` (RFC 5545 RRULE), `parent_engagement_id` link the series. The first engagement is the series parent.
- A BullMQ repeatable job `generateRecurringEngagements` (daily) materialises the next instance when within a firm-configurable lead window (default 30 days before period start), copying template + assignments from the parent and linking `parent_engagement_id`.
- Cascade edit: `PATCH /firms/{firmId}/engagements/{parentId}/series` `{ changes }` updates the parent and all **future** (not yet started, `period_start > today`) child engagements and their tasks; completed/in-progress instances are untouched. Returns counts `{ updated, skipped }`.
- Generation is idempotent: `UNIQUE(parent_engagement_id, period_start)` prevents duplicates.

**Testing**:
- `Integration: monthly RRULE, run generator at month boundary` → exactly one next instance, correct `period_label`/`due_date`.
- `Integration: run generator twice` → no duplicate (idempotent).
- `Integration: cascade edit on series with one completed + two future instances` → 2 future updated, 1 completed skipped; `{updated:2, skipped:1}`.

#### 4.4 — Firm work views (Kanban / capacity feed API)

**What**: Aggregate read endpoints powering the firm dashboard and staff capacity.

**Design**:
- `GET /firms/{firmId}/board?groupBy=status|assignee|client` → engagements grouped for Kanban columns.
- `GET /firms/{firmId}/capacity?weekOf=` → per-staff `{ targetBillableHours, scheduledHours (sum of estimated_minutes on open assigned tasks), utilizationPct }`.
- `GET /firms/{firmId}/agenda?staffId&date=` → prioritised task list (Pascal "Agenda Dashboard" parity): tasks due today/overdue ordered by engagement `priority` then `due_date`.

**Testing**:
- `Integration: board groupBy=status` → engagements bucketed by status with counts.
- `Integration: capacity` — staff with 3 open tasks of 60 min each and 20h target → `scheduledHours=3`, `utilizationPct≈15`.
- `Integration: agenda` orders overdue before due-today, higher priority first.

---

## Phase 5: Client Portal, Documents & Secure Messaging

### Purpose
Deliver the client-facing surface: document upload/download (S3, encrypted, organised by client/tax-year/engagement), the shared task view (Uku-style unified checklist), and secure messaging. After this phase clients can interact with the firm without email, and document management — a table-stakes feature — is complete.

### Tasks

#### 5.1 — Document storage, versioning & access control

**What**: `documents` table + S3-backed upload/download with checksums, versioning, and client-visibility control.

**Design**:
- Port `documents` DDL. Upload flow: `POST /firms/{firmId}/documents/upload-url` → presigned S3 PUT URL + `documentId`; client/staff uploads directly to S3 (SSE-KMS); `POST /firms/{firmId}/documents/{id}/finalize` records `file_size_bytes`, `mime_type`, `checksum_sha256` (verified server-side via S3 head/ETag or a finalize-time hash).
- `category`, `tax_year`, `is_client_visible`, `parent_document_id` (version chain). New version → `version+1`, links `parent_document_id`.
- Download: `GET /firms/{firmId}/documents/{id}/download-url` → short-TTL presigned GET; staff always allowed (tenant scoped); client contacts only if `is_client_visible=true` AND document's `client_id` matches their client.
- Every download writes `audit_log` `document.accessed` (IRS 4557 access evidence).

**Testing**:
- `Integration: upload→finalize→download` round-trip; checksum mismatch on finalize → `400 CHECKSUM_MISMATCH`.
- `Integration: client requests non-client-visible doc` → `403`.
- `Integration: client requests another client's doc` → `403`/`404`.
- `Integration: download` writes `document.accessed` audit row with actor.

#### 5.2 — Client portal task view & client task completion

**What**: Portal endpoints letting a client contact see and complete their assigned tasks and upload requested documents.

**Design**:
- `GET /portal/engagements` (client realm) → engagements for the contact's client where any task `is_client_task=true`; each with the client-visible task checklist (Uku unified-view: same task entity, filtered to client tasks).
- `PATCH /portal/tasks/{taskId}` → client may set `in_progress`→`completed` on tasks where `assigned_client_contact_id` matches; completing a client task that gates a staff task unblocks it.
- `POST /portal/tasks/{taskId}/documents` → attach uploaded document (forces `is_client_visible=true`, `uploaded_by_contact_id`).

**Testing**:
- `Integration: client completes their client-task` → status `completed`, dependent staff task unblocks.
- `Integration: client patches a non-client task` → `403`.
- `Integration: client sees only own client's engagements`.

#### 5.3 — Secure portal messaging

**What**: `portal_messages` threaded messaging between staff and client contacts, optionally tied to an engagement.

**Design**:
- Port DDL. `POST /firms/{firmId}/clients/{clientId}/messages` (staff) and `POST /portal/messages` (client). `GET` both sides with unread filter (`idx_portal_messages_unread`). Mark-read endpoint sets `is_read`, `read_at`.
- New-message notification queued (Phase 9) to the counterparty (email + in-app).

**Testing**:
- `Integration: staff→client message then client reply` → ordered thread; counts unread correctly.
- `Integration: client posts message for another client` → `403`.

---

## Phase 6: Time Tracking & Billing

### Purpose
Complete the billing table-stakes loop: capture time against engagements, generate draft invoices from unbilled time, and collect payment via Stripe. After this phase a firm can run time → invoice → payment without leaving the platform.

### Tasks

#### 6.1 — Time entries (manual + timer)

**What**: `time_entries` CRUD with timer support and approval workflow.

**Design**:
- Port DDL. `POST /firms/{firmId}/time-entries` `{ staffId, engagementId, clientId, date, durationMinutes, description, isBillable, source }`; `rate_cents` defaults to staff `hourly_rate_cents`, `amount_cents = round(durationMinutes/60 * rate_cents)`.
- Timer: `POST /time-entries/timer/start` / `/stop` (Redis-held running timer per staff) → creates a `source='timer'` entry on stop.
- Status flow `draft → approved → invoiced | written_off`. On any write, recompute parent engagement `actual_hours`.
- `GET /firms/{firmId}/time-entries?engagementId&staffId&status&from&to`.

**Testing**:
- `Unit: amount_cents` for 90 min @ $200/h → `30000`.
- `Integration: start+stop timer` creates one entry with correct duration (mock clock).
- `Integration: approve then attempt edit duration` → `409 ENTRY_LOCKED` (approved entries immutable except write-off).
- `Integration: adding time` updates engagement `actual_hours`.

#### 6.2 — Invoice generation, line items & status

**What**: `invoices`, `invoice_line_items`, `payments` with draft-from-unbilled-time generation.

**Design**:
- Port DDL. `POST /firms/{firmId}/invoices/generate` `{ clientId, engagementIds[], asOfDate }` → collects `approved`+`is_billable` time entries, groups into line items (`line_type='time'`) by engagement, computes `subtotal_cents`/`total_cents`/`balance_due_cents`, sets entries `status='invoiced'` + `invoice_id`. Returns a `draft` invoice.
- Invoice status machine: `draft → pending_review → sent → (partially_paid → paid | overdue) | written_off | void`. `invoice_number` per-firm sequence (`UNIQUE(firm_id, invoice_number)`).
- Manual line items (`fixed_fee`, `expense`, `adjustment`) addable while draft.

**Testing**:
- `Integration: generate invoice from 3 approved entries` → 3 (or grouped) line items, correct totals, entries flipped to `invoiced`.
- `Integration: generate includes only approved+billable` (drafts/non-billable excluded).
- `Integration: duplicate invoice_number in firm` → `409`.

#### 6.3 — Stripe payment integration

**What**: Collect payments via Stripe and reconcile to `payments`/`invoices`.

**Design**:
- `integrations` row provider `stripe` (Phase 7 OAuth/Connect pattern; for MVP a firm-level secret key encrypted via `crypto.ts`).
- `POST /firms/{firmId}/invoices/{id}/payment-link` → Stripe Checkout/PaymentIntent; on `payment_intent.succeeded` webhook → insert `payments` row, increment `amount_paid_cents`, recompute `balance_due_cents`, transition invoice to `partially_paid`/`paid`, set `paid_at`.
- Webhook signature verified with Stripe signing secret; idempotency via `stripe_payment_id` unique check.

**Testing**:
- `Integration (mocked Stripe): payment-link` returns URL with correct amount.
- `Integration: succeeded webhook` → payment recorded, invoice `paid` when fully covered.
- `Integration: replayed webhook (same payment id)` → no double-count (idempotent).
- `Integration: webhook with bad signature` → `400`, no mutation.

---

## Phase 7: External Integrations (QBO, Xero, Email, DocuSign)

### Purpose
Connect the platform to the systems firms already run. Implements the OAuth 2.0 connection framework and the high-value integrations: QuickBooks Online / Xero (transactional sync + uncategorised-transaction-to-task), Gmail / Microsoft 365 email surfacing, and DocuSign e-signature. After this phase the platform participates in the firm's existing toolchain rather than replacing it wholesale.

### Tasks

#### 7.1 — OAuth 2.0 integration framework

**What**: Generic per-firm OAuth connect/refresh/disconnect with encrypted token storage.

**Design**:
- Port `integrations` DDL. Tokens stored via `encryptColumn`. `GET /firms/{firmId}/integrations/{provider}/connect` → provider authorize URL (PKCE where supported: Xero, Google, Microsoft). Callback exchanges code, stores tokens + `external_tenant_id` (QBO `realmId`, Xero `tenantId`) + `scopes`.
- `IntegrationTokenService.getAccessToken(firmId, provider)` transparently refreshes when within 5 min of `token_expires_at`; on refresh failure sets `status='expired'` and emits `integration.expired` audit event.
- `DELETE .../integrations/{provider}` revokes + clears tokens.

**Testing**:
- `Integration (mocked provider): connect callback` stores encrypted tokens; raw token never appears in DB column plaintext (assert ciphertext prefix `v1:`).
- `Unit: getAccessToken near expiry` triggers refresh; refresh 400 → `status='expired'`.

#### 7.2 — QuickBooks Online & Xero sync + uncategorised-transaction tasks

**What**: Pull client financial data and convert uncategorised transactions into client tasks (Client Hub parity).

**Design**:
- Map a platform `client` to a QBO `realmId`/Xero `tenantId` (`clients.custom_fields.qbo_realm_id` or a dedicated mapping; store on client). 
- Worker job `syncAccounting` (scheduled + on-demand `POST .../clients/{id}/accounting/sync`): fetches transactions; for each uncategorised expense/deposit, upsert an `engagement_task` (or standalone client task) `is_client_task=true` named e.g. `Categorise transaction <date> <amount>`. Idempotent on external transaction id (store in task `notes`/metadata).
- Read endpoints expose synced summary (counts of uncategorised, last sync time) for the dashboard and health scoring (Phase 8).

**Testing**:
- `Integration (mocked QBO API): sync with 2 uncategorised txns` → 2 client tasks created; re-sync → no duplicates.
- `Integration: categorised txn` → no task.
- `Integration: provider 401` → integration marked `expired`, sync job fails gracefully (retried by BullMQ).

#### 7.3 — Email surfacing (Gmail / Microsoft 365)

**What**: Sync email metadata and associate messages with clients/engagements (Karbon triage parity, metadata-only per data-model-1).

**Design**:
- Port `email_messages` DDL (metadata + snippet only; bodies fetched on demand from provider). Worker `syncEmail` uses Gmail API / Graph change notifications (webhook) to ingest new messages; match `from_address`/`to_addresses` against `client_contacts.email` to set `client_id`; optionally infer `engagement_id` from subject/thread heuristics.
- `GET /firms/{firmId}/email?clientId&untriaged=true` triage feed; `POST .../email/{id}/triage` `{engagementId}` links message and sets `is_triaged`.
- On-demand body fetch: `GET .../email/{id}/body` proxies provider fetch (not stored).

**Testing**:
- `Integration (mocked Gmail): ingest message from known contact` → `email_messages` row with `client_id` set, `is_triaged=false`.
- `Integration: duplicate external_message_id` → no duplicate (`UNIQUE`).
- `Integration: triage assigns engagement_id`, writes audit row.

#### 7.4 — DocuSign e-signature

**What**: `e_signatures` lifecycle for engagement letters and Form 8879 via DocuSign.

**Design**:
- Port `e_signatures` DDL. `POST /firms/{firmId}/esign` `{ documentId|engagementLetterId, signers[], formType }` → creates DocuSign envelope; rows created `status='pending'`.
- DocuSign Connect webhook updates `status`, `signed_at`, `ip_address`, `user_agent`, `signature_hash` (SHA-256 of completed doc). On engagement-letter completion, transition the engagement-letter status (Phase 8) and unblock the gated engagement (`pending_engagement_letter → in_progress`).
- Audit `esignature.signed`.

**Testing**:
- `Integration (mocked DocuSign): create envelope` → pending e_signature rows for each signer.
- `Integration: completed webhook` → `status='signed'`, hash + IP recorded.
- `Integration: declined webhook` → `status='declined'`, engagement letter `declined`.

---

## Phase 8: Compliance — Engagement Letters, Sign-Offs, Independence, Returns

### Purpose
Implement the public-accounting compliance backbone that no incumbent fully offers: SSARS/SSAE sign-off trails, independence and conflict-of-interest tracking, engagement-letter lifecycle (with the AI drafting differentiator), and MeF tax-return status tracking. After this phase the platform credibly serves audit/attestation firms.

### Tasks

#### 8.1 — Engagement letters lifecycle + AI drafting

**What**: `engagement_letters` with status machine, SSARS-21 service-level documentation, and AI drafting from prior-year data.

**Design**:
- Port `engagement_letters` DDL. Status: `draft → ai_drafted → pending_review → sent → viewed → signed | declined | expired | superseded`. `service_level` enforces SSARS-21-appropriate template/clauses.
- AI drafting: `POST /firms/{firmId}/engagements/{id}/letter/ai-draft` → `packages/ai/src/tasks/draftEngagementLetter.ts` using prior engagement(s) for the client (`ai_generation_context` records source engagement ids + extracted scope). `generateObject` output: `{ serviceScope, feeDescription, feeAmountCents, bodyMarkdown }`. Prompt forbids fabricating fees not present in inputs; partner must review (`pending_review`).
- Sending links to DocuSign (7.4). Engagement cannot move past `pending_engagement_letter` until letter `signed`.

**Testing**:
- `Integration (mocked AI): ai-draft from a prior-year engagement` → letter `ai_drafted`, `ai_generation_context` lists source engagement.
- `Integration: send letter` → DocuSign envelope created, status `sent`.
- `Integration: signed letter` unblocks engagement to `in_progress`.
- `Unit: illegal transition signed→draft` → `409`.

#### 8.2 — Sign-off trails (SSARS 21 / SSAE 18)

**What**: `sign_off_trails` capturing attestation sign-offs referencing professional standard sections.

**Design**:
- Port DDL. `POST /firms/{firmId}/engagements/{id}/sign-offs` `{ signOffType, standardReference, conclusion, supportingDocumentId? }` — only staff with `can_sign_engagements`/appropriate role; records `signed_by_id`, `signed_at`.
- A task with `requires_sign_off=true` reaching `completed` requires a corresponding sign-off (links 4.2 `SIGN_OFF_REQUIRED`).
- `GET .../sign-offs` returns the immutable chain (no update/delete; corrections via superseding entry).

**Testing**:
- `Integration: partner records review_report sign-off` → row persisted with `standard_reference='SSARS 21 AR-C 90'`.
- `Integration: staff without can_sign_engagements` → `403`.
- `Integration: attempt to delete a sign-off` → `405`/`403` (immutable).

#### 8.3 — Independence records & conflict checks

**What**: `independence_records` and `conflict_checks` for public-accounting firms (gated by `firms.is_public_accounting`).

**Design**:
- Port DDL. `POST/GET/PATCH .../independence-records` (per client and optional staff; `threat_level`, `safeguards_applied`, `effective/expiration`).
- `POST .../conflict-checks` `{ clientId? | prospectiveClientName, prospectiveClientEin? }` → scans existing clients/relationships and `independence_records` for matches (name/EIN fuzzy + financial-interest matches), returns `{ result, conflictsFound[] }`. Acceptance requires `approved_by_id` (partner).
- Engagement acceptance (`engagement_acceptance` sign-off) blocked for public-accounting firms until a `clear`/approved conflict check exists for the client.

**Testing**:
- `Integration: conflict check on client with active financial_interest independence record` → `result='conflict_found'`.
- `Integration: conflict check clear` → `result='clear'`.
- `Integration: public-accounting firm engagement acceptance without conflict check` → `409 CONFLICT_CHECK_REQUIRED`.
- `Integration: non-public firm` → independence endpoints `404`/feature-disabled.

#### 8.4 — Tax-return MeF status tracking

**What**: `tax_returns` tracking e-file lifecycle without being the filing engine.

**Design**:
- Port DDL. CRUD `.../engagements/{id}/tax-returns`. Status machine `not_started → in_preparation → prepared → in_review → approved → e_filed → accepted | rejected | amended | paper_filed`.
- `PATCH .../tax-returns/{id}/mef-ack` `{ submissionId, ackStatus, rejectionCode? }` records `mef_submission_id`, `mef_ack_status`, `mef_ack_timestamp`; `rejected` requires `mef_rejection_code`.
- Linkage: `signer_id` must be staff with `can_sign_returns`.

**Testing**:
- `Integration: record accepted ack` → status `accepted`, timestamp set.
- `Integration: rejected ack without code` → `400`.
- `Integration: approve return with non-signer as signer_id` → `403`.

---

## Phase 9: AI Intelligence, Notifications & Scheduled Jobs

### Purpose
Deliver the remaining AI-native differentiators (deadline intelligence, write-up/write-down billing, client health scoring, anomaly detection) and the notification/scheduling backbone (reminders, recurring generation, partition maintenance) that several earlier phases reference. After this phase the platform's AI value proposition is fully realised.

### Tasks

#### 9.1 — Scheduling backbone, reminders & notifications

**What**: BullMQ repeatable jobs and a notification dispatcher used across the app.

**Design**:
- Repeatable jobs: `generateRecurringEngagements` (4.3), `ensureAuditPartitions` (1.2), `taskReminders` (daily — for tasks with `auto_reminder_days` before due, notify assignee/client), `recomputeHealthScores` (9.3), `pollTaxDeadlines` (9.2), `accrueOverdueInvoices` (flip `sent`→`overdue`).
- `NotificationService.dispatch({firmId, recipient, channel:'email'|'in_app', template, data})`; email via firm email integration or system SMTP; in-app stored for the dashboard.

**Testing**:
- `Integration: taskReminders` with a task due in `auto_reminder_days` → one notification dispatched (mock email).
- `Integration: accrueOverdueInvoices` flips a past-due sent invoice to `overdue`.
- `Integration: ensureAuditPartitions` creates next-month partition.

#### 9.2 — Deadline intelligence

**What**: Maintain `tax_deadlines` reference data and cascade changes to `client_deadlines`.

**Design**:
- Seed `tax_deadlines` with federal + major state deadlines per tax year. `pollTaxDeadlines` job uses an AI task (`packages/ai/src/tasks/checkDeadlineChanges.ts`) over configured IRS/state source URLs to detect changes; proposed changes require partner approval (no silent mutation) → `deadline.change_proposed` notification.
- `client_deadlines` instantiated per client/engagement from `tax_deadlines`. When a `tax_deadlines.due_date` changes (approved), cascade: update linked `client_deadlines` (status `upcoming`/`at_risk`) and shift the engagement `due_date`/task due dates for not-started engagements.
- `at_risk` computed when `due_date - today < firm.settings.atRiskWindowDays` (default 14) and engagement not `completed`.

**Testing**:
- `Integration: approve a federal deadline change` → linked client_deadlines + future engagements shift dates; completed ones untouched.
- `Integration (mocked AI): poll detects no change` → no proposals.
- `Unit: at_risk computation` boundary at window edge.

#### 9.3 — Write-up/write-down billing AI & client health scoring

**What**: AI billing recommendations on draft invoices and client health scores.

**Design**:
- Billing: on `POST .../invoices/generate`, optionally call `packages/ai/src/tasks/analyzeBilling.ts` comparing `actual_hours` vs `budget_hours` and historical realisation; output `{ writeUpCents, writeDownCents, notes }` stored in `invoices.ai_write_up_cents`/`ai_write_down_cents`/`ai_billing_notes` and per-entry `ai_suggested_*`. Recommendation only — partner edits before send.
- Health scoring: `recomputeHealthScores` job computes `clients.health_score` (0–100) from weighted signals — communication responsiveness (avg reply time on `portal_messages`/`email_messages`), document timeliness (client-task completion lateness), and AR aging (overdue `invoices`/`balance_due_cents`). Stores `health_score`, `health_score_updated_at`. Deterministic scoring formula (documented), with an optional AI narrative summary.

**Testing**:
- `Integration (mocked AI): generate invoice with analysis` → `ai_write_down_cents` populated when actual >> budget.
- `Unit: health score formula` — client with fast replies, on-time docs, no AR → score ≈ 90+; slow replies + overdue AR → low score.
- `Integration: recomputeHealthScores` updates `health_score_updated_at`.

#### 9.4 — Prior-year comparison & anomaly detection

**What**: Flag unusual changes during tax-prep workflows (quality-control differentiator).

**Design**:
- `POST .../engagements/{id}/anomaly-scan` compares current-year key figures (entered or synced from QBO/Xero) against the client's prior-year engagement/return data; `packages/ai/src/tasks/detectAnomalies.ts` returns `{ anomalies: [{ field, priorValue, currentValue, pctChange, severity, rationale }] }`. Flags above a configurable `pctChange` threshold create review tasks/notifications. Results stored as engagement notes/metadata (no new table — reuses existing structures).

**Testing**:
- `Integration (mocked AI): revenue down 60% vs prior year` → anomaly with `severity='high'` and a generated review task.
- `Integration: within threshold` → no anomalies.

---

## Phase 10: Public API, Webhooks, Hardening & Deployment

### Purpose
Close the "documented public API" gap and ready the platform for production: publish OpenAPI 3.1, ship CloudEvents outbound webhooks, enforce the IRS-4557/FTC/SOC-2 security posture end-to-end, and finalise self-host + managed deployment. After this phase the platform is production-deployable and integration-friendly.

### Tasks

#### 10.1 — Public REST API, OpenAPI 3.1 & API keys

**What**: A stable, documented `/v1` public API with per-firm API keys and rate limiting.

**Design**:
- NestJS Swagger module emits OpenAPI 3.1.0 at `/openapi.json` + Swagger UI at `/docs`; every DTO annotated.
- `api_keys` table (migration): `id, firm_id, name, key_hash, scopes[], last_used_at, created_at, revoked_at`. Auth via `Authorization: Bearer <key>` resolving to a `system`/scoped principal; tenant guard applies.
- Redis token-bucket rate limiting per key (default 600 req/min); `429` with `Retry-After`.

**Testing**:
- `Integration: /openapi.json` validates against OpenAPI 3.1 meta-schema.
- `Integration: API key from firm A` cannot read firm B (`403`).
- `Integration: exceed rate limit` → `429` with `Retry-After`.
- `Integration: revoked key` → `401`.

#### 10.2 — Outbound webhooks (CloudEvents)

**What**: Subscribable, HMAC-signed CloudEvents webhooks for firm events.

**Design**:
- `webhook_subscriptions` table: `id, firm_id, url, event_types[], secret(enc), is_active, created_at`. Events sourced from the same `ce_type` catalogue as `audit_log` (`engagement.task.completed`, `invoice.paid`, `esignature.signed`, ...).
- Worker `deliverWebhook` posts a CloudEvents 1.0 JSON envelope with header `Webhook-Signature: sha256=<hmac>`; retries with exponential backoff (max 6), dead-letters after exhaustion; delivery log retained.

**Testing**:
- `Integration: subscribe to invoice.paid, mark invoice paid` → webhook delivered with valid HMAC + CloudEvents fields.
- `Integration: endpoint returns 500` → retried; after max attempts → dead-lettered.
- `Unit: signature verification` with shared secret.

#### 10.3 — Security hardening (IRS 4557 / FTC / SOC 2 / OWASP)

**What**: End-to-end enforcement of the compliance floor.

**Design**:
- TLS 1.2+ enforced at ingress; HSTS. AES-256 at rest verified (S3 SSE-KMS; `crypto.ts` for sensitive columns). MFA enforced for all staff (Phase 2) — block non-MFA staff on protected routes when `require_mfa`.
- OWASP A01: automated tenant-isolation test suite hitting every `/firms/{firmId}/*` route cross-tenant. Security headers (helmet), input validation (Zod everywhere), rate limiting (10.1). Dependency scanning + `npm audit`/Snyk in CI.
- WISP/data-retention: configurable retention + deletion (GDPR/CCPA erasure) endpoints honouring referential constraints (anonymise rather than hard-delete where audit retention requires).

**Testing**:
- `Integration: cross-tenant matrix` — every protected route returns `403` for a foreign firm token (generated test).
- `Integration: GDPR erasure request` → PII anonymised, audit log retained.
- `Integration: request over plain HTTP` (behind proxy) → redirected/refused.
- `CI: npm audit` high-severity → build fails.

#### 10.4 — Deployment (self-host + managed) & docs

**What**: Production Docker images, Helm chart, migration/seed runbook, and operator docs.

**Design**:
- Multi-stage Dockerfiles (api/worker/web). `docker-compose.prod.yml` and a Helm chart (`api`, `worker`, `web`, plus external Postgres/Redis/S3 references). Startup runs `drizzle migrate` + seed (system templates, tax deadlines) idempotently.
- Health/readiness probes; OpenTelemetry exporter config; backup guidance for Postgres + S3.
- `DEPLOYMENT.md` and `SECURITY.md` (WISP template aligned to IRS Pub 4557).

**Testing**:
- `E2E (Playwright against compose stack): firm signup → create client → engagement from template → client uploads doc via portal → time entry → invoice → pay (Stripe test) → invoice paid` green path passes.
- `Integration: fresh DB startup` runs migrations + seed once; second start is a no-op.
- `Integration: readiness probe` fails when DB unavailable.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Schema Core      ─── required by everything
    │
Phase 2: Auth, MFA & Sessions                   ─── requires 1
    │
Phase 3: Clients, Service Types & Templates     ─── requires 1,2
    │
Phase 4: Engagements, Tasks & Recurrence (CORE) ─── requires 3
    ├── Phase 5: Client Portal, Documents, Msgs  ─── requires 4   (parallel with 6)
    └── Phase 6: Time Tracking & Billing         ─── requires 4   (parallel with 5)
         │
Phase 7: External Integrations (QBO/Xero/Email/DocuSign) ─── requires 5 (docs/portal) & 6 (Stripe)
    │
Phase 8: Compliance (Letters/Sign-offs/Independence/Returns) ─── requires 4; DocuSign from 7.4 for letter sending
    │
Phase 9: AI Intelligence, Notifications & Jobs  ─── requires 4,6,8 (and 7 for QBO/Xero data)
    │
Phase 10: Public API, Webhooks, Hardening, Deploy ─── requires all
```

**Parallelism opportunities**
- Phases **5 and 6** can be built concurrently once Phase 4 lands (distinct modules: portal/documents vs. time/billing).
- Within Phase 7, the four integrations (7.2 QBO/Xero, 7.3 Email, 7.4 DocuSign) can be built in parallel after the 7.1 OAuth framework exists.
- Phase 8 sub-tasks 8.2 (sign-offs), 8.3 (independence), 8.4 (returns) are independent of each other (all depend on Phase 4); 8.1 (letters) additionally needs 7.4.
- Phase 9 AI tasks (9.2 deadlines, 9.3 billing/health, 9.4 anomaly) are independent once the 9.1 scheduling/notification backbone exists.

**MVP cut line**: Phases 1–6 plus 3.3 deliver the features.md "Must-have (MVP)" set (recurring workflow/tasks, client portal + documents + e-sign [e-sign arrives with 7.4], time + draft invoice, client CRM, AI workflow generation). Phases 7–9 deliver the "Should-have (v1.1)" set; Phase 8.3 + 9.4 cover "Nice-to-have (backlog)".

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented per their Design sections.
2. All unit and integration tests for the phase pass (Vitest + Supertest; Testcontainers Postgres/Redis for integration).
3. ESLint + Prettier pass with zero warnings; `tsc --noEmit` (strict) passes across affected packages.
4. New tables have Drizzle migrations checked in and applied cleanly on a fresh database; seed data idempotent.
5. New endpoints appear in the generated OpenAPI 3.1 document with request/response DTOs annotated.
6. Every mutating endpoint writes a correct `audit_log` (CloudEvents) row; every cross-tenant access path is covered by an isolation test.
7. New config options added to `.env.example` and documented.
8. `docker compose up` builds and the affected services reach healthy; the phase's headline capability works end-to-end (demonstrated by at least one integration or E2E test).
9. Any LLM-backed task uses the `packages/ai` provider abstraction with a Zod-validated structured output and a mocked-provider test.
10. Security checks for the phase pass: no plaintext secrets in DB columns, role/tenant guards enforced, inputs Zod-validated.
```
