# Corporate Travel Expense Automation — Phased Development Plan

> Project: 478-corporate-travel-expense-automation · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB, PostgreSQL)** — it preserves typed `NUMERIC` financial columns and foreign-key integrity for the stable structural core while absorbing the high-variability AI outputs (OCR results, policy interpretations, fraud signals, ERP payloads) in JSONB columns without constant migrations. This matches the project's AI-native thesis and its self-hosted-first deployment model (single database, no polyglot operational burden).

---

## Core Requirements (synthesised)

- **What it does**: An open-source, AI-native platform that automates the full receipt-to-reimbursement workflow — vision-language-model OCR extraction, LLM-interpreted policy enforcement, agentic expense-report assembly, multi-level approvals, and near-real-time ERP sync.
- **Who uses it**: Finance leaders, AP teams, and travel managers at organisations with 50+ employees; the employees who capture receipts; the managers who approve.
- **Key differentiators**: VLM OCR (handwritten, non-English, low-quality receipts), LLM policy interpretation (handles ambiguity rigid rule engines flag wrongly), zero-touch agentic report assembly, real-time webhook ERP posting (not nightly batch), transparent open-source self-hosting.
- **MVP scope** (from features.md "Must-have"): VLM receipt OCR; ML+context categorisation; LLM policy engine; multi-level approval workflow; QuickBooks/Xero/NetSuite webhook sync; corporate card feed import (CSV + API).
- **v1.1 scope** ("Should-have"): agentic report assembly; pre-submit audit agent; duplicate/fraud detection; multi-currency FX; GPS mileage; analytics dashboard.
- **Backlog**: travel booking integration; price intelligence; non-Latin OCR tuning; public API marketplace; MCP server; AI narrative summaries.
- **Deployment model**: Self-hosted-first (Docker Compose) with a cloud/SaaS-capable multi-tenant architecture. Every table carries `organisation_id`; tenant isolation via PostgreSQL Row-Level Security.
- **Integration surface**: LLM/VLM providers (pluggable), object storage (S3-compatible) for receipt images, QuickBooks Online / Xero / NetSuite, card networks via CSV + Plaid/FDX, ECB FX rates, SMTP/IMAP for email-in, OIDC/SAML IdPs, SCIM provisioning.
- **Standards compliance**: ISO 4217 (currency), ISO 3166 (country), OAuth 2.0 + PKCE (RFC 6749/7636), JWT (RFC 7519), RFC 7807 (errors), RFC 8288 (pagination), OpenAPI 3.1 + JSON Schema 2020-12, OIDC/SAML 2.0 SSO, SCIM 2.0 (RFC 7644), OWASP ASVS L2, GDPR (pseudonymised erasure), ISO 20022 `pain.001` (reimbursement export, v1.1).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is LLM/VLM-heavy: OCR orchestration, policy interpretation, categorisation, fraud scoring, and narrative generation are all AI calls. Python has the richest, best-maintained SDK ecosystem for every major LLM/VLM provider plus structured-output tooling (Pydantic, Instructor). |
| API framework | FastAPI | Native async (essential for fanning out concurrent LLM calls and webhook delivery), first-class Pydantic v2 integration, and automatic OpenAPI 3.1 generation — directly satisfying the standards.md OpenAPI requirement with zero extra effort. |
| Data validation / schemas | Pydantic v2 | Single source of truth for request/response models, JSONB document schemas, and config. Emits JSON Schema 2020-12 used by OpenAPI and for validating JSONB structure at the application layer (the documented weakness of the JSONB model). |
| Database | PostgreSQL 16 | Required by the chosen hybrid data model: typed `NUMERIC` for money, JSONB + GIN/expression indexes for variable AI output, generated columns, range partitioning for audit log, and Row-Level Security for multi-tenancy. |
| Migrations | Alembic | Standard for SQLAlchemy; manages the relational backbone while JSONB columns evolve migration-free. |
| ORM / query layer | SQLAlchemy 2.0 (async) | Async sessions pair with FastAPI; supports JSONB operators, generated columns, and partitioned tables. |
| Task queue | Celery + Redis | OCR, categorisation, policy checks, fraud scoring, and ERP posting are long-running, retry-prone async workloads (standards.md flags AMQP-style decoupling). Celery gives retries, scheduled beat tasks (FX fetch, SLA escalation), and dead-letter handling. |
| Cache / broker | Redis 7 | Celery broker/result backend, FX-rate and per-diem cache, idempotency-key store for webhooks. |
| Object storage | S3-compatible (MinIO self-hosted; AWS S3 in cloud) | Receipt images must not live in Postgres. MinIO ships in Docker Compose for self-hosted parity. |
| LLM/VLM access | Provider-abstraction layer over OpenAI / Anthropic / local (Ollama) | The AI-native thesis demands model independence and self-hostability. A thin `LLMProvider` / `VLMProvider` interface keeps the engine model-agnostic. |
| Structured AI output | Instructor (Pydantic-validated tool/JSON mode) | Guarantees OCR/categorisation/policy results parse into typed models, with retry on validation failure. |
| Auth | Authlib (OAuth2/OIDC) + python-jose (JWT) + python3-saml | Standards.md mandates OAuth2+PKCE, OIDC, SAML 2.0, and SCIM for enterprise SSO/provisioning. |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Needs a finance dashboard (analytics, approvals, policy admin) and a responsive web receipt-capture flow. Server components for fast dashboards; PWA camera capture covers "mobile" without a separate native app in MVP. |
| API client typing | openapi-typescript | Generates TS types for the frontend from the FastAPI OpenAPI spec — single source of truth. |
| Containerisation | Docker + Docker Compose | Self-hosted-first requirement; Compose wires api, worker, beat, postgres, redis, minio, web. |
| Testing | pytest + pytest-asyncio + testcontainers; Playwright (e2e web) | Real Postgres/Redis/MinIO via testcontainers for integration; mocked LLM/ERP providers for determinism. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Fast, standard Python toolchain; mypy enforces typed boundaries around JSONB. |
| Package manager | uv | Fast, reproducible Python dependency resolution and locking. |
| Key libraries | `pdf2image`/`pillow` (receipt normalisation), `pycountry` (ISO 3166/4217), `babel` (currency/locale formatting), `python-multipart` (uploads), `tenacity` (retry), `structlog` (structured logs) | Domain-specific support for receipt handling, standards-compliant currency/country codes, and resilient integration. |

