# Data Model Suggestion 4: Graph Database Model (Neo4j)

> Candidate #478 — Corporate Travel Expense Automation

## Approach

A property graph model in Neo4j where organisations, employees, departments, expense reports, receipts, merchants, card transactions, policies, and approval workflows are modelled as nodes connected by typed, directional relationships. This approach treats the expense domain as a network of interconnected entities rather than a collection of tables, enabling relationship-first queries for fraud detection, approval chain traversal, spend pattern analysis, and organisational hierarchy navigation.

## Why a Graph Database Suits This Domain

Corporate travel expense management has several characteristics that align with graph database strengths:

1. **Fraud detection is inherently a graph problem.** Expense fraud often manifests as patterns across relationships: an employee and a vendor sharing an address, circular reimbursement chains, multiple employees submitting receipts from the same merchant on the same day at the same time, or a manager approving their own spouse's expenses through a delegation chain. Relational databases require expensive multi-way JOINs to detect these patterns; graph databases traverse them in constant time per hop.

2. **Approval chains are variable-depth hierarchies.** "Find everyone who can approve this expense" requires traversing manager-of-manager chains, delegation relationships, and role-based overrides. In SQL this requires recursive CTEs; in Cypher it is a single variable-length path expression.

3. **Organisational hierarchy queries are native.** "Show total spend for VP Sarah Chen's entire org" means traversing all `MANAGES` relationships downward and aggregating expenses. Graph databases handle this without materialised hierarchy tables.

4. **Policy applicability traverses multiple dimensions.** A policy rule might apply to "all employees in the EMEA region, in the Engineering department, travelling to countries with per-diem rates above $150." Evaluating this requires traversing organisation, department, geography, and rate nodes — a natural graph traversal.

5. **Merchant and vendor networks reveal intelligence.** Connecting merchants to categories, locations, card networks, and employee visit patterns builds a knowledge graph that powers both AI categorisation and anomaly detection.

## Schema Definition

### Node Labels and Properties

