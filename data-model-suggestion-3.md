# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

> Candidate #478 — Corporate Travel Expense Automation

## Approach

A PostgreSQL schema that keeps core financial entities in strongly-typed relational columns while using JSONB columns for inherently variable, nested, or schema-evolving data. This gives the best of both worlds: referential integrity and SQL queryability for the structured backbone (amounts, statuses, foreign keys) plus schema-free flexibility for OCR outputs, policy rule definitions, AI model results, receipt metadata, and ERP sync payloads. GIN and expression indexes on JSONB columns ensure query performance remains competitive with fully relational alternatives.

## Why This Suits the Domain

Corporate expense management has a **stable structural core** — organisations, users, expense reports, approval steps, and reimbursements follow well-defined relationships that rarely change. However, several areas exhibit **high variability**:

1. **OCR extraction outputs** vary by receipt language, format, and AI model version. A Japanese taxi receipt returns different fields than a US hotel folio. Locking these into fixed columns means constant migrations or wide, sparse tables.
2. **Policy rules** range from simple thresholds ("max $75 for meals") to complex, context-dependent natural-language rules interpreted by LLMs. A rigid `rule_type` + `threshold_amount` pair cannot represent "meals over $50 require a receipt unless the employee is travelling in a city with per-diem rates above $200/day."
3. **AI model metadata** — confidence scores, model versions, alternative classifications, fraud signals — evolves with every model update. JSONB absorbs this evolution without schema changes.
4. **ERP integration payloads** differ across QuickBooks, Xero, and NetSuite. Storing the outbound payload and the response as JSONB avoids creating ERP-specific tables.
5. **Corporate card feed data** includes raw transaction metadata that varies by card network (Visa vs. Amex vs. Mastercard), with different supplementary fields.

The hybrid model preserves ACID guarantees and relational joins for financial reporting while accommodating this variability through JSONB columns with targeted indexes.

## Schema

