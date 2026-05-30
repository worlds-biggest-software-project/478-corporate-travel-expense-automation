# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Candidate #478 — Corporate Travel Expense Automation

## Approach

An event-sourced architecture where every state change is captured as an immutable domain event in an append-only event store. The Command side validates business rules and emits events; the Query side builds optimised read projections from the event stream. This separates write-path concerns (policy validation, approval logic, fraud detection) from read-path concerns (dashboards, reporting, audit).

## Why This Suits the Domain

Corporate expense management is a domain where **auditability is paramount**. Regulators, internal auditors, and finance teams need to answer questions like "who changed this expense amount, when, and why?" Event sourcing provides this natively — the event log IS the audit trail, not a bolted-on afterthought.

The expense lifecycle is also naturally event-driven: a receipt is uploaded, OCR extracts data, an expense line is created, a policy check flags a violation, the employee corrects it, a manager approves, finance posts to the ERP. Each of these is a discrete domain event. Event sourcing captures the full narrative rather than just the final state.

Additionally, CQRS allows the read side to be optimised independently. Finance dashboards querying spend-by-category-by-month can use pre-aggregated materialized projections, while the write side focuses purely on business rule enforcement.

## Event Store Schema

The event store itself is a simple append-only table. All domain events are stored here, regardless of aggregate type.

```sql
-- ============================================================
-- EVENT STORE (append-only)
-- ============================================================

CREATE TABLE event_store (
    sequence_num    BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL, -- 'ExpenseReport', 'Receipt', 'CorporateCard', etc.
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   SMALLINT NOT NULL DEFAULT 1,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}', -- user_id, ip, correlation_id, causation_id
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, sequence_num) -- optimistic concurrency
);
CREATE INDEX idx_events_aggregate ON event_store(aggregate_type, aggregate_id, sequence_num);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_time ON event_store(occurred_at);

-- Snapshot store for performance (rebuild aggregate from snapshot + subsequent events)
CREATE TABLE event_snapshots (
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

## Domain Events

### Receipt Aggregate Events

```
ReceiptUploaded
  { receipt_id, organisation_id, uploaded_by, file_key, file_mime, upload_channel, uploaded_at }

OcrProcessingStarted
  { receipt_id, processor_version }

OcrCompleted
  { receipt_id, merchant, date, amount, currency, category_hint, line_items[], confidence, raw_output }

OcrFailed
  { receipt_id, error_code, error_message }

ReceiptMatchedToCardTransaction
  { receipt_id, card_transaction_id, match_confidence }

ReceiptAttachedToExpenseLine
  { receipt_id, expense_line_id, report_id }
```

### Expense Report Aggregate Events

```
ExpenseReportCreated
  { report_id, organisation_id, submitted_by, title, purpose, trip_start, trip_end, currency }

ExpenseLineAdded
  { report_id, line_id, category_id, merchant, date, amount, currency, description,
    converted_amount, exchange_rate, receipt_id, card_txn_id, is_billable, project_code, cost_centre }

ExpenseLineUpdated
  { report_id, line_id, changed_fields: { field: { old, new } } }

ExpenseLineRemoved
  { report_id, line_id, reason }

MileageExpenseRecorded
  { report_id, line_id, distance_km, rate_per_km, amount, currency, route_description, gps_data_ref }

PerDiemExpenseRecorded
  { report_id, line_id, location, country_code, daily_rate, num_days, amount, currency }

PolicyCheckCompleted
  { report_id, violations: [{ line_id, rule_id, rule_text, severity, ai_explanation }], passed }

FraudScoreCalculated
  { report_id, overall_score, line_scores: [{ line_id, score, signals[] }] }

DuplicateDetected
  { report_id, line_id, duplicate_of_line_id, similarity_score }

DuplicateResolved
  { report_id, line_id, duplicate_of_line_id, resolution, resolved_by }

ExpenseReportSubmitted
  { report_id, submitted_by, submitted_at, total_amount, total_currency, line_count }

ExpenseReportAutoAssembled
  { report_id, assembled_by_agent, receipt_ids[], card_txn_ids[], grouping_strategy }

AiNarrativeGenerated
  { report_id, narrative_text, model_version }
```

### Approval Aggregate Events

```
ApprovalWorkflowInitiated
  { report_id, steps: [{ step_order, approver_id, due_by }] }

ApprovalStepAssigned
  { report_id, step_order, approver_id, delegate_id, due_by }

ApprovalGranted
  { report_id, step_order, approver_id, decision_note, decided_at }

ApprovalRejected
  { report_id, step_order, approver_id, rejection_reason, decided_at }

ApprovalEscalated
  { report_id, step_order, original_approver_id, escalated_to_id, reason, escalated_at }