```cypher
// ============================================================
// ORGANISATION & IDENTITY NODES
// ============================================================

// Organisation
CREATE CONSTRAINT org_id IF NOT EXISTS FOR (o:Organisation) REQUIRE o.id IS UNIQUE;

// Properties: id (UUID), name, base_currency, fiscal_year_start,
//             timezone, created_at, updated_at

// Department
CREATE CONSTRAINT dept_id IF NOT EXISTS FOR (d:Department) REQUIRE d.id IS UNIQUE;
CREATE INDEX dept_name IF NOT EXISTS FOR (d:Department) ON (d.name);

// Properties: id (UUID), name, cost_centre, created_at

// Employee (User)
CREATE CONSTRAINT employee_id IF NOT EXISTS FOR (e:Employee) REQUIRE e.id IS UNIQUE;
CREATE INDEX employee_email IF NOT EXISTS FOR (e:Employee) ON (e.email);
CREATE INDEX employee_role IF NOT EXISTS FOR (e:Employee) ON (e.role);

// Properties: id (UUID), email, full_name, employee_code, role,
//             is_active, default_currency, locale, created_at, updated_at

// ============================================================
// EXPENSE CATEGORY & POLICY NODES
// ============================================================

// ExpenseCategory
CREATE CONSTRAINT category_id IF NOT EXISTS FOR (c:ExpenseCategory) REQUIRE c.id IS UNIQUE;
CREATE INDEX category_code IF NOT EXISTS FOR (c:ExpenseCategory) ON (c.code);

// Properties: id (UUID), code, name, gl_account, is_active,
//             tax_deductible, requires_receipt, mcc_codes (list), created_at

// Policy
CREATE CONSTRAINT policy_id IF NOT EXISTS FOR (p:Policy) REQUIRE p.id IS UNIQUE;

// Properties: id (UUID), name, description, effective_from, effective_to,
//             is_active, created_at

// PolicyRule
CREATE CONSTRAINT rule_id IF NOT EXISTS FOR (r:PolicyRule) REQUIRE r.id IS UNIQUE;

// Properties: id (UUID), rule_type, priority, threshold_amount, currency,
//             human_readable, llm_prompt_hint, severity, created_at

// ============================================================
// RECEIPT & OCR NODES
// ============================================================

// Receipt
CREATE CONSTRAINT receipt_id IF NOT EXISTS FOR (r:Receipt) REQUIRE r.id IS UNIQUE;
CREATE INDEX receipt_status IF NOT EXISTS FOR (r:Receipt) ON (r.ocr_status);

// Properties: id (UUID), file_key, file_mime, file_size_bytes,
//             upload_channel, ocr_status, created_at, updated_at

// OcrResult (separated as its own node to allow versioning and reprocessing)
CREATE CONSTRAINT ocr_id IF NOT EXISTS FOR (o:OcrResult) REQUIRE o.id IS UNIQUE;

// Properties: id (UUID), merchant_name, merchant_address, date, time,
//             subtotal, tax, total, tip, currency, currency_confidence,
//             category_hint, language_detected, payment_method,
//             model_version, overall_confidence, raw_text, processed_at

// Merchant (shared across receipts and card transactions)
CREATE CONSTRAINT merchant_id IF NOT EXISTS FOR (m:Merchant) REQUIRE m.id IS UNIQUE;
CREATE INDEX merchant_name IF NOT EXISTS FOR (m:Merchant) ON (m.name);
CREATE INDEX merchant_mcc IF NOT EXISTS FOR (m:Merchant) ON (m.mcc_code);

// Properties: id (UUID), name, mcc_code, category_group,
//             country, city, address, is_flagged, created_at

// ============================================================
// CORPORATE CARD & TRANSACTION NODES
// ============================================================

// CorporateCard
CREATE CONSTRAINT card_id IF NOT EXISTS FOR (c:CorporateCard) REQUIRE c.id IS UNIQUE;

// Properties: id (UUID), card_network, last_four, is_active,
//             spending_limit, currency, created_at

// CardTransaction
CREATE CONSTRAINT card_txn_id IF NOT EXISTS FOR (t:CardTransaction) REQUIRE t.id IS UNIQUE;
CREATE INDEX card_txn_ref IF NOT EXISTS FOR (t:CardTransaction) ON (t.transaction_ref);
CREATE INDEX card_txn_date IF NOT EXISTS FOR (t:CardTransaction) ON (t.transacted_at);

// Properties: id (UUID), transaction_ref, amount, currency,
//             transacted_at, authorization_code, pos_entry_mode, created_at

// ============================================================
// EXPENSE REPORT & LINE ITEM NODES
// ============================================================

// ExpenseReport
CREATE CONSTRAINT report_id IF NOT EXISTS FOR (r:ExpenseReport) REQUIRE r.id IS UNIQUE;
CREATE INDEX report_status IF NOT EXISTS FOR (r:ExpenseReport) ON (r.status);
CREATE INDEX report_dates IF NOT EXISTS FOR (r:ExpenseReport) ON (r.trip_start_date, r.trip_end_date);

// Properties: id (UUID), title, purpose, trip_start_date, trip_end_date,
//             status, total_amount, total_currency, ai_narrative,
//             submitted_at, approved_at, paid_at, created_at, updated_at

// ExpenseLine
CREATE CONSTRAINT line_id IF NOT EXISTS FOR (l:ExpenseLine) REQUIRE l.id IS UNIQUE;
CREATE INDEX line_date IF NOT EXISTS FOR (l:ExpenseLine) ON (l.expense_date);

// Properties: id (UUID), description, expense_date, amount, currency,
//             converted_amount, exchange_rate, is_billable, project_code,
//             cost_centre, created_at, updated_at

// MileageDetail (for mileage-type expense lines)
// Properties: distance_km, rate_per_km, route_description, gps_trace_ref

// PerDiemDetail (for per-diem-type expense lines)
// Properties: location, country_code, daily_rate, num_days, rate_source

// ============================================================
// APPROVAL WORKFLOW NODES
// ============================================================

// ApprovalStep
CREATE CONSTRAINT approval_id IF NOT EXISTS FOR (a:ApprovalStep) REQUIRE a.id IS UNIQUE;

// Properties: id (UUID), step_order, status, decision_note,
//             routing_method, ai_confidence, decided_at,
//             escalated_at, due_by, created_at

// ============================================================
// FINANCIAL INTEGRATION NODES
// ============================================================

// JournalEntry
CREATE CONSTRAINT journal_id IF NOT EXISTS FOR (j:JournalEntry) REQUIRE j.id IS UNIQUE;

// Properties: id (UUID), erp_system, erp_reference, sync_status,
//             error_message, synced_at, created_at

// Reimbursement
CREATE CONSTRAINT reimburse_id IF NOT EXISTS FOR (r:Reimbursement) REQUIRE r.id IS UNIQUE;

// Properties: id (UUID), amount, currency, payment_method, payment_ref,
//             status, paid_at, created_at

// ============================================================
// REFERENCE DATA NODES
// ============================================================

// Currency
CREATE CONSTRAINT currency_code IF NOT EXISTS FOR (c:Currency) REQUIRE c.code IS UNIQUE;
// Properties: code (CHAR 3), name, symbol

// Country
CREATE CONSTRAINT country_code IF NOT EXISTS FOR (c:Country) REQUIRE c.code IS UNIQUE;
// Properties: code (CHAR 2), name, region

// ExchangeRate
CREATE INDEX fx_date IF NOT EXISTS FOR (r:ExchangeRate) ON (r.effective_date);
// Properties: rate, source, effective_date, fetched_at

// PerDiemRate
// Properties: daily_rate, currency, effective_from, effective_to,
//             source, lodging_rate, meals_rate, incidentals_rate

// MileageRate
// Properties: rate_per_km, rate_per_mile, currency, effective_from,
//             effective_to, source

// ============================================================
// AUDIT & ANALYTICS NODES
// ============================================================

// AuditEvent
CREATE INDEX audit_time IF NOT EXISTS FOR (a:AuditEvent) ON (a.occurred_at);
CREATE INDEX audit_action IF NOT EXISTS FOR (a:AuditEvent) ON (a.action);

// Properties: id (BIGINT), action, entity_type, summary,
//             ip_address, user_agent, occurred_at

// Budget
CREATE CONSTRAINT budget_id IF NOT EXISTS FOR (b:Budget) REQUIRE b.id IS UNIQUE;
// Properties: id (UUID), fiscal_year, fiscal_month, amount,
//             currency, warn_at_pct, block_at_pct, created_at

// FraudAlert
CREATE CONSTRAINT fraud_id IF NOT EXISTS FOR (f:FraudAlert) REQUIRE f.id IS UNIQUE;
// Properties: id (UUID), score, signals (list), model_version,
//             resolution, resolved_by, analysed_at
```

