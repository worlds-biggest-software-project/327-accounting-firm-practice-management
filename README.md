# Accounting Firm Practice Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source practice management platform for accounting firms covering workflow, client management, time tracking, and document management.

Accounting Firm Practice Management is a candidate project to build an open alternative to platforms like Karbon, TaxDome, Canopy, and Uku. It targets solo CPAs, bookkeeping shops, and mid-size firms that need recurring workflow automation, a client portal, time and billing, and document management without locking themselves into expensive proprietary suites.

---

## Why Accounting Firm Practice Management?

- Entry-level commercial tools start at $9–$19/user/month, while full-featured platforms run $50–$80/user/month, and firms typically stack additional document, tax, and portal tools on top — creating a high total cost of ownership.
- Karbon's price point is prohibitive for solo and micro-firms; TaxDome has a steep learning curve and limited workflow customisation; Canopy's modular pricing is opaque; Jetpack Workflow lacks a client portal, billing, and AI features entirely.
- Most incumbents offer limited or undocumented APIs (Financial Cents, Client Hub, Jetpack Workflow, and Pascal Workflow have no published REST API), preventing sophisticated custom integrations.
- No current platform natively supports independence tracking and conflict-of-interest checking required by public accounting and audit firms.
- AI features across incumbents are uneven and often bolted on; an AI-native architecture can embed automation across workflow generation, billing, deadline intelligence, and anomaly detection from the start.

---

## Key Features

### Workflow & Task Management

- Recurring workflow and task management with a template library covering tax prep, bookkeeping, payroll, and audit
- AI-assisted workflow generation from natural language service descriptions
- Recurring task automation assigned to the correct team member at the correct time
- Cascade edits across all future jobs in a recurring series
- Staff capacity management and workload visualisation

### Client Portal & Communication

- Client portal with document upload, secure messaging, and mobile access
- E-signature collection for engagement letters and IRS Form 8879
- Email integration (Gmail and Microsoft 365) surfacing client emails within job context
- Automated client follow-up for outstanding requests
- Client CRM with contact history and custom fields

### Billing, Time & Documents

- Time tracking with draft invoice generation
- AI-powered draft invoices with write-up/write-down recommendations based on time vs. budget comparison
- Document management organised by client, tax year, and engagement
- Proposal and engagement letter generation with approval workflow
- AI engagement letter drafting from prior-year client data and historical engagement scope

### AI & Analytics

- Deadline intelligence: automated adjustment of workflow due dates from IRS and state tax calendar monitoring
- Client health scoring based on communication responsiveness, document timeliness, and AR aging
- Prior-year comparison and anomaly detection during tax preparation workflows
- QuickBooks Online and Xero integration with uncategorised-transaction-to-client-task automation

### Compliance & Public Accounting

- Independence tracking and conflict-of-interest checking for public accounting firms
- Compliant sign-off trails aligned with AICPA SSARS, SSAE, and SAS standards
- IRS e-File alignment (IRS Pub 1345) for integrated tax workflows
- SOC 2 Type II posture for the platform itself

---

## AI-Native Advantage

AI is embedded across the workflow rather than bolted on. The platform drafts engagement letters from prior-year returns and engagement history, monitors IRS and state tax calendar changes to cascade updated due dates across the client portfolio, analyses time entries against budget to produce draft invoices with write-up/write-down recommendations, and flags unusual changes in client financial data during tax preparation. Client health scoring uses communication responsiveness, document delivery timeliness, and receivables aging to surface at-risk relationships before revenue is lost.

---

## Tech Stack & Deployment

Expected to ship as a self-hostable cloud SaaS with optional managed hosting. Integration priorities include QuickBooks Online and Xero (transactional sync, not just invoicing), Gmail and Microsoft 365 email and calendar sync, and a documented public REST API — addressing a clear gap among incumbents whose APIs are limited or undocumented. Standards alignment targets AICPA MAP metrics, SSARS/SSAE/SAS sign-off trails, IRS Pub 1345 e-File requirements, SOC 2 Type II, and GDPR/CCPA data privacy.

---

## Market Context

The accounting practice management market is projected to grow at 10–14% CAGR through 2032, with more than 73% of accounting firms already running some form of practice management solution and roughly 64% preferring cloud-based systems (Coherent Market Insights, 2024; Credence Research, 2024). Incumbents range from $9/user/month (Pascal Workflow) to $59+/user/month (Karbon), with primary buyers including solo CPAs and EAs, small-firm partners, mid-size managing partners, bookkeeping firms, and public accounting firms with independence and engagement-letter requirements.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