ApprovalDelegated
  { report_id, step_order, delegator_id, delegate_id }

ExpenseReportFullyApproved
  { report_id, final_approver_id, approved_at }

ExpenseReportRejected
  { report_id, rejected_at, rejection_step, rejection_reason }
```

### Financial Integration Events

```
JournalEntryPosted
  { report_id, erp_system, erp_reference, line_entries[], posted_at }

JournalPostingFailed
  { report_id, erp_system, error_code, error_message, will_retry }

ReimbursementInitiated
  { report_id, user_id, amount, currency, payment_method }

ReimbursementCompleted
  { report_id, payment_ref, paid_at }

ReimbursementFailed
  { report_id, error_code, error_message }

ExchangeRateRecorded
  { base_currency, target_currency, rate, source, effective_date }
```

### Organisation & Policy Events

```
OrganisationCreated
  { org_id, name, base_currency, fiscal_year_start }

PolicyCreated
  { policy_id, org_id, name, effective_from, rules[] }

PolicyRuleAdded
  { policy_id, rule_id, rule_type, category_id, threshold_amount, currency, rule_text }

PolicyRuleUpdated
  { policy_id, rule_id, changed_fields }

PolicyDeactivated
  { policy_id, deactivated_at, reason }

BudgetSet
  { org_id, department_id, category_id, fiscal_year, fiscal_month, amount, currency }
```

## Command Handlers

Commands validate business invariants before emitting events. Each command handler loads the aggregate from its event stream, applies the command, and persists new events.

```
SubmitExpenseReport
  Pre-conditions:
    - Report is in 'draft' status
    - At least one expense line exists
    - All required receipts are attached (based on policy)
    - All line amounts are positive
  Emits: ExpenseReportSubmitted
  Side-effects: triggers PolicyCheckCompleted, FraudScoreCalculated, ApprovalWorkflowInitiated

ApproveExpenseReport
  Pre-conditions:
    - Current step status is 'pending'
    - Approver matches step's approver_id or delegate_id
    - Approver is not the same person as the submitter
  Emits: ApprovalGranted (and possibly ExpenseReportFullyApproved)

RejectExpenseReport
  Pre-conditions:
    - Current step status is 'pending'
    - Rejection reason is provided
  Emits: ApprovalRejected, ExpenseReportRejected

AddExpenseLine
  Pre-conditions:
    - Report is in 'draft' status
    - Amount > 0
    - Category exists and is active
    - Currency is valid ISO 4217
  Emits: ExpenseLineAdded

RecordMileage
  Pre-conditions:
    - Distance > 0
    - Valid mileage rate exists for the country/date
  Emits: MileageExpenseRecorded

PostToErp
  Pre-conditions:
    - Report is fully approved
    - No existing successful journal posting for this ERP
  Emits: JournalEntryPosted or JournalPostingFailed
```

## Read Projections

Projections subscribe to the event stream and build query-optimised read models. Each projection can be rebuilt from scratch by replaying events.

```sql
-- ============================================================
-- PROJECTION: Current Expense Report State
-- ============================================================

CREATE TABLE proj_expense_reports (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    submitted_by    UUID NOT NULL,
    submitter_name  TEXT NOT NULL,
    title           TEXT,
    purpose         TEXT,
    status          TEXT NOT NULL,
    total_amount    NUMERIC(14,2),
    currency        CHAR(3),
    line_count      INT DEFAULT 0,
    has_violations  BOOLEAN DEFAULT FALSE,
    fraud_score     REAL,
    ai_narrative    TEXT,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    current_approver_id UUID,
    created_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ
);
CREATE INDEX idx_proj_reports_org_status ON proj_expense_reports(organisation_id, status);
CREATE INDEX idx_proj_reports_submitter ON proj_expense_reports(submitted_by);

-- ============================================================
-- PROJECTION: Expense Lines (denormalized for fast reads)
-- ============================================================

CREATE TABLE proj_expense_lines (
    id              UUID PRIMARY KEY,
    report_id       UUID NOT NULL,
    category_code   TEXT,
    category_name   TEXT,
    merchant_name   TEXT,
    expense_date    DATE,
    amount          NUMERIC(14,2),
    currency        CHAR(3),
    converted_amount NUMERIC(14,2),
    receipt_id      UUID,
    has_receipt      BOOLEAN DEFAULT FALSE,
    policy_violation BOOLEAN DEFAULT FALSE,
    violation_reason TEXT,
    fraud_score     REAL,
    cost_centre     TEXT,
    project_code    TEXT
);
CREATE INDEX idx_proj_lines_report ON proj_expense_lines(report_id);

-- ============================================================
-- PROJECTION: Approval Queue (what each approver needs to action)
-- ============================================================

