---
type: Specification
title: Governance & Roles
description: Membership, management, Executive Committee roles, employees, and the permission boundaries between them.
status: Draft
version: 0.7.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [governance, roles, rbac]
---

# Role hierarchy

```
Society
├── Members                      (may own assets; use the mobile app; pay charges)
│   ├── Management               (members who manage the society; use the admin panel)
│   │   └── EC — Executive Committee (office bearers)
│   │       ├── President
│   │       ├── Vice President
│   │       ├── General Secretary
│   │       ├── Treasurer
│   │       └── Executive Member(s)
│   └── PC — Project Committee   (per project: owns and manages that Project — added 2026-07-20)
├── Employees                    (salaried, NOT members; operational duties)
└── Vendors                      (external; work against Work Orders)
```

# Role capabilities (draft)

| Role | Draft capabilities |
|---|---|
| Member | View own ownerships, charges, payment history; pay via the [payment gateway](/specifications/payments.md) |
| Management | Administer members, projects, ownerships, charges; send **direct messages** to members via email/WhatsApp ([Communication](/specifications/communication.md), decided 2026-07-20) |
| EC (designated subset) | **Approve vendor invoices** — see [Vendor Workflow](/specifications/vendor-workflow.md); via the admin panel or the mobile approval inbox ([Platforms](/specifications/platforms.md)) |
| EC (all) | Post to the **society bulletin board**; send **direct messages** to members via email/WhatsApp ([Communication](/specifications/communication.md), added 2026-07-20); view **all assets in all projects** ([Asset Management](/specifications/asset-management.md), added 2026-07-20) |
| Treasurer | Presumed lead on the finance books — see [Finance & Compliance](/specifications/finance-and-compliance.md) |
| PC (per project) | **Owns and manages its project** (added 2026-07-20): project record, ownership data for the project, and project [documents](/specifications/document-management.md) incl. their `member_visible` flags. **Administration only — no financial authority** (see Decisions). Read-only mobile access via the `/pc` surface ([Mobile Public API](/api/mobile/public-api.md)), plus its single write: posting to the PC's **project bulletin board** ([Communication](/specifications/communication.md), decided 2026-07-20). Views **all assets in its project** ([Asset Management](/specifications/asset-management.md), added 2026-07-20) |
| Employee (designated) | **Verify vendor invoices**; operational data entry; grantable **project-scoped asset view** ([Asset Management](/specifications/asset-management.md), added 2026-07-20 — none by default) |
| Vendor | No system access (decided 2026-07-20) — a record, not a user; work orders and invoices are entered by employees via the admin panel ([Platforms](/specifications/platforms.md)) |

Only the two bolded capabilities are explicit in the brief; the rest are drafts to be confirmed.

# Invariants

- Every manager is a member; every EC office bearer is a manager. Management/EC roles require **`active`** status — suspension or cessation vacates the role ([Domain Model](/specifications/domain-model.md) member-status lifecycle).
- Every PC member is a member of the society **holding an ownership in that project**; PC seats require **`active`** status. Losing active status or transferring away the member's last ownership in the project vacates the seat. One PC per project; a member may sit on several PCs.
- Employees are never members (per the brief). If reality requires a member-employee, that is a requirements change — flag it.
- "Designated" verifier (employee) and "designated subset" of EC for approvals must be configurable per society, not hardcoded roles.

# Decisions

- **Invoice approval quorum** (decided 2026-07-20): approval requires a **majority of the designated EC subset**. Approvals accumulate per EC member; the invoice becomes approved once more than half of the designated subset have approved ([Vendor Workflow](/specifications/vendor-workflow.md)).
- **Rejection & override** (decided 2026-07-20): a **single rejection** by a designated approver is **terminal** — the invoice goes to `rejected` immediately. The sole recourse is an **EC override**: a majority of the **entire EC** (all office bearers, not just the designated subset) approving the rejected invoice supersedes the rejection and approves it.

- **EC appointment & tenure** (decided 2026-07-20): elections and terms are conducted **offline**; the system records role assignments with **effective from/to dates** (tenure history for the audit trail). Assignment/vacation is an admin action — no election module.
- **Employee logins** (decided 2026-07-20): all employees are records; a **designated subset get admin-panel accounts** scoped to their capabilities (invoice verifier, data entry, finance recorder). Required because invoice verification is an employee action — a capability must attach to a real login.
- **Defaulters** (decided 2026-07-20): no automatic rights restriction in v1 — defaulter standing is surfaced in reports for management action, and **payment access is never blocked** (a defaulter must always be able to pay). Statutory restrictions (e.g. voting) are handled offline.

- **PC composition** (decided 2026-07-20): **flat membership with a designated PC Chair** — no further office bearers. Every PC member, chair included, must hold an ownership in the project (invariant above).
- **PC appointment** (decided 2026-07-20): the **EC appoints and dissolves** each PC, drawing from any `active` members meeting the ownership requirement. Composition changes are admin actions recorded with **effective from/to dates** (tenure history), exactly like EC roles — no election module.
- **PC authority scope** (decided 2026-07-20): project-scoped **administration only** — the project record, ownership data for the project, and project documents incl. `member_visible` flags. **Money stays society-level:** work-order issue, invoice verification/approval, and the books remain with employees, the EC, and the Treasurer ([Vendor Workflow](/specifications/vendor-workflow.md), [Finance & Compliance](/specifications/finance-and-compliance.md)). Giving PCs work-order or invoice authority would be a deliberate vendor-workflow change, not an interpretation of this decision.
- **PC system access** (decided 2026-07-20, amended same day ×2): **no dedicated admin-panel capability in v1** — hands-on administration happens through the existing admin roles (a PC member needing direct access gets a Management or data-entry account as today); a project-scoped admin capability is a fast-follow if practice demands it. PC members do get a **mobile surface**: the [Mobile Public API](/api/mobile/public-api.md) `/pc` surface exposes their project's record, roster, and all project documents (`member_visible` does not gate PC reads). Second amendment (2026-07-20): the surface is read-only **except one write** — posting to the PC's own project bulletin board ([Communication](/specifications/communication.md)).
