---
type: Specification
title: Domain Model
description: Estatly's core entities — Society, Project, Member, Ownership, Employee, Vendor, Work Order, Invoice — and how they relate.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [domain, entities, data-model]
---

# Entities

| Entity | Description | Key relationships |
|---|---|---|
| **Society** | Root entity. The residential society being managed. | has many Projects, Members, Employees, Vendors; managed by Management |
| **Project** | A development within the society (e.g. a phase, tower, or plotted layout). | belongs to Society; has many Ownerships; owned and managed by its PC |
| **Member** | A person with membership in the society. | belongs to Society; has zero or more Ownerships; may be part of Management / EC |
| **Ownership** | A member's holding of an Asset: **flat, plot, villa, or dividend**. References `asset_id` (decided 2026-07-20 — [Asset Management](/specifications/asset-management.md)); previously embedded `asset_type`/`asset_label`. | joins Member ↔ Asset (and thereby Project); drives Charges |
| **Charge** | A levy on a member for owning an asset, or a **recurring maintenance fee** on an owned asset. | belongs to Member (via Ownership); settled via [Payments](/specifications/payments.md); posts to ledgers |
| **Employee** | Salaried staff of the society; **not a member**. | belongs to Society; salary posts to the Expense Ledger |
| **Management** | The governing group; all managers are members. | subset of Members |
| **EC (Executive Committee)** | Office-bearing subset of Management. Roles: President, Vice President, General Secretary, Treasurer, Executive Member. | subset of Management; approves vendor invoices |
| **PC (Project Committee)** | Per-project governing group; owns and manages its Project (**added 2026-07-20**, not in the original brief). Flat membership + designated Chair. | subset of Members owning in the Project; appointed by the EC; one PC per Project — see [Governance & Roles](/specifications/governance-and-roles.md) |
| **Vendor** | External party doing contracted work. | belongs to Society; receives Work Orders; issues Invoices |
| **Work Order** | Authorization for a vendor to perform defined work. | issued to Vendor; fulfilled by one or more Invoices |
| **Invoice** | Vendor's bill for full or partial completion of a Work Order. | belongs to Work Order; verified by a designated Employee; approved by a designated EC subset |

Finance entities (Bank Book, Cash Book, Payment Ledger, Expense Ledger, scanned documents) are specified in [Finance & Compliance](/specifications/finance-and-compliance.md). The **Document** entity — attachable at Society, Project, or Member level, searchable — is specified in [Document Management](/specifications/document-management.md). The **Ticket** entity — member-raised finance / maintenance / records requests (**added 2026-07-20**, not in the original brief) — is specified in [Member Requests](/specifications/member-requests.md). The **BulletinPost** and **DirectMessage** entities — EC/PC board announcements and email/WhatsApp member messaging (**added 2026-07-20**, not in the original brief) — are specified in [Communication](/specifications/communication.md). The **Asset** entity — first-class registry of flats/plots/villas/dividend holdings with role-based visibility, referenced by Ownership via `asset_id` (**added and decided 2026-07-20**, not in the original brief) — is specified in [Asset Management](/specifications/asset-management.md).

# Member attributes (from the brief)

`member_id`, `joining_date`, `member_status`, `name`, `email`, `phone`, `address`.

# Member status lifecycle (decided 2026-07-20)

```
pending → active ⇄ suspended
              ↘      ↙
               ceased (terminal; carries cessation_reason)
```

| Status | Meaning | Rules |
|---|---|---|
| `pending` | Entered by management; admission not yet effective | No charges accrue; no app invite; cannot hold roles |
| `active` | Member in good standing | Charges accrue; full app access; `joining_date` set on admission |
| `suspended` | Rights suspended by EC action; reversible | Charges **continue to accrue**; may still log in and pay dues; cannot hold Management/EC roles |
| `ceased` | Membership ended — terminal | `cessation_reason` required: `resigned \| expelled \| deceased \| transferred`. No new charges; **outstanding dues survive** and are settled via management (offline receipts); no app access |

Deliberately **not** a status: **defaulter**. Financial standing is derived from outstanding charges at read time — encoding it in `member_status` would create a second source of truth that drifts from the Payment Ledger.

Cessation of a member who still holds ownerships: with transfer rules now decided (below), cessation requires the member's ownerships to be **transferred first**, except `deceased`, where ownerships may be retained pending succession.

# Relationship notes

- A member's ownership spans **zero or more** projects — membership does not require ownership.
- **Dividend** is an ownership type without a physical asset; charge rules for it likely differ from flat/plot/villa.
- Employees are disjoint from Members by definition; Management and EC are strictly subsets of Members. Enforce these invariants in the data model.
- Charges are per owned asset: a one-time/asset-based charge plus optional recurring maintenance fees. A member with several assets accrues several charge streams.

# Decisions (2026-07-20)

- **Ownership & joint owners:** each Ownership has a **single primary owner** who is billed; co-owner names may be stored as informational fields. No joint logins or split billing in v1.
- **Ownership transfer:** an admin action, allowed only when the ownership's **outstanding charges are settled**. Transfer closes the old ownership record and creates a new one for the new member — records are never rewritten, history is preserved.
- **Work Order contents:** scope (text), value, issue date, optional validity end date, and free-text notes. No structured milestones in v1 — partial completion is expressed through partial invoices ([Vendor Workflow](/specifications/vendor-workflow.md)).
- **Dividend ownership:** charged like any other asset type. Payouts are out of scope in v1 — if a society declares one, it is recorded as manual ledger entries, not product machinery.
