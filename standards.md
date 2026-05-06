# Standards & API Reference

> Project: Accounting Firm Practice Management · Generated: 2026-05-03

---

## Industry Standards & Specifications

### Regulatory & Professional Standards

**IRS Publication 4557 — Safeguarding Taxpayer Data**
- URL: https://www.irs.gov/pub/irs-pdf/p4557.pdf
- Mandatory guidance for all tax professionals who prepare returns for compensation. Requires a Written Information Security Plan (WISP), AES-256 encryption for data at rest and in transit, MFA on all systems accessing taxpayer data, annual security awareness training, documented vendor agreements, and an incident response plan. Violations carry civil penalties up to $46,517 per violation per day and can result in EFIN revocation.

**FTC Safeguards Rule (16 CFR Part 314)**
- URL: https://www.ftc.gov/legal-library/browse/rules/safeguards-rule
- Federal Trade Commission enforcement of GLBA compliance. Applies to all financial institutions including tax preparers. Requires formal information security programmes, access controls, multi-factor authentication, encrypted transmission, continuous monitoring, and documented third-party service provider oversight. Aligned with and cross-referenced by IRS Publication 4557.

**AICPA Statements on Standards for Attestation Engagements (SSAE No. 18)**
- URL: https://www.aicpa-cima.com/resources/download/aicpa-statement-on-standards-for-attestation-engagements-no-18
- Defines documentation requirements for attestation engagements (reviews, examinations, agreed-upon procedures). Practice management software must maintain compliant sign-off trails and preserve engagement documentation in a form satisfying SSAE 18 evidence requirements.

**AICPA Statements on Standards for Accounting and Review Services (SSARS 21)**
- URL: https://nsacct.org/ssars-21-part-1-the-preparation/
- Establishes the bright line between preparation services and compilation/review reporting services. Engagement letter requirements and documentation standards differ by service level. Workflow software should enforce appropriate documentation templates per service type.

**IRS Modernized e-File (MeF) Programme**
- URL: https://www.irs.gov/e-file-providers/modernized-e-file-overview
- Web-based system for electronic filing of individual, corporate, partnership, exempt organisation, and excise tax returns using XML format. Uses WS-I (Web Services-Interoperability) security standards for Application-to-Application (A2A) filing. Software developers must test against the MeF Assurance Testing System (ATS). For tax year 2026, WSDL version R10.9 applies. XML schema packages and business rules published annually at the IRS e-file providers portal.
- Technical documentation: https://www.irs.gov/e-file-providers/modernized-e-file-mef-user-guides-and-publications

**PCAOB Auditor Independence Rules**
- URL: https://pcaobus.org/Registration/Firms/Details/540
- Relevant for CPA firms performing public company audits. Practice management systems supporting public-company audit work must maintain independence tracking, conflict-of-interest records, and rotation schedules for engagement partners and quality reviewers.

**AICPA SOC 2 Trust Services Criteria**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- SOC 2 certification is a baseline requirement for accounting practice management platforms because their clients are themselves subject to data security obligations. The five Trust Services Criteria are: Security (mandatory), Availability, Processing Integrity, Confidentiality, and Privacy. 2026 audit expectations include documented access controls, network segmentation, MFA, least-privilege policies, API security, and device inventories.

**GDPR (EU Regulation 2016/679)**
- URL: https://gdpr.eu/
- Applies when serving EU clients or processing EU residents' data. Requires lawful basis for data processing, data subject rights (access, erasure, portability), data residency options, and Data Processing Agreements with sub-processors. UK GDPR applies post-Brexit for UK firm clients.

**CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- State-level privacy law applicable to firms serving California residents. Requires disclosure of data collection purposes, opt-out rights for data sale, and deletion rights. Relevant to practice management platforms storing client financial and tax data.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0
- The machine-readable standard for describing REST APIs. Karbon publishes its API reference as an OpenAPI 3.1.0 document at karbonhq.github.io/karbon-api-reference/. An AI-native practice management platform should publish its own API in OpenAPI 3.1.0 format to enable third-party integrations.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Standard for describing and validating JSON data structures. Relevant for defining client record schemas, engagement data models, and webhook payload structures consistently across integrations.

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The dominant standard for delegated authorisation in SaaS. Required for integrations with QuickBooks Online, Xero, Google Workspace, and Microsoft 365. All accounting practice management platforms use OAuth 2.0 for their own API access control and for connecting to third-party platforms.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Authentication layer built on OAuth 2.0. Required for enterprise SSO integration with identity providers (Okta, Azure AD, Google Workspace). Accounting firm clients with 10+ staff increasingly expect OIDC-based SSO. Provides ID tokens (JWT), a UserInfo endpoint, and a discovery mechanism (.well-known/openid-configuration).