### Project Structure

```
corporate-expense-automation/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── openapi.json                      # exported spec (CI artifact)
├── migrations/
│   └── versions/
├── src/
│   └── cea/
│       ├── __init__.py
│       ├── main.py                   # FastAPI app factory, RFC 7807 handlers
│       ├── config.py                 # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py            # async engine/session, RLS tenant context
│       │   ├── base.py               # declarative base, mixins (timestamps)
│       │   └── models/               # SQLAlchemy ORM mapped to the hybrid schema
│       │       ├── organisation.py   # organisations, departments, users
│       │       ├── catalog.py        # expense_categories, policies, policy_rules
│       │       ├── receipt.py        # receipts
│       │       ├── card.py           # corporate_cards, card_transactions
│       │       ├── report.py         # expense_reports, expense_lines
│       │       ├── approval.py       # approval_steps, approval_delegations
│       │       ├── accounting.py     # journal_entries, reimbursements
│       │       ├── reference.py      # exchange_rates, per_diem_rates, mileage_rates
│       │       ├── analytics.py      # budgets, duplicate_candidates
│       │       └── audit.py          # audit_log (partitioned)
│       ├── schemas/                  # Pydantic request/response + JSONB doc models
│       │   ├── common.py             # Money, Page[T], ProblemDetail
│       │   ├── ocr.py                # OcrResult JSONB model
│       │   ├── policy.py             # RuleDefinition, PolicyCheck JSONB models
│       │   ├── classification.py     # AiClassification JSONB model
│       │   ├── fraud.py              # FraudAnalysis JSONB model
│       │   └── ...                   # report, receipt, approval, etc.
│       ├── api/
│       │   ├── deps.py               # auth, current_user, tenant, pagination
│       │   └── v1/
│       │       ├── router.py
│       │       ├── receipts.py
│       │       ├── reports.py
│       │       ├── approvals.py
│       │       ├── policies.py
│       │       ├── categories.py
│       │       ├── cards.py
│       │       ├── analytics.py
│       │       ├── webhooks.py       # inbound card feeds, email-in
│       │       └── admin.py          # org, users, SCIM
│       ├── services/                 # business logic (framework-agnostic)
│       │   ├── ocr_service.py
│       │   ├── categorisation_service.py
│       │   ├── policy_engine.py
│       │   ├── fraud_service.py
│       │   ├── report_assembly.py    # agentic grouping
│       │   ├── audit_agent.py        # pre-submit audit
│       │   ├── approval_service.py
│       │   ├── fx_service.py
│       │   ├── mileage_service.py
│       │   └── reimbursement_service.py
│       ├── ai/
│       │   ├── providers/
│       │   │   ├── base.py           # LLMProvider, VLMProvider protocols
│       │   │   ├── openai.py
│       │   │   ├── anthropic.py
│       │   │   └── ollama.py
│       │   ├── prompts/              # versioned prompt templates
│       │   └── registry.py           # provider selection by config
│       ├── integrations/
│       │   ├── erp/
│       │   │   ├── base.py           # ErpConnector protocol
│       │   │   ├── quickbooks.py
│       │   │   ├── xero.py
│       │   │   └── netsuite.py
│       │   ├── storage.py            # S3/MinIO client
│       │   ├── card_feeds/           # CSV + Plaid/FDX importers
│       │   └── email_in.py
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── tasks.py              # ocr, categorise, policy, fraud, erp_post
│       │   └── beat.py               # fx fetch, sla escalation, fraud sweep
│       ├── auth/
│       │   ├── oidc.py
│       │   ├── saml.py
│       │   ├── scim.py
│       │   └── jwt.py
│       └── telemetry/
│           └── logging.py            # structlog + correlation IDs
├── web/                              # Next.js frontend
│   ├── package.json
│   ├── app/
│   ├── components/
│   └── lib/api/                      # generated openapi-typescript client
└── tests/
    ├── conftest.py                   # testcontainers fixtures, fake providers
    ├── fixtures/                     # sample receipts, OCR JSON, ERP payloads
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (api / services / ai / integrations / workers), so every phase adds files without restructuring.

---

## Phase 1: Foundation, Tenancy & Data Model

### Purpose
Establish the project skeleton, configuration, the full PostgreSQL hybrid schema, multi-tenant isolation, and the structured-logging/error-handling baseline. After this phase the application boots, connects to Postgres/Redis/MinIO, applies all migrations, and enforces tenant isolation — every later phase writes against this schema without altering the core.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Bootstrap the repo, dependency management, Docker Compose stack, and env-driven settings.

**Design**:
- `pyproject.toml` with dependencies grouped (`api`, `workers`, `ai`, `dev`). `uv` for locking.
- `docker-compose.yml` services: `api`, `worker`, `beat`, `postgres:16`, `redis:7`, `minio`, `web`. Healthchecks on postgres/redis/minio; api depends on them.
- `cea/config.py` — Pydantic `Settings`:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="CEA_", env_file=".env")
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_bucket: str = "receipts"
    s3_access_key: SecretStr; s3_secret_key: SecretStr
    jwt_secret: SecretStr; jwt_issuer: str = "cea"; jwt_audience: str = "cea-api"
    llm_provider: Literal["openai", "anthropic", "ollama"] = "anthropic"
    vlm_provider: Literal["openai", "anthropic", "ollama"] = "anthropic"
    llm_model: str = "claude-sonnet-4"; vlm_model: str = "claude-sonnet-4"
    base_currency: str = "USD"
    receipt_required_above: Decimal = Decimal("25.00")
    environment: Literal["dev", "test", "prod"] = "dev"
```
- `cea/main.py` — `create_app()` factory; mounts `/api/v1`, `/healthz`, `/openapi.json`; installs RFC 7807 exception handlers (see 1.4).

