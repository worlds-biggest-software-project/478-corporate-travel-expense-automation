# Corporate Travel Expense Automation

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that automates the entire receipt-to-reimbursement workflow -- replacing manual data entry, rigid rule engines, and opaque enterprise pricing with intelligent agents that handle OCR extraction, policy interpretation, and ERP synchronisation out of the box.

Corporate Travel Expense Automation is built for finance teams, accounts payable departments, and travel managers at mid-sized to large organisations who are tired of overpaying for clunky incumbent tools. It captures receipts via mobile, automatically extracts and categorises expense data using vision-language AI models, enforces travel policies through LLM-interpreted rules rather than brittle rule engines, and posts approved expenses to ERP systems in near-real-time.

---

## Why Corporate Travel Expense Automation?

- **Enterprise incumbents are expensive and opaque.** SAP Concur dominates the large-enterprise market but requires negotiated contracts, costly implementation partners, and long onboarding timelines. 88% of its TrustPilot reviews are one-star, citing an unintuitive interface.
- **SMB tools lack agentic AI.** Expensify and Zoho Expense offer reasonable pricing ($3--$18/user/mo) but their automation stops at basic OCR and rule-based policy checks. Sophisticated agentic workflows -- where receipt capture triggers the entire downstream flow without employee action -- remain locked behind enterprise pricing.
- **No credible open-source alternative exists.** Odoo Expense is the only open-source option with expense management features, but it lacks AI-grade receipt extraction, has significantly inferior OCR quality, and requires custom development for any intelligent automation.
- **Newer entrants are geographically limited.** Ramp, Brex, and the Expensify Card are largely US-only, leaving non-US mid-market organisations underserved. Spendesk addresses Europe but lacks the AI maturity of US-born platforms.
- **Non-English receipt handling is poor across the board.** AI categorisation and OCR accuracy drops significantly for non-Latin-script receipts (Japanese, Arabic, Chinese), leaving multinational organisations with manual workarounds.

---

## Key Features

### Intelligent Receipt Capture

- Mobile receipt capture powered by vision-language model OCR, handling non-English, handwritten, and low-quality receipts
- Real-time accuracy preview showing extracted merchant, date, amount, currency, and category
- Multiple upload channels: mobile camera, email forwarding, bulk desktop upload
- Corporate card feed integration with automatic transaction matching (Visa, Mastercard, Amex)

### AI-Powered Policy and Categorisation

- Automated expense categorisation against configurable chart-of-accounts using ML plus contextual signals (merchant metadata, MCC codes, employee context)
- LLM-interpreted policy engine that handles edge cases and ambiguous situations beyond rigid rule matching
- Pre-submit AI audit agent that validates completeness and policy compliance before the report reaches an approver
- Duplicate receipt detection and anomaly-based fraud scoring

### Agentic Expense Workflow

- Automated expense report assembly: group receipts by trip or time period and auto-submit without employee action
- Multi-level approval workflows with configurable delegation, mobile approval, and escalation for stalled submissions
- AI-driven approval routing based on context rather than just org-chart rules
- AI-generated expense report narrative summaries for finance managers

### Financial Integration

- Near-real-time accounting sync with QuickBooks Online, Xero, and NetSuite via webhooks (not nightly batch)
- Multi-currency support with real-time FX conversion using open exchange rate sources
- GPS-based mileage tracking with IRS/HMRC rate calculation
- Per-diem calculator with geo-aware rates

### Analytics and Audit

- Spend dashboards by category, department, cost centre, and period with budget-vs-actual tracking
- Drill-down to individual transactions from any dashboard view
- Audit trail and compliance reporting for finance and legal teams
- Spend pattern anomaly alerts and random audit sampling with evidence-pack generation

---

## AI-Native Advantage

This project goes beyond bolting AI onto a traditional expense tool. Vision-language models replace rule-based OCR, delivering accurate extraction from handwritten, non-English, and damaged receipts where incumbents fail. LLM-interpreted policy rules replace brittle rule engines, correctly handling the ambiguous edge cases that generate false violations in conventional systems. Agentic workflows orchestrate the entire receipt-to-reimbursement pipeline autonomously -- grouping expenses, validating policy, routing approvals, and gathering missing information -- so employees can stop doing data entry and finance teams can focus on strategic spend analysis rather than chasing missing receipts.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment models, ensuring organisations that require data sovereignty can run the system on their own infrastructure. Integration follows standard OAuth, REST, and JSON patterns -- no proprietary protocols. Corporate card feeds connect via published commercial card programme APIs (Visa, Mastercard, Amex). Accounting integrations use webhook-driven near-real-time sync rather than batch imports. The architecture draws on established open methods for receipt OCR, ML categorisation, and anomaly detection, avoiding any proprietary techniques or data schemas from incumbents.

---

## Market Context

The corporate travel and expense management market is dominated by SAP Concur at the enterprise tier and fragmented among Expensify, Ramp, Brex, Navan, and others in the mid-market. Pricing ranges from $3/user/month (Zoho Expense) to opaque enterprise contracts exceeding $50k in implementation costs alone (Coupa). Primary buyers are finance leaders, AP teams, and travel managers at organisations with 50+ employees who process regular employee expenses. Domain availability and market demand are both rated High, with a complexity score of 5/10.

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