### Relationship Types and Properties

```cypher
// ============================================================
// ORGANISATIONAL RELATIONSHIPS
// ============================================================

// (:Employee)-[:BELONGS_TO]->(:Organisation)
// (:Department)-[:PART_OF]->(:Organisation)
// (:Employee)-[:WORKS_IN]->(:Department)
// (:Department)-[:CHILD_OF]->(:Department)          -- hierarchy
// (:Employee)-[:MANAGES]->(:Employee)                -- org chart
// (:Employee)-[:DELEGATES_TO {valid_from, valid_to}]->(:Employee)

// ============================================================
// EXPENSE LIFECYCLE RELATIONSHIPS
// ============================================================

// (:Employee)-[:SUBMITTED]->(:ExpenseReport)
// (:ExpenseReport)-[:CONTAINS]->(:ExpenseLine)
// (:ExpenseLine)-[:CATEGORISED_AS]->(:ExpenseCategory)
// (:ExpenseCategory)-[:SUBCATEGORY_OF]->(:ExpenseCategory)
// (:ExpenseLine)-[:EVIDENCED_BY]->(:Receipt)
// (:ExpenseLine)-[:MATCHED_TO]->(:CardTransaction)
// (:ExpenseLine)-[:INCURRED_AT]->(:Merchant)
// (:ExpenseLine)-[:HAS_MILEAGE]->(:MileageDetail)
// (:ExpenseLine)-[:HAS_PER_DIEM]->(:PerDiemDetail)
// (:ExpenseLine)-[:CONVERTED_USING]->(:ExchangeRate)

// ============================================================
// RECEIPT & OCR RELATIONSHIPS
// ============================================================

// (:Employee)-[:UPLOADED]->(:Receipt)
// (:Receipt)-[:OCR_PRODUCED]->(:OcrResult)
// (:OcrResult)-[:IDENTIFIED_MERCHANT]->(:Merchant)
// (:Receipt)-[:MATCHES_TRANSACTION {confidence}]->(:CardTransaction)

// ============================================================
// CARD RELATIONSHIPS
// ============================================================

// (:Employee)-[:HOLDS]->(:CorporateCard)
// (:CorporateCard)-[:ISSUED_BY {card_network}]->(:Organisation)
// (:CardTransaction)-[:CHARGED_TO]->(:CorporateCard)
// (:CardTransaction)-[:TRANSACTED_AT]->(:Merchant)
// (:CardTransaction)-[:IN_CURRENCY]->(:Currency)

// ============================================================
// POLICY RELATIONSHIPS
// ============================================================

// (:Organisation)-[:HAS_POLICY]->(:Policy)
// (:Policy)-[:INCLUDES_RULE]->(:PolicyRule)
// (:PolicyRule)-[:APPLIES_TO_CATEGORY]->(:ExpenseCategory)
// (:PolicyRule)-[:APPLIES_TO_DEPARTMENT]->(:Department)
// (:PolicyRule)-[:APPLIES_TO_COUNTRY]->(:Country)
// (:ExpenseLine)-[:VIOLATED {severity, description, ai_interpretation}]->(:PolicyRule)

// ============================================================
// APPROVAL RELATIONSHIPS
// ============================================================

// (:ExpenseReport)-[:REQUIRES_APPROVAL]->(:ApprovalStep)
// (:ApprovalStep)-[:ASSIGNED_TO]->(:Employee)
// (:ApprovalStep)-[:DELEGATED_TO]->(:Employee)
// (:ApprovalStep)-[:ESCALATED_TO {reason, escalated_at}]->(:Employee)
// (:ApprovalStep)-[:NEXT_STEP]->(:ApprovalStep)     -- approval chain

// ============================================================
// FINANCIAL INTEGRATION RELATIONSHIPS
// ============================================================

// (:ExpenseReport)-[:POSTED_AS]->(:JournalEntry)
// (:ExpenseReport)-[:REIMBURSED_VIA]->(:Reimbursement)
// (:Reimbursement)-[:PAID_TO]->(:Employee)

// ============================================================
// REFERENCE DATA RELATIONSHIPS
// ============================================================

// (:ExchangeRate)-[:FROM_CURRENCY]->(:Currency)
// (:ExchangeRate)-[:TO_CURRENCY]->(:Currency)
// (:Merchant)-[:LOCATED_IN]->(:Country)
// (:PerDiemRate)-[:FOR_COUNTRY]->(:Country)
// (:MileageRate)-[:FOR_COUNTRY]->(:Country)

// ============================================================
// AUDIT RELATIONSHIPS
// ============================================================

// (:AuditEvent)-[:PERFORMED_BY]->(:Employee)
// (:AuditEvent)-[:AFFECTED]->(:ExpenseReport)  -- or any auditable node
// (:AuditEvent)-[:TRIGGERED_BY]->(:AuditEvent) -- causation chain

// ============================================================
// FRAUD DETECTION RELATIONSHIPS
// ============================================================

// (:FraudAlert)-[:FLAGS]->(:ExpenseLine)
// (:FraudAlert)-[:RELATED_TO]->(:FraudAlert)         -- connected fraud patterns
// (:ExpenseLine)-[:DUPLICATE_OF {similarity}]->(:ExpenseLine)
// (:Employee)-[:SHARES_ADDRESS_WITH]->(:Merchant)     -- collusion signal
// (:Employee)-[:FREQUENT_VISITOR {visit_count, total_spend}]->(:Merchant)

// ============================================================
// BUDGET RELATIONSHIPS
// ============================================================

// (:Budget)-[:FOR_DEPARTMENT]->(:Department)
// (:Budget)-[:FOR_CATEGORY]->(:ExpenseCategory)
// (:Budget)-[:SET_BY]->(:Organisation)
```