**Testing**:
- `Unit: Settings loads from env with prefix → all fields populated, secrets masked in repr`.
- `Unit: missing CEA_DATABASE_URL → ValidationError naming database_url`.
- `Integration: GET /healthz with Compose up → 200 {"status":"ok","db":true,"redis":true,"s3":true}`.
- `Integration: docker compose build && up → all services healthy within 60s`.

#### 1.2 — Database schema & migrations

**What**: Implement the full hybrid schema from Data Model Suggestion 3 as SQLAlchemy models and an initial Alembic migration.

**Design**:
- All tables from suggestion-3: `organisations`, `departments`, `users`, `expense_categories`, `expense_policies`, `policy_rules`, `exchange_rates`, `receipts` (incl. generated `ocr_merchant`/`ocr_amount`/`ocr_currency` columns), `corporate_cards`, `card_transactions`, `expense_reports`, `expense_lines`, `approval_steps`, `approval_delegations`, `journal_entries`, `reimbursements`, `audit_log` (range-partitioned), `budgets`, `per_diem_rates`, `mileage_rates`, `duplicate_candidates`.
- Money columns `NUMERIC(14,2)`; currency `CHAR(3)` (ISO 4217); country `CHAR(2)` (ISO 3166-1 alpha-2). Status fields use the CHECK enumerations exactly as in the schema.
- JSONB columns: `settings`, `metadata`, `preferences`, `classification`, `rule_definition`, `ocr_result`, `card_metadata`, `raw_feed_data`, `ai_summary`, `ai_classification`, `policy_check`, `fraud_analysis`, `travel_details`, `request_payload`, `response_payload`, `error_details`, `change_detail`, `request_context`, `alert_config`, `rate_breakdown`, `match_details`.
- Recreate all GIN, expression, and partial indexes verbatim (`idx_receipts_ocr_gin`, `idx_lines_fraud`, `idx_lines_violations`, etc.).
- Audit log: create the migration with current + next-quarter partitions; a beat task (Phase 8) auto-creates future partitions.
- Alembic env configured for async engine; first revision `0001_initial`.

**Testing**:
- `Integration (testcontainers PG16): alembic upgrade head → all 22 tables + indexes exist (assert via information_schema)`.
- `Integration: insert receipt with ocr_result JSONB → generated ocr_amount equals amounts.total::numeric`.
- `Integration: insert expense_line referencing non-existent report_id → ForeignKeyViolation`.
- `Integration: insert expense_report status='bogus' → CheckViolation`.
- `Integration: alembic downgrade base then upgrade head → idempotent, no errors`.

#### 1.3 — Multi-tenant isolation (RLS) & tenant context

**What**: Enforce per-organisation data isolation via PostgreSQL Row-Level Security driven by a request-scoped tenant context.

**Design**:
- Enable RLS on every tenant-scoped table; policy: `USING (organisation_id = current_setting('cea.org_id')::uuid)`.
- `db/session.py`: each request sets `SET LOCAL cea.org_id = :org` from the authenticated principal before queries; a `tenant_context(org_id)` async context manager wraps the session.
- A dedicated migration-runner role bypasses RLS; the application role does not.

**Testing**:
- `Integration: org A session SELECT receipts → only org A rows (org B inserted but invisible)`.
- `Integration: query without tenant context set → returns zero rows (fail-closed)`.
- `Unit: tenant_context sets and resets SET LOCAL correctly across nested calls`.

#### 1.4 — Errors, pagination, and structured logging

**What**: Standards-compliant error envelope, pagination contract, and correlation-ID logging used by all endpoints.

**Design**:
- RFC 7807 `ProblemDetail` Pydantic model `{type, title, status, detail, instance, errors?}`. Global handlers map `ValidationError`→422, `NotFound`→404, `PermissionError`→403, unhandled→500, each as `application/problem+json`.
- Pagination: cursor-based `Page[T] = {items, next_cursor, total?}`; responses set an RFC 8288 `Link: <...>; rel="next"` header.
- `telemetry/logging.py`: structlog JSON renderer; middleware generates/propagates `correlation_id`, binds `org_id` and `user_id`, written into `audit_log.request_context`.

**Testing**:
- `Unit: ValidationError → ProblemDetail with status 422 and per-field errors`.
- `Integration: paginated list of 250 rows, limit 100 → 100 items + next_cursor + Link header; following cursor returns next 100`.
- `Unit: every log line includes correlation_id and org_id when bound`.

---

## Phase 2: Identity, Auth & Org Administration

### Purpose
Provide authentication, authorisation, and the organisation/user/category/policy administration surface that everything else depends on. After this phase an admin can stand up an org, create users with roles, define a chart of accounts and an expense policy — the prerequisites for capturing and processing expenses.

### Tasks

#### 2.1 — Authentication: OAuth2 + PKCE, JWT, OIDC

**What**: Issue and validate JWT access tokens; support Authorization Code + PKCE and OIDC login.

**Design**:
- `auth/jwt.py`: mint RS256 JWTs with claims `{iss, aud, sub, org_id, role, scope, exp, iat}`; validate signature, issuer, audience, expiry, and scope on every request per OWASP ASVS V10. Refresh-token rotation with reuse detection.
- `auth/oidc.py` (Authlib): `/auth/login` → IdP redirect with PKCE `code_challenge`; `/auth/callback` validates the code, maps the OIDC `sub`/email to a `users` row (auto-provision optional), issues local JWT.
- `api/deps.py`: `current_principal()` decodes JWT and yields `Principal{user_id, org_id, role, scopes}`; `require_scope("expenses:write")` dependency.
- Scopes: `receipts:write`, `expenses:read`, `expenses:write`, `approvals:write`, `policies:admin`, `org:admin`.