```sql
-- ============================================================
-- ORGANISATION & IDENTITY
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    fiscal_year_start SMALLINT NOT NULL DEFAULT 1 CHECK (fiscal_year_start BETWEEN 1 AND 12),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { "timezone": "America/New_York", "receipt_required_above": 25.00,
    --             "auto_submit_enabled": true, "default_approval_levels": 2,
    --             "supported_languages": ["en","ja","de"], "logo_url": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    cost_centre     TEXT,
    parent_id       UUID REFERENCES departments(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { "budget_owner_email": "...", "erp_department_code": "4200" }
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
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences: { "default_currency": "GBP", "mileage_unit": "miles",
    --                "notification_channels": ["email","push"], "locale": "en-GB",
    --                "delegate_approver_id": null, "bank_account_last4": "7890" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_dept ON users(department_id);
CREATE INDEX idx_users_manager ON users(manager_id);

-- ============================================================
-- CHART OF ACCOUNTS & EXPENSE CATEGORIES
-- ============================================================

CREATE TABLE expense_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    gl_account      TEXT,
    parent_id       UUID REFERENCES expense_categories(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    classification  JSONB NOT NULL DEFAULT '{}',
    -- classification: { "tax_deductible": true, "requires_receipt": true,
    --                   "mcc_codes": ["5812","5813"], "default_tax_rate": 0.08,
    --                   "erp_mappings": { "quickbooks": "6200", "xero": "429" } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, code)
);
CREATE INDEX idx_categories_org ON expense_categories(organisation_id);
CREATE INDEX idx_categories_mcc ON expense_categories
    USING GIN ((classification -> 'mcc_codes'));

-- ============================================================
-- EXPENSE POLICIES
-- ============================================================

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
    priority        SMALLINT NOT NULL DEFAULT 0,
    rule_definition JSONB NOT NULL,
    -- rule_definition stores the full rule in a structured-but-flexible format:
    -- {
    --   "type": "max_amount",
    --   "conditions": {
    --     "currency": "USD",
    --     "threshold": 75.00,
    --     "applies_to_roles": ["employee"],
    --     "excluded_departments": ["executive"],
    --     "geo_exceptions": [
    --       { "country": "JP", "city": "Tokyo", "threshold": 120.00 }
    --     ]
    --   },
    --   "action": "flag_violation",
    --   "severity": "warning",
    --   "human_readable": "Meals capped at $75 USD except Tokyo ($120). Executive dept exempt.",
    --   "llm_prompt_hint": "Consider whether the meal was a client entertainment event..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_rules_policy ON policy_rules(policy_id);
CREATE INDEX idx_policy_rules_type ON policy_rules
    USING GIN ((rule_definition -> 'type'));

-- ============================================================
-- CURRENCY & EXCHANGE RATES
-- ============================================================

CREATE TABLE exchange_rates (
    id              BIGSERIAL PRIMARY KEY,
    base_currency   CHAR(3) NOT NULL,
    target_currency CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    source          TEXT NOT NULL DEFAULT 'ecb',
    effective_date  DATE NOT NULL,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (base_currency, target_currency, effective_date, source)
);
CREATE INDEX idx_exchange_rates_lookup
    ON exchange_rates(base_currency, target_currency, effective_date DESC);

-- ============================================================
-- RECEIPTS & OCR
-- ============================================================

CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    file_key        TEXT NOT NULL,
    file_mime       TEXT NOT NULL,
    file_size_bytes BIGINT,
    upload_channel  TEXT NOT NULL
                    CHECK (upload_channel IN ('mobile_camera','email','bulk_upload','card_feed')),
    ocr_status      TEXT NOT NULL DEFAULT 'pending'
                    CHECK (ocr_status IN ('pending','processing','completed','failed')),
    ocr_result      JSONB,
    -- ocr_result stores the full, variable OCR extraction:
    -- {
    --   "merchant": { "name": "Tokyo Taxi Co.", "address": "1-2-3 Shibuya", "phone": "+81..." },
    --   "date": "2026-03-15",
    --   "time": "14:32",
    --   "amounts": {
    --     "subtotal": 2800, "tax": 280, "total": 3080, "tip": null,
    --     "currency": "JPY", "currency_confidence": 0.98
    --   },
    --   "line_items": [
    --     { "description": "Metered fare", "amount": 2800, "quantity": 1 }
    --   ],
    --   "category_hint": "transportation",
    --   "language_detected": "ja",
    --   "payment_method": "credit_card",
    --   "card_last_four": "4532",
    --   "model_version": "vlm-receipt-v3.2",
    --   "confidence": 0.94,
    --   "raw_text": "...",
    --   "bounding_boxes": { ... }
    -- }
    ocr_merchant    TEXT GENERATED ALWAYS AS (ocr_result ->> 'merchant_name') STORED,
    ocr_amount      NUMERIC(14,2) GENERATED ALWAYS AS (
                        (ocr_result #>> '{amounts,total}')::NUMERIC(14,2)
                    ) STORED,
    ocr_currency    CHAR(3) GENERATED ALWAYS AS (
                        (ocr_result #>> '{amounts,currency}')::CHAR(3)
                    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_receipts_org ON receipts(organisation_id);
CREATE INDEX idx_receipts_user ON receipts(uploaded_by);
CREATE INDEX idx_receipts_status ON receipts(ocr_status);
CREATE INDEX idx_receipts_ocr_gin ON receipts USING GIN (ocr_result jsonb_path_ops);
-- Expression index for fast merchant lookups extracted from JSONB
CREATE INDEX idx_receipts_merchant ON receipts ((ocr_result ->> 'merchant_name'))
    WHERE ocr_status = 'completed';

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
    card_metadata   JSONB NOT NULL DEFAULT '{}',
    -- card_metadata: { "program_id": "...", "spending_limit": 5000, "currency": "USD",
    --                  "card_type": "corporate", "billing_address_country": "US" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cards_user ON corporate_cards(user_id);

CREATE TABLE card_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_id         UUID NOT NULL REFERENCES corporate_cards(id),
    transaction_ref TEXT NOT NULL,
    merchant_name   TEXT,
    mcc_code        CHAR(4),
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    transacted_at   TIMESTAMPTZ NOT NULL,
    matched_expense_id UUID,
    raw_feed_data   JSONB NOT NULL DEFAULT '{}',
    -- raw_feed_data: Network-specific data preserved verbatim:
    -- {
    --   "network_transaction_id": "VTS-2026-abc123",
    --   "merchant_category_group": "Travel",
    --   "merchant_country": "JP",
    --   "merchant_city": "Tokyo",
    --   "authorization_code": "A7X92B",
    --   "pos_entry_mode": "chip",
    --   "is_recurring": false,
    --   "enhanced_data": { ... }   -- varies by network
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (card_id, transaction_ref)
);
CREATE INDEX idx_card_txn_card ON card_transactions(card_id);
CREATE INDEX idx_card_txn_unmatched ON card_transactions(matched_expense_id)
    WHERE matched_expense_id IS NULL;
CREATE INDEX idx_card_txn_feed ON card_transactions USING GIN (raw_feed_data jsonb_path_ops);

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
    ai_summary      JSONB,
    -- ai_summary: {
    --   "narrative": "3-day client visit to Tokyo. 6 expenses totalling ¥47,200...",
    --   "model_version": "gpt-4o-2026-03",
    --   "generated_at": "2026-03-18T10:00:00Z",
    --   "key_highlights": ["2 meals over per-diem", "all receipts attached"],
    --   "suggested_tags": ["client-meeting","Q1-2026"]
    -- }
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reports_org ON expense_reports(organisation_id);
CREATE INDEX idx_reports_user ON expense_reports(submitted_by);
CREATE INDEX idx_reports_status ON expense_reports(status);
CREATE INDEX idx_reports_dates ON expense_reports(trip_start_date, trip_end_date);

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
    converted_amount NUMERIC(14,2),
    exchange_rate   NUMERIC(18,8),
    exchange_rate_id BIGINT REFERENCES exchange_rates(id),
    is_billable     BOOLEAN NOT NULL DEFAULT FALSE,
    project_code    TEXT,
    cost_centre     TEXT,
    ai_classification JSONB,
    -- ai_classification: {
    --   "predicted_category_id": "uuid",
    --   "confidence": 0.91,
    --   "alternatives": [
    --     { "category_id": "uuid", "confidence": 0.06, "reason": "..." },
    --     { "category_id": "uuid", "confidence": 0.03, "reason": "..." }
    --   ],
    --   "signals": ["mcc_5812","merchant_db_match","receipt_text_keyword"],
    --   "model_version": "cat-v2.1",
    --   "classified_at": "2026-03-15T14:35:00Z"
    -- }
    policy_check    JSONB,
    -- policy_check: {
    --   "passed": false,
    --   "violations": [
    --     {
    --       "rule_id": "uuid",
    --       "rule_type": "max_amount",
    --       "severity": "warning",
    --       "description": "Meal exceeds $75 limit by $12.50",
    --       "ai_interpretation": "Appears to be a client dinner; consider exception.",
    --       "auto_waived": false
    --     }
    --   ],
    --   "checked_at": "2026-03-15T14:36:00Z",
    --   "policy_version_id": "uuid"
    -- }
    fraud_analysis  JSONB,
    -- fraud_analysis: {
    --   "score": 0.12,
    --   "signals": ["normal_merchant","expected_geography","amount_within_norm"],
    --   "duplicate_candidates": [],
    --   "analysed_at": "2026-03-15T14:36:01Z",
    --   "model_version": "fraud-v1.4"
    -- }
    travel_details  JSONB,
    -- travel_details: For mileage, per-diem, and travel-specific data:
    -- Mileage: { "type": "mileage", "distance_km": 45.2, "rate_per_km": 0.67,
    --            "route": "Office to Client HQ", "gps_trace_ref": "s3://..." }
    -- Per-diem: { "type": "per_diem", "location": "Tokyo", "country": "JP",
    --             "daily_rate": 197.00, "num_days": 3, "rate_source": "gsa" }
    -- Flight: { "type": "flight", "airline": "JAL", "flight_no": "JL5",
    --           "class": "economy", "departure": "LAX", "arrival": "NRT" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lines_report ON expense_lines(report_id);
CREATE INDEX idx_lines_category ON expense_lines(category_id);
CREATE INDEX idx_lines_date ON expense_lines(expense_date);
CREATE INDEX idx_lines_fraud ON expense_lines
    ((fraud_analysis ->> 'score')::REAL)
    WHERE (fraud_analysis ->> 'score')::REAL > 0.5;
CREATE INDEX idx_lines_violations ON expense_lines
    USING GIN (policy_check jsonb_path_ops)
    WHERE policy_check IS NOT NULL;
CREATE INDEX idx_lines_travel_type ON expense_lines
    ((travel_details ->> 'type'))
    WHERE travel_details IS NOT NULL;

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
    routing_context JSONB,
    -- routing_context: {
    --   "routing_method": "ai",
    --   "reason": "Amount > $5000 requires VP approval",
    --   "original_approver_id": "uuid",
    --   "ai_confidence": 0.95,
    --   "escalation_history": [
    --     { "from": "uuid", "to": "uuid", "reason": "SLA breach", "at": "..." }
    --   ]
    -- }
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
    erp_system      TEXT NOT NULL,
    erp_reference   TEXT,
    sync_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (sync_status IN ('pending','synced','failed','retrying')),
    request_payload JSONB NOT NULL,
    -- request_payload: The exact payload sent to the ERP, structure varies per system:
    -- QuickBooks: { "type": "Bill", "VendorRef": {...}, "Line": [...], "TxnDate": "..." }
    -- Xero:       { "type": "BankTransaction", "Contact": {...}, "LineItems": [...] }
    -- NetSuite:   { "recordType": "vendorbill", "fields": {...}, "sublists": {...} }
    response_payload JSONB,
    -- response_payload: The raw response from the ERP
    error_details   JSONB,
    -- error_details: { "code": "DUPLICATE_DOC_NUMBER", "message": "...", "retry_count": 2 }
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
    payment_method  TEXT NOT NULL
                    CHECK (payment_method IN ('payroll','bank_transfer','card_credit')),
    payment_ref     TEXT,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','processing','completed','failed')),
    payment_details JSONB,
    -- payment_details: { "bank_name": "...", "account_last4": "7890",
    --                    "routing_number_last4": "1234", "processing_id": "PAY-abc",
    --                    "settlement_date": "2026-03-20", "failure_reason": null }
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
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    change_detail   JSONB NOT NULL DEFAULT '{}',
    -- change_detail: {
    --   "old": { "status": "draft", "total_amount": 0 },
    --   "new": { "status": "submitted", "total_amount": 450.00 },
    --   "diff_fields": ["status","total_amount"]
    -- }
    request_context JSONB NOT NULL DEFAULT '{}',
    -- request_context: { "ip": "203.0.113.42", "user_agent": "...",
    --                    "correlation_id": "req-abc", "source": "mobile_app" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create quarterly partitions
CREATE TABLE audit_log_2026_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE audit_log_2026_q4 PARTITION OF audit_log
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_changes ON audit_log USING GIN (change_detail jsonb_path_ops);

-- ============================================================
-- BUDGETS & ANALYTICS
-- ============================================================

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    department_id   UUID REFERENCES departments(id),
    category_id     UUID REFERENCES expense_categories(id),
    fiscal_year     SMALLINT NOT NULL,
    fiscal_month    SMALLINT,
    amount          NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    alert_config    JSONB NOT NULL DEFAULT '{}',
    -- alert_config: { "warn_at_pct": 80, "block_at_pct": 100,
    --                 "notify_emails": ["finance@example.com"],
    --                 "slack_channel": "#budget-alerts" }
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
    source          TEXT NOT NULL,
    rate_breakdown  JSONB,
    -- rate_breakdown: { "lodging": 150.00, "meals": 47.00,
    --                  "incidentals": 15.00, "first_last_day_pct": 75 }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_perdiem_lookup ON per_diem_rates(country_code, city, effective_from DESC);

CREATE TABLE mileage_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    rate_per_km     NUMERIC(6,4) NOT NULL,
    rate_per_mile   NUMERIC(6,4),
    currency        CHAR(3) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source          TEXT NOT NULL,
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
    match_details   JSONB NOT NULL DEFAULT '{}',
    -- match_details: { "matching_fields": ["amount","merchant","date"],
    --                  "receipt_image_similarity": 0.97,
    --                  "same_card_transaction": false }
    resolution      TEXT CHECK (resolution IN ('duplicate','not_duplicate','pending')),
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (line_a_id < line_b_id)
);
CREATE INDEX idx_dupes_pending ON duplicate_candidates(resolution) WHERE resolution = 'pending';
```