### Sample Data Creation

```cypher
// Create an organisation with departments
CREATE (acme:Organisation {
    id: randomUUID(), name: 'Acme Corp', base_currency: 'USD',
    fiscal_year_start: 1, timezone: 'America/New_York',
    created_at: datetime()
})
CREATE (eng:Department {
    id: randomUUID(), name: 'Engineering', cost_centre: 'ENG-100',
    created_at: datetime()
})
CREATE (sales:Department {
    id: randomUUID(), name: 'Sales', cost_centre: 'SAL-200',
    created_at: datetime()
})
CREATE (eng)-[:PART_OF]->(acme)
CREATE (sales)-[:PART_OF]->(acme)

// Create employees with management chain
CREATE (cfo:Employee {
    id: randomUUID(), email: 'cfo@acme.com', full_name: 'Jane Torres',
    role: 'finance_admin', is_active: true, created_at: datetime()
})
CREATE (mgr:Employee {
    id: randomUUID(), email: 'mgr@acme.com', full_name: 'Bob Kim',
    role: 'approver', is_active: true, created_at: datetime()
})
CREATE (emp:Employee {
    id: randomUUID(), email: 'alice@acme.com', full_name: 'Alice Chen',
    role: 'employee', is_active: true, default_currency: 'USD',
    created_at: datetime()
})
CREATE (cfo)-[:BELONGS_TO]->(acme)
CREATE (mgr)-[:BELONGS_TO]->(acme)
CREATE (emp)-[:BELONGS_TO]->(acme)
CREATE (mgr)-[:WORKS_IN]->(eng)
CREATE (emp)-[:WORKS_IN]->(eng)
CREATE (cfo)-[:MANAGES]->(mgr)
CREATE (mgr)-[:MANAGES]->(emp)

// Create a merchant
CREATE (merchant:Merchant {
    id: randomUUID(), name: 'Tokyo Grand Hotel', mcc_code: '7011',
    category_group: 'Lodging', country: 'JP', city: 'Tokyo',
    is_flagged: false, created_at: datetime()
})

// Create an expense report with full lifecycle
CREATE (report:ExpenseReport {
    id: randomUUID(), title: 'Tokyo Client Visit Q1', purpose: 'Client meetings',
    trip_start_date: date('2026-03-10'), trip_end_date: date('2026-03-13'),
    status: 'approved', total_amount: 2450.00, total_currency: 'USD',
    submitted_at: datetime('2026-03-14T09:00:00Z'),
    approved_at: datetime('2026-03-15T11:30:00Z'),
    created_at: datetime()
})
CREATE (emp)-[:SUBMITTED]->(report)

CREATE (line1:ExpenseLine {
    id: randomUUID(), description: 'Hotel 3 nights', expense_date: date('2026-03-10'),
    amount: 1800.00, currency: 'USD', converted_amount: 1800.00,
    is_billable: true, project_code: 'PROJ-42', created_at: datetime()
})
CREATE (report)-[:CONTAINS]->(line1)
CREATE (line1)-[:INCURRED_AT]->(merchant)
```