**Testing**:
- `Unit: token missing required scope → 403 ProblemDetail`.
- `Unit: expired/invalid-signature/wrong-audience token → 401`.
- `Integration (mocked IdP): callback with valid PKCE code → JWT issued, user provisioned`.
- `Unit: reused refresh token → entire token family revoked`.

#### 2.2 — SAML 2.0 SSO & SCIM 2.0 provisioning

**What**: Enterprise SP-initiated SAML login and SCIM user lifecycle.

**Design**:
- `auth/saml.py` (python3-saml): SP metadata endpoint, ACS endpoint validating signed assertions; maps NameID/attributes to a user, issues local JWT.
- `auth/scim.py` (RFC 7644): `/scim/v2/Users` (GET/POST/PATCH/DELETE) and `/Groups`; maps SCIM `active=false` → `users.is_active=false`; bearer-token-secured per org.

**Testing**:
- `Integration: signed SAML response → session established; tampered signature → 401`.
- `Integration: SCIM POST User → users row created; PATCH active=false → deactivated; GET filters by org`.

#### 2.3 — Organisation, department & user admin API

**What**: CRUD for orgs, departments, and users with role management.

**Design**:
- Endpoints: `POST /admin/organisations`, `GET/PATCH /admin/organisations/{id}`, `POST/GET/PATCH/DELETE /admin/departments`, `POST/GET/PATCH/DELETE /admin/users`.
- `users.role` ∈ `{employee, approver, finance_admin, org_admin}`; `manager_id` self-FK; department hierarchy via `parent_id`.
- `org_admin` scope required for org/user mutation. Org `settings` JSONB validated against a Pydantic `OrgSettings` model (timezone, receipt_required_above, auto_submit_enabled, default_approval_levels, supported_languages).

**Testing**:
- `Integration: create org → returns base_currency default USD; settings validated`.
- `Integration: create user role='approver' with manager → row linked; duplicate email in org → 409`.
- `Unit: OrgSettings rejects invalid timezone / negative threshold`.

#### 2.4 — Chart of accounts & policy administration

**What**: Manage `expense_categories` (with MCC mappings + ERP code mappings) and `expense_policies`/`policy_rules`.

**Design**:
- Category endpoints support hierarchy (`parent_id`) and `classification` JSONB (`tax_deductible`, `requires_receipt`, `mcc_codes`, `erp_mappings`).
- Policy endpoints create a versioned policy (`effective_from`/`effective_to`) with ordered `policy_rules`; each rule's `rule_definition` validated against the `RuleDefinition` Pydantic model:
```python
class GeoException(BaseModel):
    country: str; city: str | None = None; threshold: Decimal
class RuleConditions(BaseModel):
    currency: str | None = None; threshold: Decimal | None = None
    applies_to_roles: list[str] = []; excluded_departments: list[str] = []
    geo_exceptions: list[GeoException] = []
class RuleDefinition(BaseModel):
    type: Literal["max_amount","receipt_required","approved_vendor","time_of_day","per_diem_cap","free_form"]
    conditions: RuleConditions
    action: Literal["flag_violation","block","auto_waive"]
    severity: Literal["info","warning","error"]
    human_readable: str
    llm_prompt_hint: str | None = None
```

**Testing**:
- `Unit: rule_definition with type='free_form' + llm_prompt_hint → valid; missing human_readable → 422`.
- `Integration: create category with mcc_codes → GIN-indexed lookup by MCC returns it`.
- `Integration: overlapping effective policies for same org → second creation flags warning but persists (versioning)`.

---

## Phase 3: Receipt Capture & VLM OCR (Core Value #1)

### Purpose
Deliver the heart of the product: capture a receipt image and extract structured data using a vision-language model. This is the first user-visible "magic" and the input to every downstream step. After this phase a user uploads a receipt and receives accurate structured extraction with a confidence preview.

### Tasks

#### 3.1 — AI provider abstraction

**What**: Pluggable LLM/VLM provider interface so the engine is model-agnostic and self-hostable.

**Design**:
```python
class VLMProvider(Protocol):
    async def extract(self, image_bytes: bytes, mime: str,
                      schema: type[BaseModel], prompt: str) -> tuple[BaseModel, float]:
        """Return (parsed_model, confidence)."""
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str,
                       schema: type[BaseModel] | None = None) -> BaseModel | str: ...
```
- Implementations: `openai.py`, `anthropic.py`, `ollama.py`. Structured output via Instructor; `tenacity` retry (max 3) on validation failure with the parse error fed back into the prompt.
- `ai/registry.py` selects providers from `Settings.vlm_provider`/`llm_provider`. Prompt templates live in `ai/prompts/` and carry a `model_version` string written into JSONB outputs.

**Testing**:
- `Unit (fake provider): extract returns schema-valid model + confidence in [0,1]`.
- `Unit: provider returns malformed JSON twice then valid → succeeds on retry 3; always-invalid → raises after max retries`.
- `Unit: registry returns the configured provider class`.

#### 3.2 — Receipt upload & storage

**What**: Multi-channel receipt ingestion to object storage and a `receipts` row.

**Design**:
- `POST /receipts` (multipart) — `upload_channel` ∈ `{mobile_camera, email, bulk_upload, card_feed}`. Validates MIME (`image/jpeg|png|heic`, `application/pdf`), max size, normalises HEIC→JPEG and multi-page PDF→images via `pdf2image`/`pillow`.
- Stores original at `s3://receipts/{org_id}/{receipt_id}/original.{ext}`; writes `receipts` row `ocr_status='pending'`; enqueues `ocr.process` Celery task; returns `202` with receipt id.
- `POST /receipts/bulk` accepts a zip/multiple files. Email-in handler (`integrations/email_in.py`) polls IMAP, creates receipts with `upload_channel='email'`.

