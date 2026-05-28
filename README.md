# StreamDash MVP

## Product scope

### Goal

Unify multiple revenue streams (Stripe + PayPal + manual entries) into one place
with consistent tagging, monthly profit, and exportable reports, without becoming
full accounting software.

### Non-goals (MVP)

- No invoicing, billing, or charge creation.
- No double-entry general ledger, depreciation, payroll, inventory, or 1099/W‑2
  filing.
- No accrual revenue recognition beyond basic “cash received vs refunded/disputed.”

## Personas and use cases

- Solo founder with 2–10 income streams wants monthly profit by stream and “how
  much to set aside for taxes.”
- Agency owner wants to tag income/expenses per client/project and see profitability.
- Indie hacker wants a single dashboard across Stripe + PayPal without building
  spreadsheets.

## Functional requirements

1. Accounts, orgs, and security

- Users can sign up/sign in (email magic link or OAuth) and enable 2FA.
- Users can create one “Workspace” and invite at least 1 additional user
  (admin/viewer roles).
- Audit log for: user invites, integration connects/disconnects, rule changes, exports.

1. Data sources (integrations)

## Stripe (required in MVP)

- Connect a Stripe account (OAuth).
- Ingest transactions needed to compute net cash movement: charges/payments, refunds,
  disputes, fees, payouts/balance changes (use Stripe’s balance-transaction worldview
  so fees/refunds are visible as distinct entries). Stripe documents that disputes
  and dispute fees debit your Stripe balance during the dispute process, so these
  must be represented.[^1][^2]
- Webhooks for freshness (minimum: new successful payments, refunds, disputes) plus
  scheduled backfill sync.

### PayPal (optional but strongly recommended)

- Connect a PayPal account and pull transaction history via PayPal’s Transaction
  Search / reporting endpoint.[^3][^4]
- Daily incremental sync + manual “resync date range” backfill.

### Manual sources (required)

- CSV import for bank/credit card exports and affiliate payouts.
- Manual “cash entry” form for single rows (e.g., check received, cash expense).

### 3) Normalized transaction model

Store every money movement as an immutable “Transaction” record:

- Required fields: id, source (Stripe/PayPal/CSV/manual), external_id, date, amount,
  currency, direction (in/out), type (income/expense/fee/refund/dispute/payout/adjustment),
  description, counterparty (if known), raw_payload JSON.
- Deduplication: enforce unique (source, external_id) and idempotent sync.

### 4) Tagging and classification (the core value)

- Users can define:
  - Streams (a.k.a. products/projects): e.g., “Agency”, “SaaS A”, “Affiliates”,
    “One-offs”.
  - Categories: “Revenue”, “COGS”, “Ops”, “Marketing”, “Software”, “Taxes”, “Owner
    draw”, etc.
- Rule engine (MVP rules):
  - Match on description contains / regex, counterparty, Stripe reporting category/type,
    PayPal event codes (if available), amount sign, currency.
  - Rule actions: assign stream, category, and “tax deductible” flag.
- Manual overrides always win, but rule changes can be re-applied to a selected
  date range.

### 5) Dashboard and reporting

- Overview page:
  - This month: gross inflow, refunds/disputes, fees, net inflow, expenses, net
    profit.
  - “Tax set-aside” estimate = net profit × configured effective tax rate.
- Streams page:
  - Monthly net profit by stream (table is fine in MVP).
  - Drill-down to transactions filtered by stream.
- Transactions list:
  - Fast filters: date range, source, stream, category, unclassified, amount range.
  - Bulk actions: assign stream/category, mark deductible/non-deductible.

### 6) Exports (accountant-friendly)

- CSV export of transactions with stream/category columns and original source ids.
- “Monthly summary” export: totals by month × stream × category.
- QuickBooks-compatible journal export is **not** required in MVP, but design the
  export schema so you can add it later.

### 7) Close checklist (behavioral lock-in)

- Monthly “Close” flow:
  - Show: count of unclassified transactions, missing categories, negative anomalies.
  - Button to “Mark month as closed” (freezes reports unless user re-opens month).
- Reconciliation (MVP-lite):
  - For Stripe: compare sum of payouts vs computed net for period (surface mismatch
    warning rather than full reconciliation).

## Non-functional requirements

### Performance

- Transactions list loads under 2 seconds for up to 50k transactions/workspace
  (with pagination and indexed filters).
- Background sync jobs must be resumable and idempotent.

### Security \& privacy

- Store minimal secrets; use OAuth tokens, encrypt at rest, rotate/refresh tokens.
- Raw payload retention policy configurable (e.g., keep 12–24 months) and deletable
  per workspace.

### Reliability

- Webhook processing must be idempotent (same event can arrive multiple times).
- Backfill must handle replays and late-arriving refunds/disputes (Stripe disputes/fees
  can occur after initial payment).[^2]

## MVP acceptance criteria (testable)

- User connects Stripe and sees at least: payments, refunds, disputes/fees, and
  payouts reflected as transactions within 10 minutes of connection.[^1][^2]
- User can create 3 streams and 10 rules; 80%+ of imported transactions get auto
  -classified on a typical account after rules are added.
- User can export a CSV of all transactions and a monthly rollup for a selected
  date range.
- User can set an effective tax rate and see an updated tax set-aside number on
  the overview.

If you tell me your intended stack (Node/TS? Symfony? Rust?) and your target “first
integration set” (Stripe-only vs Stripe+PayPal), I can turn this into a week-by-week
build plan plus a minimal DB schema.

[^1]: <https://docs.stripe.com/reports/balance-transaction-types>

[^2]: <https://docs.stripe.com/disputes/how-disputes-work>

[^3]: <https://docs.paypal.ai/growth/reports/api>

[^4]: <https://developer.paypal.com/docs/api/transaction-search/v1/>

## License

MIT, see [LICENSE](LICENSE).

## Contributing

PRs welcome. Please open an issue first for major changes.