### Key Analytical Queries

```cypher
// ============================================================
// FRAUD DETECTION: Find employees sharing merchants with unusual patterns
// ============================================================
MATCH (e1:Employee)-[:SUBMITTED]->(:ExpenseReport)-[:CONTAINS]->(l1:ExpenseLine)
      -[:INCURRED_AT]->(m:Merchant)<-[:INCURRED_AT]-(l2:ExpenseLine)
      <-[:CONTAINS]-(:ExpenseReport)<-[:SUBMITTED]-(e2:Employee)
WHERE e1 <> e2
  AND l1.expense_date = l2.expense_date
  AND m.is_flagged = false
WITH m, collect(DISTINCT e1) AS employees, count(DISTINCT l1) AS claim_count
WHERE size(employees) >= 3
RETURN m.name, m.city, size(employees) AS employee_count, claim_count
ORDER BY claim_count DESC;

// ============================================================
// FRAUD DETECTION: Circular approval patterns (approver approves
// their own manager's expenses through delegation)
// ============================================================
MATCH path = (approver:Employee)-[:MANAGES*1..5]->(subordinate:Employee)
      -[:SUBMITTED]->(report:ExpenseReport)-[:REQUIRES_APPROVAL]->
      (step:ApprovalStep)-[:ASSIGNED_TO]->(approver)
RETURN approver.full_name, subordinate.full_name, report.id, report.total_amount,
       length(path) AS chain_length;

// ============================================================
// ORGANISATIONAL SPEND: Total spend under a VP's entire org tree
// ============================================================
MATCH (vp:Employee {email: 'vp-eng@acme.com'})-[:MANAGES*0..10]->(team_member:Employee)
      -[:SUBMITTED]->(report:ExpenseReport)
WHERE report.status IN ['approved', 'paid']
  AND report.trip_start_date >= date('2026-01-01')
RETURN vp.full_name AS manager,
       count(DISTINCT team_member) AS team_size,
       count(report) AS report_count,
       sum(report.total_amount) AS total_spend;

// ============================================================
// APPROVAL CHAIN: Find the full approval path for a report
// ============================================================
MATCH (report:ExpenseReport {id: $reportId})
      -[:REQUIRES_APPROVAL]->(first:ApprovalStep)
MATCH path = (first)-[:NEXT_STEP*0..10]->(step:ApprovalStep)
MATCH (step)-[:ASSIGNED_TO]->(approver:Employee)
OPTIONAL MATCH (step)-[:DELEGATED_TO]->(delegate:Employee)
RETURN step.step_order, approver.full_name AS approver,
       delegate.full_name AS delegate,
       step.status, step.decided_at
ORDER BY step.step_order;

// ============================================================
// POLICY EVALUATION: Which rules apply to this expense?
// ============================================================
MATCH (line:ExpenseLine {id: $lineId})<-[:CONTAINS]-(report:ExpenseReport)
      <-[:SUBMITTED]-(emp:Employee)-[:WORKS_IN]->(dept:Department)
      -[:PART_OF]->(org:Organisation)-[:HAS_POLICY]->(policy:Policy)
      -[:INCLUDES_RULE]->(rule:PolicyRule)
WHERE policy.is_active = true
  AND policy.effective_from <= date()
  AND (policy.effective_to IS NULL OR policy.effective_to >= date())
OPTIONAL MATCH (rule)-[:APPLIES_TO_CATEGORY]->(cat:ExpenseCategory)
OPTIONAL MATCH (rule)-[:APPLIES_TO_DEPARTMENT]->(rule_dept:Department)
WHERE cat IS NULL OR (line)-[:CATEGORISED_AS]->(cat)
  AND (rule_dept IS NULL OR rule_dept = dept)
RETURN rule.rule_type, rule.human_readable, rule.threshold_amount,
       rule.severity, cat.name AS applies_to_category;

// ============================================================
// MERCHANT INTELLIGENCE: Build merchant visit frequency network
// ============================================================
MATCH (emp:Employee)-[:SUBMITTED]->(:ExpenseReport)-[:CONTAINS]->
      (line:ExpenseLine)-[:INCURRED_AT]->(merchant:Merchant)
WITH emp, merchant, count(line) AS visits, sum(line.amount) AS total_spent
WHERE visits >= 3
MERGE (emp)-[freq:FREQUENT_VISITOR]->(merchant)
SET freq.visit_count = visits, freq.total_spend = total_spent;

// ============================================================
// DUPLICATE DETECTION: Find potential duplicate expenses
// by amount, merchant, and date proximity
// ============================================================
MATCH (l1:ExpenseLine)-[:INCURRED_AT]->(m:Merchant)<-[:INCURRED_AT]-(l2:ExpenseLine)
WHERE l1.id < l2.id
  AND l1.amount = l2.amount
  AND l1.currency = l2.currency
  AND abs(duration.inDays(l1.expense_date, l2.expense_date).days) <= 1
MATCH (l1)<-[:CONTAINS]-(r1:ExpenseReport)<-[:SUBMITTED]-(e1:Employee)
MATCH (l2)<-[:CONTAINS]-(r2:ExpenseReport)<-[:SUBMITTED]-(e2:Employee)
RETURN e1.full_name, e2.full_name, m.name, l1.amount, l1.currency,
       l1.expense_date, l2.expense_date,
       CASE WHEN e1 = e2 THEN 'same_employee' ELSE 'different_employees' END AS type;
```