**Testing**:
- `Integration: POST jpeg → 202, receipt row pending, object in MinIO, ocr task enqueued (mock broker)`.
- `Unit: 20MB tiff → 415 unsupported / 413 too large ProblemDetail`.
- `Integration: HEIC upload → stored as normalised JPEG`.

#### 3.3 — VLM OCR extraction pipeline

**What**: The Celery task that runs VLM extraction and populates `ocr_result`.

**Design**:
- `OcrResult` Pydantic model mirrors the JSONB structure (merchant{name,address,phone}, date, time, amounts{subtotal,tax,total,tip,currency,currency_confidence}, line_items[], category_hint, language_detected, payment_method, card_last_four, model_version, confidence, raw_text).
- Task `ocr.process(receipt_id)`: load image from S3 → `VLMProvider.extract(image, schema=OcrResult, prompt=RECEIPT_PROMPT)` → set `ocr_status='processing'→'completed'`, persist `ocr_result`; on failure set `'failed'` and record error. Amount/currency validated against ISO 4217 (`pycountry`).
- `RECEIPT_PROMPT` (system): "You are an expert receipt-data extractor. Extract every field into the schema. Detect the receipt language and currency from symbols/format. Return amounts as numbers in the receipt's own currency. If a field is absent, return null — never guess. Report a calibrated confidence 0-1."

**Testing**:
- `Integration (fake VLM, fixture image): ocr.process → status completed, generated ocr_amount/currency populated`.
- `Fixture: Japanese taxi receipt JSON → language_detected='ja', currency='JPY', total numeric`.
- `Integration: VLM raises → status='failed', error captured, task does not crash worker`.
- `Unit: extracted currency 'XYZ' (not ISO 4217) → flagged low-confidence, currency nulled`.

#### 3.4 — Receipt preview & correction API

**What**: Return extraction with confidence for the real-time preview, and allow user correction.

**Design**:
- `GET /receipts/{id}` → receipt + `ocr_result` + per-field confidence; presigned image URL.
- `PATCH /receipts/{id}/extraction` → user overrides (merchant/date/amount/currency/category_hint). Corrections stored in `ocr_result.corrections` and emit an `audit_log` entry; corrections become training signal (flagged for future retraining export).

**Testing**:
- `Integration: GET completed receipt → ocr_result + presigned URL (valid, expiring)`.
- `Integration: PATCH amount → stored under corrections, audit_log row written`.
- `Integration: GET while pending → ocr_status pending, ocr_result null, 200`.

---

## Phase 4: Categorisation & Expense Lines

### Purpose
Turn extracted receipts into categorised expense line items — the unit that policy, approval, and accounting all operate on. After this phase a receipt becomes a typed `expense_line` with an AI-suggested category and confidence/alternatives.

### Tasks

#### 4.1 — AI categorisation service

**What**: Map a receipt/transaction to an `expense_categories` row using multi-modal signals.

**Design**:
- `categorisation_service.classify(line_context) -> AiClassification` combines: MCC code (card txn), merchant name, OCR `category_hint`, receipt text keywords, and employee/department context. LLM call returns `AiClassification{predicted_category_id, confidence, alternatives[], signals[], model_version, classified_at}`.
- Deterministic MCC→category lookup (via category `classification.mcc_codes` GIN index) runs first; LLM only resolves ambiguity / no-match. Result stored in `expense_lines.ai_classification`.

**Testing**:
- `Unit: MCC 5812 with a category mapping → deterministic match, no LLM call`.
- `Unit (fake LLM): ambiguous merchant → AiClassification with alternatives sorted desc by confidence`.
- `Integration: classify writes ai_classification JSONB; predicted_category_id references a real category`.

#### 4.2 — Expense line creation & receipt→line conversion

**What**: Create `expense_lines` from receipts (and standalone), with categorisation.

**Design**:
- `POST /reports/{id}/lines` from `receipt_id` or manual fields. On create: copy merchant/date/amount/currency from `ocr_result`, trigger categorisation, set `category_id` from prediction (employee may override).
- Money handling: `amount`/`currency` typed; `converted_amount`/`exchange_rate` filled in Phase 7. Line links to at most one `receipt_id` and one `card_txn_id`.
- `enforce receipt_required`: if category `classification.requires_receipt` and `amount > org.receipt_required_above` and no receipt → line flagged `needs_receipt`.

**Testing**:
- `Integration: create line from completed receipt → amount/currency copied, category predicted`.
- `Unit: line over threshold in receipt-required category without receipt → needs_receipt flag`.
- `Integration: override predicted category → category_id updated, ai_classification retained for audit`.

---

## Phase 5: LLM Policy Engine (Core Value #2)

### Purpose
Replace brittle rule engines with LLM-interpreted policy evaluation that correctly handles ambiguous edge cases — the second core differentiator. After this phase every expense line carries a `policy_check` result with violations, severity, and AI interpretation.

### Tasks

#### 5.1 — Deterministic rule evaluation

**What**: Evaluate structured rule types (max_amount, receipt_required, approved_vendor, time_of_day, per_diem_cap) without an LLM.

**Design**:
- `policy_engine.evaluate_deterministic(line, active_policy) -> list[Violation]`. Resolves the active policy by `effective_from/to`; applies rules in `priority` order; honours `geo_exceptions` (country/city threshold overrides) and `excluded_departments`/`applies_to_roles`.
- `Violation` Pydantic model: `{rule_id, rule_type, severity, description, ai_interpretation: str|None, auto_waived: bool}`.

**Testing**:
- `Unit: $87 meal, $75 cap, no geo exception → max_amount violation "exceeds by $12.00"`.
- `Unit: same line in Tokyo with geo_exception threshold $120 → no violation`.
- `Unit: executive department excluded → rule skipped`.
- `Unit: rules applied in priority order`.

#### 5.2 — LLM policy interpretation for ambiguous rules

