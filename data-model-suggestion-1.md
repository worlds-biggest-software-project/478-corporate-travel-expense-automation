# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

> Candidate #478 — Corporate Travel Expense Automation

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity occupies its own table with proper foreign keys, check constraints, and indexes. This is the most widely understood pattern for financial applications, maps cleanly to SQL reporting, and provides strong transactional guarantees for monetary data.

## Why This Suits the Domain

Expense management is inherently transactional: receipts are captured, line items are created, reports are assembled, approvals are granted, and journal entries are posted. Each step has well-defined relationships (an expense line belongs to exactly one report; a report has exactly one submitter). Relational databases excel at enforcing these invariants. Financial auditors expect a schema they can query directly, and normalized tables make ad-hoc compliance queries straightforward. PostgreSQL specifically offers strong ACID guarantees, mature row-level security for multi-tenant isolation, and excellent support for monetary arithmetic via `NUMERIC` types.

## Schema

```sql
-- ============================================================
-- ORGANISATION & IDENTITY
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    fiscal_year_start SMALLINT NOT NULL DEFAULT 1, -- month 1-12
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    cost_centre     TEXT,
    parent_id       UUID REFERENCES departments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);
CREATE INDEX idx_departments_org ON departments(organisation_id);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,
    department_id   UUID REFERENCES departments(id),
    role            TEXT NOT NULL DEFAULT 'employee'
                    CHECK (role IN ('employee','approver','finance_admin','org_admin')),
    manager_id      UUID REFERENCES users(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_dept ON users(department_id);
CREATE INDEX idx_users_manager ON users(manager_id);

-- ============================================================
-- CHART OF ACCOUNTS & POLICIES
-- ============================================================

CREATE TABLE expense_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    gl_account      TEXT,
    parent_id       UUID REFERENCES expense_categories(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
CREATE INDEX idx_categories_org ON expense_categories(organisation_id);

CREATE TABLE expense_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    description     TEXT,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE policy_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES expense_policies(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES expense_categories(id),
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('max_amount','requires_receipt','requires_approval',
                                         'per_diem','mileage_rate','custom')),
    currency        CHAR(3),
    threshold_amount NUMERIC(14,2),
    rule_text       TEXT, -- human-readable rule for LLM interpretation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_rules_policy ON policy_rules(policy_id);

-- ============================================================
-- CURRENCY & EXCHANGE RATES
-- ============================================================

CREATE TABLE exchange_rates (
    id              BIGSERIAL PRIMARY KEY,
    base_currency   CHAR(3) NOT NULL,
    target_currency CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    source          TEXT NOT NULL DEFAULT 'ecb', -- ecb, openexchangerates, manual
    effective_date  DATE NOT NULL,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (base_currency, target_currency, effective_date, source)
);
CREATE INDEX idx_exchange_rates_lookup ON exchange_rates(base_currency, target_currency, effective_date);

-- ============================================================
-- RECEIPTS & OCR
-- ============================================================

CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    file_key        TEXT NOT NULL, -- S3/object-store key
    file_mime       TEXT NOT NULL,
    file_size_bytes BIGINT,
    upload_channel  TEXT NOT NULL
                    CHECK (upload_channel IN ('mobile_camera','email','bulk_upload','card_feed')),
    ocr_status      TEXT NOT NULL DEFAULT 'pending'
                    CHECK (ocr_status IN ('pending','processing','completed','failed')),
    ocr_merchant    TEXT,
    ocr_date        DATE,
    ocr_amount      NUMERIC(14,2),
    ocr_currency    CHAR(3),
    ocr_confidence  REAL, -- 0.0 to 1.0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_receipts_org ON receipts(organisation_id);
CREATE INDEX idx_receipts_user ON receipts(uploaded_by);
CREATE INDEX idx_receipts_status ON receipts(ocr_status);

-- ============================================================
-- CORPORATE CARD FEEDS
-- ============================================================

CREATE TABLE corporate_cards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    card_network    TEXT NOT NULL CHECK (card_network IN ('visa','mastercard','amex')),
    last_four       CHAR(4) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cards_user ON corporate_cards(user_id);

CREATE TABLE card_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_id         UUID NOT NULL REFERENCES corporate_cards(id),
    transaction_ref TEXT NOT NULL, -- external reference from card network
    merchant_name   TEXT,
    mcc_code        CHAR(4),
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    transacted_at   TIMESTAMPTZ NOT NULL,
    matched_expense_id UUID, -- set when matched to an expense line
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (card_id, transaction_ref)
);
CREATE INDEX idx_card_txn_card ON card_transactions(card_id);
CREATE INDEX idx_card_txn_unmatched ON card_transactions(matched_expense_id) WHERE matched_expense_id IS NULL;

-- ============================================================
-- EXPENSE REPORTS & LINE ITEMS
-- ============================================================

CREATE TABLE expense_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    submitted_by    UUID NOT NULL REFERENCES users(id),
    title           TEXT,
    purpose         TEXT,
    trip_start_date DATE,
    trip_end_date   DATE,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','pending_approval',
                                      'approved','rejected','paid','cancelled')),
    total_amount    NUMERIC(14,2) NOT NULL DEFAULT 0,
    total_currency  CHAR(3) NOT NULL,
    ai_narrative    TEXT, -- LLM-generated summary
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reports_org ON expense_reports(organisation_id);
CREATE INDEX idx_reports_user ON expense_reports(submitted_by);
CREATE INDEX idx_reports_status ON expense_reports(status);

CREATE TABLE expense_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_reports(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES expense_categories(id),
    receipt_id      UUID REFERENCES receipts(id),
    card_txn_id     UUID REFERENCES card_transactions(id),
    description     TEXT,
    merchant_name   TEXT,
    expense_date    DATE NOT NULL,
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    converted_amount NUMERIC(14,2), -- in org base currency
    exchange_rate   NUMERIC(18,8),
    exchange_rate_id BIGINT REFERENCES exchange_rates(id),
    is_billable     BOOLEAN NOT NULL DEFAULT FALSE,
    project_code    TEXT,
    cost_centre     TEXT,
    mileage_km      NUMERIC(10,2),
    mileage_rate    NUMERIC(6,4),
    per_diem_location TEXT,
    per_diem_rate   NUMERIC(14,2),
    ai_category_confidence REAL,
    policy_violation BOOLEAN NOT NULL DEFAULT FALSE,
    violation_reason TEXT,
    fraud_score     REAL, -- 0.0 to 1.0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lines_report ON expense_lines(report_id);
CREATE INDEX idx_lines_category ON expense_lines(category_id);
CREATE INDEX idx_lines_date ON expense_lines(expense_date);
CREATE INDEX idx_lines_fraud ON expense_lines(fraud_score) WHERE fraud_score > 0.5;

-- ============================================================
-- APPROVAL WORKFLOW
-- ============================================================

CREATE TABLE approval_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_reports(id) ON DELETE CASCADE,
    step_order      SMALLINT NOT NULL,
    approver_id     UUID NOT NULL REFERENCES users(id),
    delegate_id     UUID REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','rejected','skipped','escalated')),
    decision_note   TEXT,
    decided_at      TIMESTAMPTZ,
    escalated_at    TIMESTAMPTZ,
    due_by          TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_approvals_report ON approval_steps(report_id);
CREATE INDEX idx_approvals_approver ON approval_steps(approver_id);
CREATE INDEX idx_approvals_pending ON approval_steps(status) WHERE status = 'pending';

CREATE TABLE approval_delegations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    delegator_id    UUID NOT NULL REFERENCES users(id),
    delegate_id     UUID NOT NULL REFERENCES users(id),
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_to        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ACCOUNTING & ERP SYNC
-- ============================================================

CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_reports(id),
    erp_system      TEXT NOT NULL, -- 'quickbooks', 'xero', 'netsuite'
    erp_reference   TEXT, -- ID in external system
    sync_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (sync_status IN ('pending','synced','failed','retrying')),
    sync_payload    TEXT, -- serialised payload sent
    error_message   TEXT,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_journal_report ON journal_entries(report_id);
CREATE INDEX idx_journal_status ON journal_entries(sync_status);

CREATE TABLE reimbursements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES expense_reports(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    payment_method  TEXT NOT NULL CHECK (payment_method IN ('payroll','bank_transfer','card_credit')),
    payment_ref     TEXT,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','processing','completed','failed')),
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reimbursements_report ON reimbursements(report_id);
CREATE INDEX idx_reimbursements_user ON reimbursements(user_id);

-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    entity_type     TEXT NOT NULL, -- 'expense_report', 'expense_line', 'approval_step', etc.
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL, -- 'created', 'updated', 'submitted', 'approved', etc.
    old_values      TEXT,
    new_values      TEXT,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);

-- ============================================================
-- BUDGETS & ANALYTICS
-- ============================================================

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    department_id   UUID REFERENCES departments(id),
    category_id     UUID REFERENCES expense_categories(id),
    fiscal_year     SMALLINT NOT NULL,
    fiscal_month    SMALLINT, -- NULL = annual budget
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, department_id, category_id, fiscal_year, fiscal_month)
);
CREATE INDEX idx_budgets_org ON budgets(organisation_id);

-- ============================================================
-- PER-DIEM & MILEAGE RATES
-- ============================================================

CREATE TABLE per_diem_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    city            TEXT,
    daily_rate      NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source          TEXT NOT NULL, -- 'irs', 'hmrc', 'custom'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_perdiem_country ON per_diem_rates(country_code, city);

CREATE TABLE mileage_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    rate_per_km     NUMERIC(6,4) NOT NULL,
    rate_per_mile   NUMERIC(6,4),
    currency        CHAR(3) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source          TEXT NOT NULL, -- 'irs', 'hmrc', 'custom'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DUPLICATE & FRAUD DETECTION
-- ============================================================

CREATE TABLE duplicate_candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    line_a_id       UUID NOT NULL REFERENCES expense_lines(id),
    line_b_id       UUID NOT NULL REFERENCES expense_lines(id),
    similarity_score REAL NOT NULL,
    resolution      TEXT CHECK (resolution IN ('duplicate','not_duplicate','pending')),
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (line_a_id < line_b_id) -- prevent mirrored pairs
);
CREATE INDEX idx_dupes_pending ON duplicate_candidates(resolution) WHERE resolution = 'pending';
```