## Trade-offs

**Strengths:**

- **Fraud detection is dramatically more efficient.** Queries that require 5-6 table JOINs with recursive CTEs in PostgreSQL become simple 3-4 hop traversals in Cypher. Neo4j's index-free adjacency means each relationship traversal takes constant time regardless of dataset size. Real-world fraud detection benchmarks show 10-100x faster query times compared to relational databases for pattern-matching across connected data.
- **Organisational hierarchy queries are natural.** Variable-length path expressions (`-[:MANAGES*0..10]->`) replace recursive CTEs. Finding "all expenses submitted by anyone in Sarah's org" is a single traversal, not a materialised closure table.
- **Schema flexibility without JSONB.** Adding a new relationship type (e.g., `SHARES_DEVICE_WITH` for fraud detection) requires no schema migration — just create the relationships. New node labels and properties can be added at any time.
- **Visual investigation.** Neo4j's built-in browser and Bloom visualisation tool let fraud analysts and finance managers explore expense networks visually — seeing connections between employees, merchants, and transactions that are invisible in tabular data.
- **Graph algorithms out of the box.** Neo4j's Graph Data Science library provides community detection (find clusters of suspicious activity), PageRank (identify influential merchants), shortest path (trace approval chains), and similarity algorithms (find employees with similar spending patterns) without custom implementation.