**JSON Web Tokens (RFC 7519)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard format for access and identity tokens in OAuth 2.0 and OIDC flows. All major accounting SaaS APIs issue JWTs as bearer tokens for API authentication.

**WebSub / PubSubHubbub (W3C)**
- URL: https://www.w3.org/TR/websub/
- Publish-subscribe protocol for webhook-style push notifications from web resources. Informs the design of webhook subscription models used by Karbon and Canopy for real-time event notifications.

**W3C WCAG 2.2 (Web Content Accessibility Guidelines)**
- URL: https://www.w3.org/TR/WCAG22/
- Accessibility standard for web-based software. Accounting firm practice management software used across large firms should meet WCAG 2.2 Level AA to comply with ADA requirements and serve users with disabilities.

---

### Security & Authentication Standards

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST guidance on authentication assurance levels. IRS Publication 4557 aligns with NIST 800-63B Level 2 requirements, which mandate MFA for all systems accessing sensitive taxpayer data. Relevant to session management, password policies, and authenticator requirements.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- Standard awareness document for web application security. Particularly relevant: A01 Broken Access Control (multi-tenant data isolation between firms), A02 Cryptographic Failures (AES-256 at rest and TLS 1.2+ in transit per IRS 4557), and A07 Identification and Authentication Failures (MFA enforcement).

**TLS 1.3 (RFC 8446)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- Current version of the Transport Layer Security protocol. All data transmitted between accounting practice management software and clients, integrations, and storage must use TLS 1.2 as a minimum (TLS 1.3 preferred) per IRS 4557 and FTC Safeguards Rule encryption mandates.

---

## Similar Products — Developer Documentation & APIs

### Karbon

- **Description:** Collaborative cloud practice management platform covering workflow, email triage, client management, time tracking, billing, and AI automation. Market leader by G2 rankings for 18 consecutive quarters through Spring 2026.
- **API Documentation:** https://developers.karbonhq.com/
- **API Reference (OpenAPI 3.1.0):** https://karbonhq.github.io/karbon-api-reference/
- **GitHub:** https://github.com/karbonhq/karbon-api-reference
- **Standards:** REST/JSON, OpenAPI 3.1.0, Webhook subscriptions (custom field, work, contact events)
- **Authentication:** API Key + Tenant Access Key (retrieved from within the Karbon app)
- **Developer notes:** Postman collections and OpenAPI spec files available on GitHub. Snowflake connector for read-only data warehouse access. Webhooks support CustomField, NoteComment, and other entity event types.

---

### Canopy

- **Description:** Modular accounting practice management platform offering CRM, document management, workflow, time and billing, client portal, payments, and (from Summer 2026) bookkeeping intelligence. Positioned as the most modular platform in the category.
- **API Documentation:** https://www.getcanopy.com/api/
- **API Tracker listing:** https://apitracker.io/a/canopytax
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0
- **Developer notes:** Official API described as enabling automation of accounting workflows. Canopy Coworker AI is available as a platform capability; no public AI API exposed as of 2026.

---

### QuickBooks Online (Intuit)

- **Description:** The dominant small-business accounting platform in the US and a required integration target for any accounting practice management tool. All invoicing, ledger data, and payroll data that flows through a client's books is accessible via the QBO API.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **API Explorer:** https://developer.intuit.com/app/developer/qbo/docs/learn/explore-the-quickbooks-online-api
- **SDKs:** Official SDKs for Java, PHP, Python, Ruby, and .NET available on the Intuit Developer Portal.
- **Standards:** REST/JSON, OAuth 2.0 (1-hour access tokens, 100-day rolling refresh tokens)
- **Authentication:** OAuth 2.0
- **Developer notes:** Resources grouped by entity type (customers, invoices, payments). Rate limits apply. Intuit requires app certification for production access to customer-facing apps.

---

### Xero

- **Description:** Cloud accounting platform dominant in the UK, Australia, New Zealand, and international small business markets. Key integration target for accounting practice management tools serving firms with international clients.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **API Tracker listing:** https://apitracker.io/a/xero
- **SDKs:** Official SDKs for .NET, Java, Python, Ruby, PHP, and Node.js available on the Xero Developer portal.
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI specification available
- **Authentication:** OAuth 2.0 (PKCE flow recommended)
- **Developer notes:** Uncertified apps receive 5,000 API calls per day per tenant. Higher quotas available for Xero App Store certified partners. Xero also offers a Practice Manager API specifically for accounting firms.