## Trade-offs

**Strengths:**
- Strong referential integrity enforced at the database level. Every relationship is explicit and constrained.
- Straightforward querying for finance teams and auditors using standard SQL. Budget-vs-actual reports, spend-by-category rollups, and approval pipeline queries are all natural joins.
- Mature tooling: ORM support, migration frameworks, backup/restore, read replicas.
- Row-level security in PostgreSQL enables multi-tenant isolation without application-layer complexity.
- NUMERIC types with fixed precision prevent floating-point rounding errors in financial calculations.

**Weaknesses:**
- Schema rigidity: adding a new field to receipts or expense lines requires a migration. In a domain where receipt formats and policy structures vary across organisations, this can lead to frequent ALTER TABLE operations.
- The OCR output columns on `receipts` (ocr_merchant, ocr_date, etc.) are somewhat awkward — if the OCR model returns 15 fields, you need 15 columns. A JSONB approach (see suggestion 3) handles this more gracefully.
- Audit log stores old/new values as TEXT, losing type safety. A more structured approach would use a dedicated event store (see suggestion 2).
- Multi-currency conversions involve joining to `exchange_rates` with date-based lookups, which can become complex in aggregate queries.

## Scalability Considerations

- Partitioning `audit_log` and `card_transactions` by `created_at` (range partitioning) is recommended once these tables exceed ~50M rows.
- Read replicas handle reporting load without affecting transactional writes.
- The `expense_lines` table will be the hottest table; the composite indexes on `(report_id)`, `(category_id)`, and `(expense_date)` cover the most common query patterns.
- For organisations processing millions of receipts, consider partitioning `receipts` by `organisation_id` using hash partitioning.

## Migration Path

This schema serves as a natural starting point. It can evolve toward:
- **Suggestion 3 (hybrid JSONB):** Add JSONB columns to `receipts` and `expense_lines` for flexible metadata without restructuring the core schema.
- **Suggestion 2 (event sourcing):** Layer an event store alongside this schema, using it as the read projection. The existing tables become materialised views rebuilt from events.
- **Suggestion 4 (graph):** Export relationship data from this schema into a graph database for fraud analysis while keeping the relational schema as the system of record.