**What**: Evaluate `free_form` rules and add contextual interpretation to deterministic violations.

**Design**:
- For `type='free_form'` rules (and to enrich borderline deterministic ones), call `LLMProvider.complete` with the rule's `human_readable` + `llm_prompt_hint`, the line context, and trip context, returning a structured `PolicyJudgement{passed, reason, suggested_severity, recommend_auto_waive}`.
- Prompt (system): "You are a corporate expense-policy auditor. Given a policy rule in plain English and an expense line with context, decide whether the expense complies. Be lenient on reasonable business judgement (e.g. legitimate client entertainment) and strict on clear abuse. Explain your reasoning in one sentence." Output merged into the line's `policy_check.violations[].ai_interpretation`.

**Testing**:
- `Unit (fake LLM): free_form "client dinners may exceed cap with justification" + justified line → passed=true`.
- `Unit: unjustified over-cap meal → violation retained, ai_interpretation explains`.

#### 5.3 — Policy check orchestration & storage

**What**: Combine deterministic + LLM results into `expense_lines.policy_check` and expose it.

**Design**:
- `policy_engine.check(line) -> PolicyCheck{passed, violations[], checked_at, policy_version_id}`; runs on line create/update and on report submit. Triggered as Celery task `policy.check(line_id)` for non-blocking UX; synchronous fast path for the deterministic-only case.
- `GET /reports/{id}/lines/{lid}/policy` returns the stored check. `idx_lines_violations` GIN index supports org-wide violation queries.

**Testing**:
- `Integration: line update → policy.check task runs → policy_check JSONB persisted with passed flag`.
- `Integration: query lines WHERE policy_check @> '{"passed": false}' → uses GIN index, returns violators`.
- `Integration: re-check after policy edit → policy_version_id reflects new policy`.

---

## Phase 6: Approval Workflow

### Purpose
Route submitted reports through configurable multi-level approval with delegation and escalation. After this phase a report can be submitted, approved/rejected through ordered steps, and moved toward payment.

### Tasks

#### 6.1 — Report submission & approval chain construction

**What**: Build the ordered `approval_steps` for a report on submission.

**Design**:
- `POST /reports/{id}/submit`: validates report (lines present, no blocking violations unless waived), transitions `draft→submitted→pending_approval`, recomputes `total_amount`/`total_currency`, builds approval chain.
- Chain rules: level 1 = submitter's `manager_id`; additional levels from org `settings.default_approval_levels` and amount thresholds (e.g. > $5000 adds a finance_admin step). Each step gets `step_order`, `approver_id`, `due_by`. Delegations (`approval_delegations` valid window) substitute `delegate_id`.
- Report status state machine: `draft → submitted → pending_approval → (approved | rejected) → paid | cancelled`. Enforced centrally; illegal transitions raise 409.

**Testing**:
- `Unit: submit with blocking (severity=error, not waived) violation → 409, not submitted`.
- `Integration: submit $6000 report → 2 steps (manager, finance_admin) ordered`.
- `Integration: approver on leave with active delegation → delegate set on step`.
- `Unit: approve a draft report → 409 illegal transition`.

#### 6.2 — Approve / reject / escalate

**What**: Act on approval steps and advance or finalise the report.

**Design**:
- `POST /approvals/{step_id}/decision` `{decision: approved|rejected, note}`. Approving the last pending step → report `approved`; any rejection → report `rejected`. Only the step's `approver_id` or `delegate_id` may act (enforced).
- Beat task `approvals.escalate` (Phase 8) sweeps `due_by`-breached pending steps → sets `escalated`, appends to `routing_context.escalation_history`, reassigns to the approver's manager.

**Testing**:
- `Integration: approve only step → report approved, approved_at set`.
- `Integration: reject step 1 → report rejected, later steps skipped`.
- `Unit: non-approver POSTs decision → 403`.
- `Integration: due_by past + escalate task → step escalated, reassigned, history appended`.

#### 6.3 — Mobile/web approval surface & notifications

**What**: List pending approvals and notify approvers.

**Design**:
- `GET /approvals?status=pending` returns the current approver's queue with report summaries (paginated, RFC 8288). Notification dispatch (email + optional Slack/Teams webhook) on assignment and escalation, channel per `users.preferences.notification_channels`.

**Testing**:
- `Integration: approver lists pending → only their assigned steps`.
- `Integration (mock SMTP): assignment → notification sent to approver email`.

---

## Phase 7: Accounting/ERP Sync, FX & Reimbursement

### Purpose
Close the loop: post approved reports to accounting systems in near-real-time, convert currencies, and record reimbursements. After this phase an approved report produces a journal entry in QuickBooks/Xero/NetSuite and a reimbursement record.

### Tasks

#### 7.1 — Multi-currency FX service

**What**: Fetch and apply ISO 4217 exchange rates to expense lines.

**Design**:
- `fx_service.convert(amount, from_ccy, to_ccy, on_date) -> (converted, rate, rate_id)`. Rates stored in `exchange_rates` (source `ecb`), fetched daily by a beat task from the ECB reference feed, cached in Redis. Line conversion uses the rate effective on `expense_date`; falls back to most recent prior rate.
- Report totals computed in the org `base_currency` by summing `converted_amount`.

**Testing**:
- `Unit: convert 100 GBP→USD with stored rate 1.27 → 127.00, rate_id set`.
- `Unit: no rate on exact date → uses most recent prior rate`.
- `Integration (mock ECB): daily fetch → exchange_rates populated, Redis cache warmed`.

#### 7.2 — ERP connector abstraction & QuickBooks/Xero/NetSuite

**What**: Pluggable connectors that map a report to each ERP's document model and post it.

