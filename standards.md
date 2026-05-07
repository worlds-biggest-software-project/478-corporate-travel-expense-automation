# Standards & API Reference

> Project: Corporate Travel & Expense Automation · Candidate #478 · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Financial Services Messaging**
- URL: https://www.iso20022.org/
- The global universal financial messaging standard for payment transactions, account reporting, and remittance data. Relevant for employee reimbursement flows: ISO 20022 `pain.001` (Customer Credit Transfer Initiation) and `camt.053` (Bank-to-Customer Statement) messages are used by modern banking rails (SWIFT, Fedwire, SEPA) for reimbursement payments. Request-for-Payment (RFP) message types allow one party to request a reimbursement from another with structured metadata. Adopted by FedNow (US) from 2025, CHAPS (UK), and TARGET2 (EU).

**ISO 4217 — Currency Codes**
- URL: https://www.iso.org/iso-4217-currency-codes.html
- Three-letter alphabetic codes (e.g. USD, EUR, GBP, JPY) and three-digit numeric codes for fiat currencies. Required for multi-currency expense line items, FX conversion rate storage, and report aggregation. Any expense data model must reference ISO 4217 codes for currency fields.

**ISO 3166 — Country Codes**
- URL: https://www.iso.org/iso-3166-country-codes.html
- Two-letter country codes (e.g. US, GB, DE) required for per-diem rate tables, tax jurisdiction rules, and receipt geolocation. Most expense policy engines reference ISO 3166-1 alpha-2 codes for country-specific rules.