---

### DocuSign eSignature

- **Description:** Market-leading e-signature platform. Accounting practice management tools require e-signature capability for engagement letters, 8879 tax forms, and client consent documents. DocuSign is the most common integration target for this capability.
- **API Documentation:** https://developers.docusign.com/docs/esign-rest-api/
- **REST API Reference:** https://developers.docusign.com/docs/esign-rest-api/reference/
- **Developer Center:** https://developers.docusign.com/
- **SDKs:** Official SDKs for C#, Java, Python, Ruby, Node.js, PHP available on the DocuSign Developer Center.
- **Standards:** REST/JSON, OAuth 2.0, webhook event notifications
- **Authentication:** OAuth 2.0 (Authorization Code Grant and JWT Grant)
- **Developer notes:** Supports dynamic field placement via API, reusable envelope templates, sequential and parallel signing workflows, audit trails, and automated reminders. Demo environment available for development and testing.

---

### Microsoft Graph API (Microsoft 365)

- **Description:** Unified API for Microsoft 365 services including Outlook email, Teams, SharePoint, OneDrive, and Calendar. Essential integration target for accounting firms running on Microsoft infrastructure, covering email triage, calendar sync, and document storage.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/overview
- **SDKs:** Microsoft Graph SDKs for .NET, JavaScript/TypeScript, Java, Python, Go, PHP, and PowerShell.
- **Standards:** REST/JSON, OData v4, OpenAPI, OAuth 2.0 / OpenID Connect
- **Authentication:** OAuth 2.0 via Microsoft Identity Platform (Azure AD / Entra ID)
- **Developer notes:** Delegated permissions (user-consented) and application permissions (admin-consented) model. Supports webhooks (change notifications) for mailbox, calendar, and file events.

---

### Google Workspace APIs (Gmail / Calendar / Drive)

- **Description:** Integration target for accounting firms running on Google Workspace. Gmail API enables email triage features; Calendar API enables deadline sync; Drive API enables document management bridges.
- **API Documentation:** https://developers.google.com/workspace
- **Gmail API:** https://developers.google.com/gmail/api/guides
- **Calendar API:** https://developers.google.com/calendar/api/guides/overview
- **Drive API:** https://developers.google.com/drive/api/guides/about-sdk
- **SDKs:** Google API client libraries for Python, Java, Node.js, PHP, Ruby, .NET, and Go.
- **Standards:** REST/JSON, OAuth 2.0, OpenID Connect, Google's discovery document standard
- **Authentication:** OAuth 2.0 (Google Identity Services)
- **Developer notes:** Pub/Sub push notifications available for Gmail and Drive events. Workspace Marketplace listing required for distribution to Workspace organisations.

---

### Stripe

- **Description:** Payment processing platform used by Karbon (and others) for integrated billing within practice management tools. Enables invoice generation, payment collection, and reconciliation within workflow context.
- **API Documentation:** https://stripe.com/docs/api
- **SDKs:** Official SDKs for Ruby, Python, PHP, Java, Node.js, Go, and .NET.
- **Standards:** REST/JSON, webhook events, OpenAPI specification available
- **Authentication:** API Key (publishable key for client-side, secret key for server-side)
- **Developer notes:** Stripe Connect enables multi-party payment flows (relevant if the platform facilitates payments between clients and firms). Stripe Billing supports subscription and invoice automation aligned with recurring engagement billing models.

---

## Notes

- **No open API standard for accounting practice management data models exists.** Each platform uses proprietary data schemas for clients, jobs, tasks, and engagements. A standardised open data model (analogous to OpenAPI for REST or FHIR for healthcare) would be a significant contribution of an open-source platform in this space.
- **IRS MeF XML schemas** are published annually and are the closest thing to a shared data standard for tax filing workflows, but they cover return submission only, not practice management data.
- **The FTC Safeguards Rule and IRS Publication 4557** together create a compliance floor that any practice management platform must meet before accounting firms can adopt it. These are non-negotiable baseline requirements, not optional enhancements.
- **Webhook standards** across the platforms are inconsistent: Karbon uses a subscription model with an API; TaxDome relies on Zapier; Financial Cents and Pascal Workflow have no documented webhook capability. An AI-native platform should adopt a standardised webhook model (following the CloudEvents specification at cloudevents.io) to enable ecosystem interoperability.