**Design**:
```python
class ErpConnector(Protocol):
    name: str
    async def post_report(self, report: ExpenseReport, mapping: dict) -> ErpResult: ...
    # ErpResult{erp_reference, response_payload, status}
```
- Each connector builds the system-specific `request_payload` (QuickBooks Bill / Xero BankTransaction / NetSuite vendorbill) using category `classification.erp_mappings`. OAuth2 tokens per org stored encrypted. Idempotency via `journal_entries` unique on `(report_id, erp_system)`.

**Testing**:
- `Unit: QuickBooks connector maps report → Bill payload with correct VendorRef and Line GL accounts`.
- `Unit: Xero maps → BankTransaction with LineItems`.
- `Unit: missing erp_mapping for a category → ProblemDetail, no post attempted`.

#### 7.3 — Near-real-time posting pipeline & retries

**What**: On approval, post to the ERP via a resilient task and record the result.

**Design**:
- Report→`approved` triggers `erp.post(report_id)`. Task creates `journal_entries` row `pending`, calls the connector, stores `response_payload`/`erp_reference`, sets `synced`; on failure sets `failed`→`retrying` with exponential backoff (tenacity), capturing `error_details`. Webhook receiver (`api/v1/webhooks.py`) ingests ERP confirmations for two-way status.

**Testing**:
- `Integration (mock ERP): approved report → journal_entry synced, erp_reference stored`.
- `Integration: ERP 500 → status retrying, error_details captured, retried per backoff`.
- `Integration: duplicate post for same (report, erp) → blocked by unique constraint, no double entry`.

#### 7.4 — Reimbursement records (+ ISO 20022 export, v1.1)

**What**: Create reimbursement records and optionally export a payment file.

**Design**:
- On `approved` (or post-sync), create `reimbursements` row (amount in base currency, `payment_method`). `paid` transition sets `paid_at` and report `paid`.
- v1.1: `GET /reimbursements/export?format=pain.001` emits an ISO 20022 `pain.001` Customer Credit Transfer Initiation XML for a batch (per standards.md), enabling RTP/SEPA/FedNow disbursement.

**Testing**:
- `Integration: approved report → reimbursement pending in base currency`.
- `Unit: pain.001 export → schema-valid XML, amounts/currencies match (validate against XSD)`.

---

## Phase 8: Agentic Automation & Pre-Submit Audit (v1.1)

### Purpose
Deliver the zero-touch differentiator: agents that assemble reports automatically and audit them before they reach an approver. After this phase captured receipts flow into auto-assembled, pre-audited reports without employee data entry.

### Tasks

#### 8.1 — Card-feed import & receipt↔transaction matching

**What**: Import corporate card transactions and match them to receipts.

**Design**:
- `integrations/card_feeds/`: CSV importer (Visa/Mastercard/Amex export formats) and Plaid/FDX API importer, writing `card_transactions` with network-specific `raw_feed_data` JSONB. `POST /webhooks/card-feed` for push feeds (signature-verified, idempotent).
- Matching: candidate = same `last_four` + amount within tolerance + date window; merchant fuzzy match. On match set `card_transactions.matched_expense_id` and link `expense_lines.card_txn_id`. Unmatched txns surface via `idx_card_txn_unmatched`.

**Testing**:
- `Integration: import Visa CSV → card_transactions rows, raw_feed_data preserved`.
- `Unit: txn matches receipt within tolerance/date window → matched; outside → unmatched`.
- `Integration (mock): card-feed webhook bad signature → 401, nothing ingested`.

#### 8.2 — Agentic report assembly

**What**: Group unreported receipts/transactions into reports automatically.

**Design**:
- `report_assembly.assemble(user_id)`: cluster unattached receipts/matched txns by trip (date proximity + geography from `ocr_result`/`raw_feed_data`) or time period; create a `draft` report, generate lines (Phase 4), run categorisation + policy + fraud; if org `settings.auto_submit_enabled` and no blocking violations, auto-submit (Phase 6). Generates `ai_summary` narrative via LLM.
- Beat task runs per-user on a schedule; also exposed as `POST /reports/assemble`.

**Testing**:
- `Integration (fake AI): 5 receipts across 3 days same city → one trip report, lines created, ai_summary present`.
- `Unit: auto_submit_enabled + clean report → auto-submitted; with blocking violation → left draft`.

#### 8.3 — Pre-submit audit agent

**What**: Validate completeness and compliance before a report reaches an approver.

**Design**:
- `audit_agent.audit(report) -> AuditResult{ready, issues[], summary}`: checks missing receipts, policy violations, duplicates (Phase 9), unusual amounts; LLM produces a manager-readable summary and a go/no-go recommendation. Blocks submission on `ready=false` unless overridden by finance_admin. Result stored in `expense_reports.ai_summary`.

**Testing**:
- `Unit (fake LLM): report missing a required receipt → ready=false, issue listed`.
- `Integration: clean report → ready=true, narrative summary stored`.

#### 8.4 — Scheduled jobs (beat)

**What**: Centralise periodic tasks.

**Design**:
- `workers/beat.py` schedules: daily FX fetch (7.1), hourly approval escalation (6.2), nightly fraud sweep (9.x), nightly agentic assembly (8.2), quarterly `audit_log` partition creation (1.2).

**Testing**:
- `Unit: beat schedule registers all five tasks with expected crontab`.
- `Integration: partition-creation task creates next quarter's audit_log partition`.

---

## Phase 9: Fraud, Duplicate Detection & Analytics (v1.1)

### Purpose
Add spend assurance and visibility — anomaly/duplicate detection and the finance dashboards that justify the purchase. After this phase finance teams see spend analytics with drill-down and receive fraud/duplicate alerts.

### Tasks

#### 9.1 — Duplicate detection

**What**: Detect duplicate expense lines/receipts.

**Design**:
- `fraud_service.find_duplicates(line)`: compare amount+merchant+date proximity and receipt image similarity (perceptual hash); write `duplicate_candidates` (with the `line_a_id < line_b_id` invariant) and `similarity_score`. Pending candidates surface for resolution; resolution recorded with `resolved_by`.