**Weaknesses:**

- **Aggregation and reporting are weaker.** Queries like "total spend by category by month for the last 12 months across all departments" — the bread and butter of finance dashboards — are significantly slower and more awkward in Cypher than in SQL. Graph databases are optimised for traversal, not for aggregating large flat result sets.
- **ACID transactions across multiple nodes are costlier.** While Neo4j supports ACID transactions, write throughput for bulk operations (e.g., importing 50,000 card transactions) is lower than PostgreSQL. Graph databases optimise for read-heavy, relationship-traversal workloads.
- **Monetary precision.** Neo4j stores numbers as Java doubles (64-bit floating point) by default. Financial calculations require careful handling to avoid floating-point rounding errors. Storing amounts as integer cents (multiplied by 100) or as strings with application-level parsing is the standard workaround, but it adds complexity.
- **Operational maturity.** PostgreSQL has decades of enterprise tooling for backup, point-in-time recovery, connection pooling, and monitoring. Neo4j's operational tooling is improving but less mature. Fewer DBAs have Neo4j experience, increasing operational risk.
- **Ecosystem and ORM support.** Most application frameworks have mature PostgreSQL/relational ORM support. Neo4j drivers exist for major languages but lack the depth of tooling (migration frameworks, schema management, seeding tools) available for relational databases.
- **Storage efficiency.** Graph databases store each relationship as an explicit record with forward and backward pointers. For a system with millions of expense lines, each having 5-8 relationships, storage requirements can be 3-5x higher than the equivalent relational schema.

