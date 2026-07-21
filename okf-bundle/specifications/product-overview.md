---
type: Specification
title: Product Overview
description: What Estatly is, the deliverable surfaces, and the standing constraints from the requirements brief.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [overview, scope]
---

# Purpose

Estatly manages a **society** (Indian residential society / co-operative): its projects, members and their asset ownership, employees, governance (Executive Committee), vendors and work orders, and society finance with Indian statutory compliance. The product is similar in scope to [[Urban.ai]] but uses its own architecture.

Source of truth for scope: this bundle — the original requirements brief (`__init.md`, retired 2026-07-20) is fully absorbed into these specs.

# Deliverable surfaces

| Surface | Users | Notes |
|---|---|---|
| Admin panel | Management, EC, designated employees | Web app — day-to-day society administration |
| Landing site | Public | Marketing / public web presence |
| Mobile app (iOS + Android) | Members | Includes payment-gateway access for paying charges |
| Backend | — | AWS, built with **AWS Blocks** — see [AWS Blocks backend](/architecture/aws-blocks-backend.md) |
| Payment gateway | Members (via mobile app) | Built on third-party payment software — see [Payments](/specifications/payments.md) |

# Standing constraints

- **Compliance:** finance data must comply with local laws of India; **GST compliance** is mandatory.
- **Tally:** finance data must be **exportable to Tally**.
- **Document retention:** all bills, vouchers, receipts, challans, and images are kept in scanned form in bulk storage, linked from ledger entries — see [Finance & Compliance](/specifications/finance-and-compliance.md).
- **Backend platform:** AWS, using AWS Blocks (mandated by the brief).

# Core specs

- [Domain Model](/specifications/domain-model.md) — entities and relationships.
- [Governance & Roles](/specifications/governance-and-roles.md) — members, management, EC, employees.
- [Vendor Workflow](/specifications/vendor-workflow.md) — work order → invoice → verification → approval.
- [Finance & Compliance](/specifications/finance-and-compliance.md) — books, ledgers, GST, Tally, document storage.
- [Payments](/specifications/payments.md) — member-facing payment gateway.
- [Platforms](/specifications/platforms.md) — the client surfaces.

# Decisions

- **Single society per deployment** (decided 2026-07-20): each society runs its own backend stack with its own data store — no cross-society tenancy in the data model, no `society_id` scoping. Onboarding a new society means provisioning a new deployment ([AWS Blocks backend](/architecture/aws-blocks-backend.md) — infrastructure-from-code makes this repeatable). Society-level settings are deployment configuration.

- **Launch scope** (decided 2026-07-20): India, AWS `ap-south-1`. **English-only UI in v1** (Hindi and regional languages later). Statutory compliance is **parameterized by state** and implemented first for the pilot society's state — no multi-state machinery in v1 beyond configuration.
- **Urban.ai relationship** (decided 2026-07-20): reference material only — no shared code or components.