## Trade-offs

**Strengths:**

- **Single database, no polyglot overhead.** Unlike approaches that pair PostgreSQL with a separate document store, everything lives in one database. Operational complexity (backups, monitoring, failover) stays manageable.
- **Schema evolution without migrations for variable data.** Adding a new OCR output field, a new fraud signal, or a new ERP integration payload requires no ALTER TABLE. The JSONB columns absorb the change; only application code needs updating.
- **Relational integrity where it matters.** Foreign keys still enforce that every expense line belongs to a valid report, every report belongs to a valid user, and every user belongs to a valid organisation. Financial amounts remain in typed NUMERIC columns with proper precision.
- **Queryable flexibility.** Finance teams can query JSONB fields directly: `SELECT * FROM expense_lines WHERE policy_check @> '{"passed": false}'` finds all policy violations. GIN indexes make these queries performant.
- **Generated columns bridge the gap.** The `receipts` table uses PostgreSQL generated columns to extract frequently-queried OCR fields (merchant, amount, currency) into proper typed columns while keeping the full OCR result in JSONB. This provides fast B-tree index access for common queries without sacrificing flexibility.
- **Natural fit for AI outputs.** Every AI model (OCR, categorisation, fraud, policy interpretation) returns structured but evolving JSON. Storing these outputs in JSONB columns preserves the full model response for debugging, retraining, and audit without forcing each model's output into a rigid column structure.