**Testing**:
- `Unit: two lines same amount/merchant/date → candidate with high score`.
- `Integration: resolve candidate as not_duplicate → resolution + resolved_by stored`.
- `Unit: ordering invariant enforced (a<b)`.

#### 9.2 — Anomaly/fraud scoring

**What**: Score lines for fraud using historical spend patterns.

**Design**:
- `fraud_service.score(line) -> FraudAnalysis{score, signals[], duplicate_candidates[], analysed_at, model_version}`. Unsupervised anomaly detection over the user/category spend time-series (z-score / isolation-forest) plus rule signals (round amounts, off-hours, geo mismatch vs card country). Stored in `expense_lines.fraud_analysis`; high scores (>0.5) indexed via `idx_lines_fraud` and alerted.

**Testing**:
- `Unit: amount 10x the user's category mean → high score, signal "amount_outlier"`.
- `Integration: scored line persists fraud_analysis; high-score query uses partial index`.

#### 9.3 — Analytics & budgets

**What**: Spend dashboards and budget-vs-actual.

**Design**:
- `GET /analytics/spend?group_by=category|department|cost_centre&period=...` returns aggregates in base currency with drill-down to lines. `budgets` table drives budget-vs-actual; alerts at `alert_config.warn_at_pct`/`block_at_pct`. Queries run against a read replica in production.

**Testing**:
- `Integration: seed lines across 2 departments → group_by=department returns correct base-currency totals`.
- `Unit: spend at 85% of budget with warn_at_pct=80 → warning emitted`.
- `Integration: drill-down from a category aggregate → underlying lines returned`.

---

## Phase 10: Mileage, Per-Diem & Web Application

### Purpose
Complete the user-facing experience and the remaining travel-specific expense types. After this phase users have a full web app (capture, reports, approvals, dashboards) and can log mileage and per-diem expenses.

### Tasks

#### 10.1 — Mileage & per-diem services

**What**: Compute mileage and per-diem expense lines with statutory rates.

**Design**:
- `mileage_service`: distance (GPS trace or origin/destination) × `mileage_rates.rate_per_km`/`rate_per_mile` for the user's country (IRS/HMRC). Stored as a line with `travel_details{type:"mileage",...}`.
- Per-diem: `per_diem_rates` lookup by `country_code`/`city`/date × days; `travel_details{type:"per_diem",...}`. Rate tables seeded from GSA/published sources.

**Testing**:
- `Unit: 45.2 km × 0.67 → 30.28 line amount, travel_details.type=mileage`.
- `Unit: 3 days Tokyo per-diem → daily_rate × 3, first/last-day pct applied`.

#### 10.2 — Web frontend (Next.js)

**What**: The finance + employee web application.

**Design**:
- App Router pages: receipt capture (PWA camera, real-time OCR preview from 3.4), report builder, approvals queue, analytics dashboard, policy/category admin. Auth via OIDC (Phase 2). API access through `openapi-typescript`-generated client (regenerated from `/openapi.json` in CI). shadcn/ui + Tailwind.

**Testing**:
- `E2E (Playwright, mocked API): capture receipt → preview shows merchant/amount → add to report → submit → appears in approver queue`.
- `E2E: approver approves → report status approved in UI`.
- `Component: OCR preview renders low-confidence fields highlighted`.

#### 10.3 — OpenAPI publication & MCP server (backlog seed)

**What**: Publish the OpenAPI 3.1 spec and expose an MCP server.

**Design**:
- CI exports `openapi.json` and validates it against OAS 3.1. `mcp` server (standards.md) exposes tools: `query_reports`, `submit_receipt`, `trigger_approval`, `get_policy`, mapped to existing services with the caller's scopes enforced.

**Testing**:
- `Integration: GET /openapi.json → valid OAS 3.1 (schema-validated)`.
- `Integration (MCP client): query_reports tool → returns reports respecting tenant + scope`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Data Model      ─── required by everything
    │
Phase 2: Identity, Auth & Org Admin            ─── requires Phase 1
    │
Phase 3: Receipt Capture & VLM OCR             ─── requires Phase 2
    │
Phase 4: Categorisation & Expense Lines        ─── requires Phase 3
    │
Phase 5: LLM Policy Engine                     ─── requires Phase 4
    │
Phase 6: Approval Workflow                     ─── requires Phase 5
    │
Phase 7: ERP Sync, FX & Reimbursement          ─── requires Phase 6
    │
    ├── Phase 8: Agentic Automation & Audit     ─── requires Phases 4-7
    ├── Phase 9: Fraud, Duplicates & Analytics  ─── requires Phase 4 (parallel with 8)
    └── Phase 10: Mileage, Per-Diem & Web App   ─── requires Phase 6 (parallel with 8 & 9)
```

**Parallelism opportunities**
- Phases 8, 9, and 10 can be developed concurrently once Phase 7 lands (Phase 9 needs only Phase 4; Phase 10's web app needs Phase 6).
- Within Phase 2, SAML/SCIM (2.2) can be deferred and parallelised against the rest of auth.
- The Next.js web app (10.2) can begin against the OpenAPI spec as soon as Phases 3-6 endpoints stabilise.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`); mocked-provider and testcontainers suites both green.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict` on `src/cea`).
5. `docker compose build` succeeds and the affected services start healthy.
6. The phase's primary capability works end-to-end (verified by at least one integration or E2E test).
7. New config options added to `Settings` and documented in `.env.example`.
8. New API endpoints appear in the auto-generated `/openapi.json` (validated as OAS 3.1) and, where consumed by the frontend, regenerate the TypeScript client cleanly.
9. Alembic migration created and reversible (`upgrade head` / `downgrade` round-trips) for any relational change; JSONB-only changes documented in the relevant Pydantic schema.
10. Tenant isolation holds: every new query path respects Row-Level Security (covered by an isolation test).
11. New AI prompts versioned with a `model_version` recorded in the stored JSONB output.