**ISO 8583 — Financial Transaction Card Message**
- URL: https://www.iso.org/standard/31628.html
- The base messaging standard for card payment transactions at point-of-sale. Corporate card feed integrations from Visa, Mastercard, and Amex commercial programmes originate as ISO 8583 messages, enriched by the card network before delivery to expense platforms via proprietary feeds. Understanding this standard is important when building direct card-feed ingestion rather than using a third-party aggregator.

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- The leading information security management standard. Enterprise procurement requirements for T&E platforms typically mandate ISO 27001 certification. Expense platforms handle highly sensitive financial data (card numbers, bank accounts, salary-linked reimbursements) making this certification a practical necessity for enterprise sales.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation standard for delegated API access. All major expense platforms (SAP Concur, Expensify, Ramp, Brex, Zoho Expense) use OAuth 2.0 for their developer APIs. The Authorization Code + PKCE flow (RFC 7636) is the required pattern for web and mobile clients integrating with expense APIs.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the standard token format used in OAuth 2.0 bearer tokens. Expense API access tokens issued by platforms such as Concur, Ramp, and Brex are JWTs carrying scoped claims (e.g. `expense:read`, `cards:write`). JWT validation (signature, issuer, audience, expiry, scope) must be implemented server-side for every API call per OWASP guidelines.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard JSON error response format for REST APIs. Recommended for any expense management API to provide machine-readable error details (type URI, title, status, detail, instance) rather than ad-hoc error strings, improving developer integration experience.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standard `Link` header for pagination in REST API responses. Relevant for expense report list endpoints that may return thousands of records; platforms should implement cursor-based or offset-based pagination using RFC 8288 `rel=next` links.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Used by expense platforms for SSO integration with enterprise identity providers (Okta, Microsoft Entra ID, Google Workspace). Required for enterprise procurement sign-off: customers expect SAML 2.0 or OIDC-based SSO as a baseline feature.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- XML-based SSO federation standard. Large enterprise buyers (SAP Concur's primary market) require SAML 2.0 SP-initiated SSO via their corporate IdP (Active Directory Federation Services, Okta, Ping Identity). A new expense platform targeting enterprise accounts must implement SAML 2.0 alongside OIDC.

**SCIM 2.0 — System for Cross-domain Identity Management (RFC 7644)**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Standard REST API for provisioning and deprovisioning user accounts. Corporate expense platforms use SCIM to automatically create/update/deactivate employee records when HR changes occur (new hire, role change, termination). Brex and Ramp use SCIM for HRIS-driven card provisioning.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for documenting REST APIs. All major expense platforms publish OpenAPI (formerly Swagger) specifications: SAP Concur publishes OpenAPI specs at `api.sap.com`, Ramp publishes `/openapi/developer-api.json` from their docs site, and Brex uses OpenAPI to generate their SDKs and Postman collections. A new expense platform should publish an OpenAPI 3.1 spec to enable auto-generated SDKs and third-party integrations.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- The data validation vocabulary used natively in OpenAPI 3.1. Expense report, receipt, and transaction data models should be defined as JSON Schema documents to enable server-side request validation, client-side form validation, and API documentation generation from a single source of truth.

**OFX 2.3 — Open Financial Exchange**
- URL: https://financialdataexchange.org/about-fdx/ofx-work-group/
- XML/SGML legacy standard for bank transaction exports, deployed at over 7,000 financial institutions. OFX files are the primary mechanism for importing legacy bank statement data into expense tools for card reconciliation. OFX 2.2+ includes OAuth token-based authentication. Now managed by Financial Data Exchange (FDX).

**FDX API Standard**
- URL: https://financialdataexchange.org/
- The Financial Data Exchange (FDX) defines a standardised JSON REST API for consumer-permissioned financial data sharing, aligned with open banking mandates. Plaid's Core Exchange and similar aggregators implement FDX API specifications. Expense platforms that ingest bank transaction feeds from US financial institutions should target FDX API compliance as it is the emerging US open banking standard (in parallel with Plaid, MX, and Finicity).

**IATA NDC — New Distribution Capability**
- URL: https://developer.iata.org/en/ndc/
- IATA's XML-based API standard for airline content distribution. NDC enables corporate travel booking platforms (Navan, TravelPerk, SAP Concur Travel) to access full airline inventory, ancillaries, and personalised offers directly from airlines rather than through legacy GDS pipes. Relevant for any expense platform that incorporates a travel booking module or integrates with a TMC that uses NDC feeds.

**ISO/IEC 19464 — AMQP 1.0 (Advanced Message Queuing Protocol)**
- URL: https://www.amqp.org/
- Open standard for asynchronous message queuing. Used in enterprise expense platform architectures for decoupling card-feed ingestion, receipt OCR processing, ERP posting, and notification delivery. Azure Service Bus and AWS SQS both implement AMQP 1.0.

---

### Security & Authentication Standards

**OWASP Application Security Verification Standard (ASVS) 5.0**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- OWASP ASVS 5.0 includes a dedicated chapter (V10) on OAuth 2.0 and OIDC security requirements. For an expense platform, the relevant controls cover JWT validation, token binding, PKCE enforcement, refresh token rotation, and scope minimisation. ASVS Level 2 is appropriate for mid-market expense SaaS; Level 3 for enterprise-grade platforms handling bulk payroll-linked reimbursements.

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Mandatory compliance standard for any platform that stores, processes, or transmits cardholder data. Expense platforms with corporate card issuance (Ramp, Brex, Expensify Card) must attain PCI DSS SAQ D (or full QSA audit) for their card management components. Platforms that only import card transaction data (without PANs) via issuer APIs may qualify for a reduced scope assessment.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/soc-2
- The de-facto security certification for B2B SaaS. Enterprise and mid-market buyers of expense management software routinely require SOC 2 Type II attestation covering Security, Availability, and Confidentiality Trust Service Criteria. SOC 2 is not a prescriptive standard but an audit framework; obtaining it typically takes 6–12 months for a new platform.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Applies to any expense platform processing personal data of EU residents (employee names, bank account details, location data from GPS mileage tracking, receipt metadata). Key obligations: lawful basis for processing (legitimate interest for employer payroll reimbursements), data minimisation, right to erasure, cross-border transfer mechanisms (SCCs or Adequacy Decision). Spendesk and other European platforms have invested significantly in GDPR-native architecture.

**Financial-Grade API (FAPI) Security Profile**
- URL: https://openid.net/wg/fapi/
- FAPI is an OpenID Foundation specification that extends OAuth 2.0 with additional security requirements for high-value financial APIs. FAPI 2.0 (Baseline and Advanced profiles) mandates PKCE, JARM (JWT-secured authorisation response), mTLS sender-constrained tokens, and pushed authorisation requests (PAR). Relevant if the platform exposes reimbursement payment APIs that write to banking systems.

---

### MCP Server Specifications

**Anthropic Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Open standard for connecting AI agents to data systems and tools. By 2026, Forrester projects 30% of enterprise app vendors will publish MCP servers. Microsoft Dynamics 365 Finance & Operations already offers an MCP interface. An expense management platform exposing an MCP server could allow AI agents (Claude, GPT-4o, Gemini) to: query expense reports by date/category/employee, submit receipts, trigger approvals, and fetch policy rules — enabling natural-language expense management from within AI workspaces (Copilot, Claude Projects, Cursor).

---

## Similar Products — Developer Documentation & APIs

### SAP Concur

- **Description:** Enterprise T&E management platform processing a large share of global corporate expense spend. The Concur developer ecosystem covers expense reports, travel bookings, invoices, and spend requests.
- **API Documentation:** https://developer.concur.com/ and https://api.sap.com/products/SAPConcur/apis/REST
- **SDKs/Libraries:** No official first-party SDKs; community SDKs in Python and Node.js via GitHub (`api-evangelist/sap-concur`). Postman collections available via SAP API Hub.
- **Developer Guide:** https://developers.sap.com/tutorials/data-to-value-conn-concur-part01.html
- **Standards:** REST/JSON, OpenAPI 3.0 spec published at api.sap.com
- **Authentication:** OAuth 2.0 (Authorization Code flow; `company.write`, `expense.report.read` scopes)
- **Key Endpoints:** `GET /expensereports/v4/users/{userId}/context/{contextType}/reports`, `POST /expensereceipts/v4/users/{userId}/context/{contextType}/receipts`
- **Rate Limits:** Not publicly documented; partner-tier limits apply

---

### Expensify

- **Description:** SMB and mid-market expense automation with SmartScan OCR, Concierge AI, and the Expensify Card. The Integration Server API enables programmatic access to expense report data and account management.
- **API Documentation:** https://integrations.expensify.com/Integration-Server/doc/
- **SDKs/Libraries:** No official SDKs; community Python and JavaScript wrappers on GitHub. Postman collection: https://documenter.getpostman.com/view/28050470/2sA35D74CW
- **Developer Guide:** https://github.com/Expensify/Integrations
- **Standards:** REST/JSON over HTTPS; non-standard request structure (JSON-encoded in form POST, not standard REST body)
- **Authentication:** Partner credentials (`partnerUserID` + `partnerUserSecret`) generated at https://www.expensify.com/tools/integrations/
- **Key Endpoints:** All requests POST to `https://integrations.expensify.com/Integration-Server/ExpensifyIntegrations`; job-type parameter distinguishes operation (e.g. `Export`, `Create`)

---

### Ramp

- **Description:** AI-first spend management platform combining corporate cards, expense management, AP automation, and procurement. Developer API provides programmatic access to transactions, cards, and users.
- **API Documentation:** https://docs.ramp.com/
- **SDKs/Libraries:** OpenAPI spec at `/openapi/developer-api.json`; community Python SDK; Postman collection available via docs
- **Developer Guide:** https://ramp.com/developer-api; sandbox environment at `https://demo-api.ramp.com`
- **Standards:** REST/JSON, OpenAPI 3.1 spec, LLM-friendly docs (`/llms-full.txt`, `/llms-api.txt`)
- **Authentication:** OAuth 2.0 Bearer token (API tokens created in Ramp dashboard Developer settings)
- **Rate Limits:** 100 requests per minute
- **Notable:** Built-in Developer Assist AI in the documentation portal for real-time API Q&A

---

### Brex

- **Description:** Corporate card and spend management platform for startups and high-growth companies. Developer portal exposes Team (users/cards), Transactions, and Accounting APIs.
- **API Documentation:** https://developer.brex.com/
- **SDKs/Libraries:** OpenAPI spec with auto-generated SDKs; Postman collections at https://www.postman.com/brex-api/brex-developer/; Python docs via dltHub
- **Developer Guide:** https://www.brex.com/support/brex-api; journal article on API design: https://www.brex.com/journal/building-the-brex-api
- **Standards:** REST/JSON, OpenAPI-described (Redocly-published docs)
- **Authentication:** OAuth 2.0 Bearer token (created in Brex Dashboard → Developer settings); all requests require HTTPS
- **Base URL:** `https://api.brex.com`

---

### Navan (formerly TripActions)

- **Description:** Unified corporate travel booking and expense management platform. Provides Expense API, Booking API, and HR-SFTP connection for integration with ERP and HRIS systems.
- **API Documentation:** https://app.navan.com/app/helpcenter/articles/travel/admin/other-integrations/navan-tmc-api-integration-documentation
- **SDKs/Libraries:** Fivetran connector for data warehouse ingestion: https://fivetran.com/docs/connectors/applications/navan/setup-guide; Supergood API catalogue: https://docs.supergood.ai/navan-api/
- **Developer Guide:** https://navan.com/integrations; integration guide: https://navan.com/blog/corporate-travel-booking-software-integration-guide
- **Standards:** REST/JSON; HR integration via SFTP (for Workday, Rippling bulk sync)
- **Authentication:** OAuth 2.0 (details available to enterprise contract customers)
- **Key Capabilities:** Expense transaction feed, booking data feed, GL code sync, reimbursement data, employee profile provisioning

---

### Zoho Expense

- **Description:** SMB and mid-market expense management with deep Zoho ecosystem integration. REST API v1 covers expenses, receipts, expense reports, users, currencies, and categories.
- **API Documentation:** https://www.zoho.com/expense/api/v1/introduction/
- **SDKs/Libraries:** Python data connector via dltHub; Zoho Developer REST reference: https://www.zoho.com/developer/rest-api.html
- **Developer Guide:** https://www.zoho.com/expense/api/v1/
- **Standards:** REST/JSON, OAuth 2.0 (Zoho-oauthtoken)
- **Authentication:** OAuth 2.0 Bearer token (`Zoho-oauthtoken`) in Authorization header; `X-com-zoho-expense-organizationid` header required
- **Base URL:** `https://www.zohoapis.com/expense/v1`
- **Rate Limits:** 100 requests per minute per organisation

---

### Plaid (Financial Data Aggregation)

- **Description:** Open banking API platform connecting to 7,000+ financial institutions. Provides enriched transaction data (merchant names, categories, location) for bank and card feed ingestion into expense tools. FDX-aligned via Core Exchange.
- **API Documentation:** https://plaid.com/docs/api/products/transactions/
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, Java, Go at https://plaid.com/docs/libraries/
- **Developer Guide:** https://plaid.com/docs/; open banking guide: https://plaid.com/resources/open-finance/financial-api-integration/
- **Standards:** REST/JSON, FDX-aligned (Core Exchange), OAuth 2.0
- **Authentication:** `client_id` + `secret` + `access_token` (user-linked token obtained via Link flow)
- **Relevance:** Primary mechanism for US expense platforms to ingest personal and corporate bank account transactions without OFX file downloads. Provides 90%+ merchant categorisation accuracy and up to 24 months of history.

---

### IATA NDC Developer Portal

- **Description:** IATA's New Distribution Capability XML API standard for airline content distribution. Used by travel booking modules (TravelPerk, Navan, Concur Travel) to access full airline inventory and ancillaries.
- **API Documentation:** https://developer.iata.org/en/ndc/
- **Standards:** XML Schema (XSD), IATA NDC Schema 21.3+
- **Authentication:** Varies by airline implementation; typically API key or mutual TLS
- **Relevance:** Essential for any expense platform that includes a travel booking module or integrates directly with airlines. NDC adoption by 87% of corporate travel managers for cost savings makes this a strategic integration target.

---

## Notes

**Emerging Standards to Monitor:**
- **FDX API v6+**: The Financial Data Exchange is the leading candidate to become the regulatory-mandated open banking standard in the US (CFPB Section 1033 rulemaking). Expense platforms should design bank feed ingestion against FDX API rather than proprietary aggregator APIs to ensure long-term compliance.
- **ISO 20022 for Reimbursements**: As FedNow (US), SEPA Instant (EU), and Faster Payments (UK) all adopt ISO 20022, expense platforms should expose reimbursement payment initiation in ISO 20022 `pain.001` format to enable same-day ACH/RTP disbursement without payroll cycle dependency.
- **MCP Servers for Expense Agents**: The rapid enterprise adoption of MCP (30% of app vendors expected to publish servers in 2026 per Forrester) creates an opportunity to expose the expense platform's data and actions to AI agents via a standardised MCP server, enabling conversational expense management without custom integrations.
- **GDPR Article 17 (Right to Erasure) in multi-year audit trails**: Expense platforms face a tension between GDPR erasure requests and financial audit trail retention requirements (typically 7 years). The standard approach is pseudonymisation of personal identifiers while retaining financial amounts and categories for audit purposes.