**Weaknesses:**

- **Partial loss of schema enforcement.** JSONB columns accept any valid JSON. A bug that writes `{"ammount": 100}` instead of `{"amount": 100}` will not be caught by the database. Application-level validation or PostgreSQL CHECK constraints with `jsonb_typeof()` calls are needed.
- **GIN index write overhead.** Every INSERT or UPDATE to a row with a GIN-indexed JSONB column decomposes and indexes the full document. For `expense_lines` with multiple JSONB columns (ai_classification, policy_check, fraud_analysis, travel_details), this can reduce insert throughput by 30-50% compared to plain relational inserts. Mitigate with `jsonb_path_ops` operator class and partial indexes.
- **Harder to enforce JSONB structure across teams.** Unlike a column with a CHECK constraint, a JSONB column's expected structure is documented in comments and application code, not in the DDL. Schema registries or JSON Schema validation at the application layer help but add complexity.
- **Query syntax complexity.** JSONB path expressions (`ocr_result #>> '{amounts,total}'`) are less readable than plain column references. Developers unfamiliar with PostgreSQL's JSON operators face a learning curve.
- **Backup and analytics tooling may struggle.** Some BI tools and ETL pipelines handle JSONB columns poorly, requiring extraction transforms to flatten JSONB into columns for downstream consumption.

