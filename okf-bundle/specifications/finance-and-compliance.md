---
type: Specification
title: Finance & Compliance
description: The four books of account, scanned-document storage, Indian statutory compliance, GST, and Tally export.
status: Draft
version: 0.2.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [finance, ledger, gst, tally, compliance]
---

# Books of account (from the brief)

| Book | Draft purpose |
|---|---|
| **Bank Book** | All transactions through the society's bank account(s) |
| **Cash Book** | Cash receipts and cash payments |
| **Payment Ledger** | Member-side: charges raised and payments received per member (see [Payments](/specifications/payments.md)) |
| **Expense Ledger** | Society-side outflows: vendor invoices (see [Vendor Workflow](/specifications/vendor-workflow.md)), employee salaries, other expenses |

# Document storage

All **bills, vouchers, receipts, challans, and images** are kept in scanned form in bulk storage (S3 via the FileBucket Block — see [AWS Blocks backend](/architecture/aws-blocks-backend.md)) and **linked from the corresponding ledger entries**. A ledger entry should be able to carry multiple document links; documents should be immutable once attached (compliance evidence). These are records in the society-wide document registry — see [Document Management](/specifications/document-management.md); a ledger-linked document can never be archived.

# Compliance requirements

- **Local laws of India** — the books must satisfy the applicable co-operative society / apartment-association accounting rules of the society's state.
- **GST compliance** — GST treatment of maintenance charges and vendor invoices; GSTIN capture; GST-compliant invoice/receipt formats; return-ready reporting.
- **Tally export** — finance data must be exportable to Tally (scope decided below: day-book vouchers as Tally-importable XML).

# Design implications (draft)

- Use double-entry semantics internally even though the brief names four books — it is the only reliable way to keep Bank/Cash books and the two ledgers consistent and Tally-exportable.
- Money as exact decimal (never floats); INR; Indian financial year (Apr–Mar) reporting periods.
- Ledger entries are append-only with an audit trail; corrections are reversing entries, not edits.

# Decisions (2026-07-20)

- **Book representation:** the four books are **derived views over a single append-only double-entry journal** — they are not separately maintained stores. Bank accounts and cash are ledger accounts; the Bank Book and Cash Book are projections of postings touching those accounts, and the Payment and Expense Ledgers are counterparty projections (by member, by vendor/payee). One posting keeps all four books consistent by construction and maps directly to Tally voucher export. The books remain first-class in the admin panel UI and the [Admin Panel API](/api/admin/public-api.md) — only the storage model is the single journal.
- **State law:** compliance formats are **parameterized per deployment** (state is deployment configuration, fitting single-society-per-deployment). Implemented first for the pilot society's state; no multi-state machinery in v1.
- **GST registration:** the society's **GSTIN is optional per-society configuration**. Unregistered societies get plain receipts; setting a GSTIN activates GST-compliant receipt/invoice formats and report fields. v1 supports both.
- **Tally export scope:** **day-book vouchers as Tally-importable XML** for the requested period, plus the ledger masters needed to import them. No full chart-of-accounts synchronization.
- **Receipt numbering:** auto-numbered, **gapless, per financial year**, with a society prefix (e.g. `<SOC>/2026-27/000123`). Receipts are system-generated under the Treasurer's authority — Treasurer's name printed, no signature image in v1.
