---
type: Specification
title: Asset Management
description: The asset registry — flats, plots, villas, dividend holdings — and role-based visibility across members, EC, PCs, and employees.
status: Draft
version: 0.3.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [assets, ownership, registry, visibility]
---

# Purpose

**New requirement beyond the original brief** (added 2026-07-20): make assets first-class and visible by role. A member sees the assets mapping to their ownerships; EC members see all assets in all projects; PC members see all assets in their project; employees can be granted project-level asset view; vendors see nothing.

The brief models assets only *inside* [Ownership](/specifications/domain-model.md) (`asset_type` + `asset_label` on the ownership record). This spec promotes **Asset** to its own entity (decided 2026-07-20 — see Decisions).

# Asset entity (draft)

| Field | Notes |
|---|---|
| `asset_id` | Unique within the deployment |
| `project_id` | Every asset belongs to exactly one Project |
| `type` | `flat \| plot \| villa \| dividend` — the canonical enum (confirmed 2026-07-20, see Decisions) |
| `label` | Human identifier, e.g. "A-204", "Plot 17" (absent for dividend-type holdings) |
| `attributes` | Optional physical metadata: area (sq ft), block/floor — draft, extend as needed |
| `status` | `allotted \| available \| society_retained` (draft) — an asset can exist before/without an ownership |
| `current_ownership` | Link to the active Ownership, null if unowned. Ownership transfer (close-and-create, [Domain Model](/specifications/domain-model.md)) re-points this link — the asset persists, giving a per-asset **owner history** for free |
| Timestamps | `created_at`, `updated_at` |

Ownership then references `asset_id` instead of embedding `asset_type`/`asset_label`; charges keep flowing per owned asset exactly as today ([Domain Model](/specifications/domain-model.md)).

# Visibility matrix

| Role | Asset visibility |
|---|---|
| **Member** | Own assets only — those mapped to their ownerships |
| **EC member** | **All assets in all projects** |
| **PC member** | **All assets in their PC project(s)** — consistent with the PC's project-administration authority over ownership data ([Governance & Roles](/specifications/governance-and-roles.md)) |
| **Management** | Administers the registry (create/edit assets) via the existing member/ownership admin workflows — draft |
| **Employee** | **None by default**; a project-scoped **asset-view** grant can be given per employee ([Governance & Roles](/specifications/governance-and-roles.md) "designated is configurable" invariant) |
| **Vendor** | **No access** — vendors are records, not users (decided 2026-07-20, [Platforms](/specifications/platforms.md)) |

EC, PC, and granted-employee views include the current owner's identity — all three roles already see ownership data within their scope. Member views never show other members' assets.

# Surfaces

Per [Platforms](/specifications/platforms.md); contracts updated 2026-07-20 (Ownership schemas reference `asset_id`; asset endpoints on both):

- **Mobile `/me`** — the member's own assets: the ownerships view ([Mobile Public API](/api/mobile/public-api.md) `GET /me/ownerships`) now embeds each ownership's `asset` (type, label, attributes, status).
- **Mobile `/pc`** — PC members list their project's assets via `GET /pc/projects/{projectId}/assets` (read-only, like the rest of the surface; includes current-owner identity).
- **Admin panel** — the `/v1/assets` registry for Management (visibility-scoped reads for EC and granted employees) plus `GET`/`PUT /v1/employees/{employeeId}/asset-view-grants` ([Admin Panel API](/api/admin/public-api.md)).

# Decisions

- **Type vocabulary** (decided 2026-07-20): the asset types are **`flat | plot | villa | dividend`** — the brief's enum stands unchanged. "Apartment" is merely a display synonym for `flat`; "deposits" refers to the existing non-physical `dividend` holding, not a separate finance instrument.
- **First-class Asset registry** (decided 2026-07-20): Asset is its own entity, not a view derived from ownerships — the only way "all assets in a project" can include unsold/society-retained ones, and it yields per-asset owner history across close-and-create transfers. **Ownership gains `asset_id`** and drops the embedded `asset_type`/`asset_label`; both API contracts' Ownership schemas change accordingly (contracts updated 2026-07-20).
- **Employee grant mechanism** (decided 2026-07-20): an **asset-view capability per employee per project**, assigned by Management as an audited admin action — mirroring the designated verifier/finance-recorder pattern ([Governance & Roles](/specifications/governance-and-roles.md)); no new role.
- **EC/Management surface** (decided 2026-07-20): **admin panel only in v1** — EC members are Management and have admin accounts. A mobile `/ec` assets read is a fast-follow.
- **Asset-level documents** (decided 2026-07-20): v1 keeps asset paperwork (allotment letters, sale deeds) as **project-level documents referencing the asset label in metadata** — the [document registry](/specifications/document-management.md) is unchanged. A first-class asset-level binding is a fast-follow if practice demands it.