## Scalability Considerations

- **Audit log partitioning** is built into the schema (quarterly range partitions on `created_at`). Old partitions can be detached and archived to cold storage.
- **JSONB column sizing** should be monitored. If `ocr_result` documents average 5KB each and the system processes 500K receipts/year, that is 2.5GB/year in JSONB data alone for that one column. GIN indexes add 2-3x overhead. Plan for appropriate storage.
- **Read replicas** handle reporting load. BI tools should query replicas, not the primary.
- **Expression indexes** on frequently-queried JSONB paths (fraud score, policy pass/fail, travel type) provide B-tree performance for the most common filter patterns without the overhead of full GIN indexes.
- **TOAST compression** (PostgreSQL's automatic large-value compression) handles JSONB columns efficiently. Documents under 2KB are stored inline; larger ones are compressed and stored out-of-line automatically.
- **Connection pooling** via PgBouncer is recommended, as JSONB queries can hold connections slightly longer than simple column queries due to decompression overhead.

## Migration Path

- **From suggestion 1 (normalized relational):** Add JSONB columns alongside existing typed columns. Backfill JSONB from existing columns using `UPDATE receipts SET ocr_result = jsonb_build_object('merchant_name', ocr_merchant, 'amounts', jsonb_build_object('total', ocr_amount, 'currency', ocr_currency))`. Once validated, drop the redundant typed columns.
- **From suggestion 2 (event sourcing):** This hybrid schema works well as a read projection for an event-sourced write side. Event handlers populate both the relational columns and the JSONB fields.
- **To suggestion 4 (specialty):** JSONB columns like `fraud_analysis` and `travel_details` can be extracted and loaded into a graph or specialised analytics database for advanced analysis, while this schema remains the transactional system of record.