## Scalability Considerations

- **Neo4j Aura (managed cloud)** handles operational scaling, replication, and backups. For self-hosted deployments, Neo4j Enterprise supports causal clustering with read replicas.
- **Sharding is limited.** Neo4j Fabric (introduced in 4.0+) supports data federation across multiple databases but is not transparent horizontal sharding like PostgreSQL's Citus. For a single-organisation expense system this is rarely a concern; for a multi-tenant SaaS serving thousands of organisations, consider sharding by organisation into separate Neo4j databases.
- **Graph Data Science projections** run on in-memory graph projections, not the live database. Large fraud analysis runs (community detection across all expenses) require sufficient RAM — budget approximately 8-16 bytes per node and 16-32 bytes per relationship for the in-memory projection.
- **Write scaling** for high-volume card transaction feeds (thousands per second) may require a write-ahead buffer (e.g., Kafka) that batches and bulk-loads transactions into Neo4j, rather than individual Cypher CREATE statements.
- **Archival strategy:** Move expense data older than the retention period (e.g., 7 years for tax compliance) to a read-only Neo4j instance or export to a relational archive. The primary graph stays performant with active data only.

## Recommended Architecture: Graph as Analytical Layer

The strongest deployment pattern for this domain uses Neo4j as a **specialised analytical and fraud detection layer** alongside a relational primary store (suggestions 1 or 3):

```
                    +-----------------------+
                    |   PostgreSQL (Primary) |
                    |   - Transactional CRUD |
                    |   - Financial reporting|
                    |   - ACID guarantees    |
                    +-----------+-----------+
                                |
                      CDC / Event Stream
                      (Debezium / Kafka)
                                |
                    +-----------v-----------+
                    |   Neo4j (Analytical)   |
                    |   - Fraud detection    |
                    |   - Approval routing   |
                    |   - Org hierarchy      |
                    |   - Spend networks     |
                    |   - Visual investigation|
                    +-----------------------+
```

This hybrid architecture plays to each database's strengths: PostgreSQL handles transactional integrity and financial reporting; Neo4j handles relationship-intensive queries that would be prohibitively expensive in SQL. A change-data-capture pipeline (Debezium or application-level events) keeps the graph synchronised with the relational source of truth.

## Migration Path

- **From suggestion 1 (normalised relational):** Export relational data into Neo4j using `neo4j-admin import` or the `LOAD CSV` command. Map foreign keys to relationships. The relational schema remains the system of record; Neo4j is added as a read-only analytical layer.
- **From suggestion 2 (event sourcing):** Build a graph projection subscriber that consumes domain events and creates/updates nodes and relationships in Neo4j. This is the cleanest integration path — every `ExpenseLineAdded`, `ApprovalGranted`, or `FraudScoreCalculated` event translates directly into graph mutations.
- **From suggestion 3 (hybrid JSONB):** Extract JSONB fields (fraud_analysis, travel_details, policy_check) and materialise them as first-class nodes and relationships in the graph, enabling traversal queries that are impossible against nested JSON.
- **To a standalone graph system:** If the team determines that the graph model should become the primary store, begin by migrating write operations for fraud detection and approval workflows to Neo4j first, keeping PostgreSQL for financial ledger operations. Full migration is possible but rarely advisable for financial systems that require SQL-based regulatory reporting.
