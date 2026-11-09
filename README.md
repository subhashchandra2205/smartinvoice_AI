# SmartInvoice AI — invoice PDF to governed decision

Multi-tenant accounts-payable automation: upload an invoice PDF, verify its bytes
server-side, extract fields (with OCR for scanned pages), match the vendor and
purchase order, validate money and tax, then require an **independent human
approval** before any purchase-order balance is consumed.

Nothing is auto-approved. No payment is executed.

## What it does

| Area | Behaviour |
|---|---|
| Identity | Email sign-in, invite-only registration, organizations with five roles: Admin, Finance Manager, Invoice Processor, Reviewer, Read-only |
| Invoices | Authenticated PDF upload, byte-level verification (20 MiB max), server-side text extraction, AI Gateway OCR for image-only pages, durable organization/hash jobs with leases and retry |
| Matching | Normalized vendor identity plus an explicit PO reference; ambiguous vendor candidates never auto-select a PO; mismatches and excess spend go to review |
| Money | Exact decimal arithmetic (decimal.js), subtotal + tax reconciliation to grand total, GSTIN checksum, IGST/CGST/SGST component reconciliation |
| PO balance | Concurrency-safe reservation: organization advisory lock + `SELECT … FOR UPDATE`, cumulative approved spend recomputed inside one transaction, saved tolerance respected |
| Decisions | Every invoice needs a separate authorized approver — uploaders cannot approve their own work; audited rejection and field correction with retained evidence |
| Records | Organization-scoped history with server-side paging (25 rows), filters, on-demand evidence, safe CSV export; approved-only spending/GST reports |
| Storage | Private `invoice-pdfs` bucket under `<organization_id>/…`; direct browser uploads and deletes are denied; admin-only SHA-256-verified PDF backup pages |

## Stack

TanStack Start v1 (React 19, TypeScript), Tailwind CSS v4, shadcn/ui, Vitest,
Drizzle Kit for migration files, and a managed Postgres backend with row-level
security, auth and private object storage.

## Setup

Requires Node.js 22+ or Bun.

```sh
cp .env.example .env      # fill in your own backend values
npm install
npm run dev               # http://localhost:8080
```

Environment variables (browser values are public after build; never commit a
secret/service-role key):

| Variable | Purpose |
|---|---|
| `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_PROJECT_ID` | Browser client |
| `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_PROJECT_ID` | Server-side client |
| `LOVABLE_DB_MIGRATION_URL` | Drizzle Kit migration generation only |

## Database

`drizzle/migrations/` holds the full additive history (0000–0023): foundation
tables, privileges, invitations, invite-only registration, shared invoice
processing, vendor/PO management, concurrency-safe approval, trusted PDF upload
boundary, production policy and workflow, independent decisions, durable jobs,
audited corrections, decision guards, bounded history and report aggregates.

`drizzle/schema.ts` is the Drizzle mirror of that schema. Apply migrations
through your backend's migration path; every business table carries
`organization_id` and is protected by RLS with membership checks that cannot be
self-granted from the client.

## Tests

```sh
npm test                                   # application unit tests
python3 src/test/database/approval_concurrency.py   # real Postgres approval/role tests
python3 src/test/database/history_load.py           # 10,050-row history + aggregate tests
python3 src/test/database/organization_access.py    # organization isolation checks
```

The Python tests run against a disposable local PostgreSQL instance and never
touch application data.

## Documentation

- `PRODUCTION_HANDOVER.md` — delivered features, verification results, what is
  not yet production-certified, and the pre-deployment checklist
- `PRODUCTION_AUDIT.md` — read-only architecture and gap audit
- `PDF_VERIFICATION.md` — upload limits and extraction provenance
- `BACKUP_RECOVERY.md` — separate database and PDF backup runbook, retention
  recommendation
- `AGENTS.md` — repository rules for automated contributors
- `roadmap.md` — open work items

## Limitations

OCR confidence is model-reported, not independently calibrated. GSTIN checksum
validation is not government registry verification. Tax-jurisdiction completeness,
calibrated low-quality-scan handling, hosted concurrent-user benchmarks, and
email delivery (invitation, confirmation, password reset) are not yet verified.
Retention automation and automatic deletion of abandoned files are intentionally
disabled pending an approved retention policy.