CREATE TABLE proj_approval_queue (
    report_id       UUID NOT NULL,
    step_order      SMALLINT NOT NULL,
    approver_id     UUID NOT NULL,
    submitter_name  TEXT NOT NULL,
    report_title    TEXT,
    total_amount    NUMERIC(14,2),
    currency        CHAR(3),
    submitted_at    TIMESTAMPTZ,
    due_by          TIMESTAMPTZ,
    PRIMARY KEY (report_id, step_order)
);
CREATE INDEX idx_proj_queue_approver ON proj_approval_queue(approver_id);

-- ============================================================
-- PROJECTION: Spend Analytics (pre-aggregated)
-- ============================================================

CREATE TABLE proj_spend_by_category_month (
    organisation_id UUID NOT NULL,
    category_id     UUID NOT NULL,
    category_name   TEXT NOT NULL,
    department_id   UUID,
    fiscal_year     SMALLINT NOT NULL,
    fiscal_month    SMALLINT NOT NULL,
    total_amount    NUMERIC(14,2) NOT NULL DEFAULT 0,
    transaction_count INT NOT NULL DEFAULT 0,
    budget_amount   NUMERIC(14,2),
    currency        CHAR(3) NOT NULL,
    PRIMARY KEY (organisation_id, category_id, department_id, fiscal_year, fiscal_month)
);

-- ============================================================
-- PROJECTION: Audit Timeline (human-readable)
-- ============================================================

CREATE TABLE proj_audit_timeline (
    sequence_num    BIGINT PRIMARY KEY,
    organisation_id UUID NOT NULL,
    report_id       UUID,
    event_type      TEXT NOT NULL,
    actor_id        UUID,
    actor_name      TEXT,
    summary         TEXT NOT NULL, -- human-readable description
    occurred_at     TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_audit_org ON proj_audit_timeline(organisation_id, occurred_at DESC);
CREATE INDEX idx_proj_audit_report ON proj_audit_timeline(report_id);
```

## Event Processing Pipeline

```
Event Store  -->  Event Bus (PostgreSQL LISTEN/NOTIFY or Kafka)
                     |
                     +--> Report Projection Updater
                     +--> Approval Queue Projection Updater
                     +--> Spend Analytics Aggregator
                     +--> Audit Timeline Builder
                     +--> ERP Sync Reactor (posts to external ERPs)
                     +--> Fraud Detection Reactor (recalculates fraud scores)
                     +--> Notification Reactor (sends emails/push/Slack)
```

## Trade-offs

**Strengths:**
- Complete, immutable audit trail is inherent to the architecture — not an afterthought. Every change to every expense is permanently recorded. This is a significant advantage for financial compliance (SOX, SOC 2, GDPR right-to-access).
- Temporal queries are trivial: "What did this report look like at 3pm Tuesday?" is answered by replaying events up to that timestamp.
- The CQRS split allows read models to be denormalised for dashboard performance without compromising write-side integrity.
- Event-driven reactors enable loose coupling: adding a new notification channel or analytics view requires only a new projection subscriber, no changes to core business logic.
- Natural fit for the agentic AI workflow: each AI agent action (OCR extraction, policy check, fraud scoring) emits its own event, creating a traceable chain of AI decisions.

**Weaknesses:**
- Significantly higher implementation complexity. Developers must understand eventual consistency, idempotent event handlers, and projection rebuild strategies.
- Eventual consistency between the write store and read projections means a just-submitted expense report might not appear in the dashboard for milliseconds to seconds. For most expense workflows this is acceptable, but it requires careful UX design.
- Schema evolution for events requires versioning (hence `event_version`). Changing an event's structure requires upcasting old events during replay.
- Storage grows monotonically. An organisation processing 100K expenses/year with ~20 events per expense lifecycle generates ~2M events/year. This is manageable but requires periodic snapshotting and archival strategies.
- Querying across aggregates requires projections; there is no simple "SELECT * FROM expenses WHERE amount > 1000" on the write side.

## Scalability Considerations

- The event store can be partitioned by `aggregate_type` or by `occurred_at` range.
- Projections can be rebuilt in parallel from the event store, enabling zero-downtime schema changes on the read side.
- Event processing can be scaled horizontally with consumer groups (Kafka) or partitioned LISTEN/NOTIFY channels.
- Snapshots should be taken every ~100 events per aggregate to keep rebuild times under 50ms.

## Migration Path

- **From suggestion 1 (relational):** Wrap existing relational writes in event-emitting commands. The relational tables become the initial read projections. Gradually shift the source of truth to the event store.
- **To suggestion 3 (hybrid JSONB):** Projections can use JSONB columns for flexible read models while the event store remains the canonical source.
- **To suggestion 4 (graph):** Graph projections can subscribe to the event stream and build relationship networks for fraud analysis in real time.
