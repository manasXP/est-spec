---
type: Specification
title: Member Requests (Ticketing)
description: Member-raised tickets for finance, maintenance, and records queries — entity, lifecycle, routing, and surfaces.
status: Draft
version: 0.2.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [tickets, requests, helpdesk, members]
---

# Purpose

**New requirement beyond the original brief** (added 2026-07-20): a ticketing system through which members raise requests or queries — **finance** (a charge, receipt, or ledger question), **maintenance** (something needs fixing), or **records** (a correction or copy of member/ownership/document data) — and track them to resolution. Today these arrive by phone or walk-in and leave no trail; a ticket gives every request an owner, a status, and an audit history.

# Ticket entity

| Field | Notes |
|---|---|
| `ticket_id` | Unique; also the member-facing reference number |
| `category` | `finance \| maintenance \| records \| general` — fixed enum in v1 (see Decisions); drives routing |
| `subject`, `description` | Free text from the member |
| `raised_by` | The Member; `entered_by` records the admin user when raised on the member's behalf (see Decisions) |
| `status` | See lifecycle below |
| `assignee` | An Employee with an admin login ([Governance & Roles](/specifications/governance-and-roles.md)); defaulted by category routing, reassignable |
| `comments` | Threaded conversation between the member and staff; append-only |
| `attachments` | Ticket-scoped files (a photo of the issue, a disputed receipt) in bulk storage via presigned upload/download — **not** registry documents (see Decisions) |
| `work_order` | Optional link, maintenance tickets only — see Routing |
| `resolution_note` | Required to move to `resolved` |
| Timestamps | `created_at`, `updated_at`, `resolved_at`, `closed_at` |

# Lifecycle

```
open → in_progress → resolved → closed   (auto-close 7 days after resolution)
open | in_progress → withdrawn           (by the raising member, before resolution)
resolved → open                          (member reopens, within the 7-day window)
```

- `resolved` requires a `resolution_note`; the member is notified and can reopen or let it auto-close.
- `closed` and `withdrawn` are terminal — a further request is a new ticket (optionally linking the old one), mirroring the never-revive rule in [Vendor Workflow](/specifications/vendor-workflow.md).
- Tickets are never deleted; the thread and status history are the audit trail.

# Routing

- **Category-based default assignee:** per-society configuration maps each category to a designated employee (`GET`/`PUT /v1/ticket-routing` on the [Admin Panel API](/api/admin/public-api.md)) — consistent with the "designated is configurable, not hardcoded" invariant in [Governance & Roles](/specifications/governance-and-roles.md). Management may reassign any ticket.
- **Maintenance** tickets needing vendor work get an optional link to a **Work Order** ([Vendor Workflow](/specifications/vendor-workflow.md)). The link is informational: the ticket resolves when the member's request is addressed, and the work-order → invoice → approval money path is unchanged by ticketing.
- **Finance** tickets never move money. Answers may reference ledger entries and receipts; an actual correction still goes through the reversal-only mechanics of [Finance & Compliance](/specifications/finance-and-compliance.md).
- **Records** tickets request changes or copies of member, ownership, or document data; the change itself is performed through the existing admin workflows, then the ticket is resolved referencing it.

# Surfaces

Per [Platforms](/specifications/platforms.md):

- **Mobile app** — members raise tickets, attach photos, comment, withdraw, reopen, and track status of their own tickets: the `/me/tickets` endpoints on the [Mobile Public API](/api/mobile/public-api.md) (→ v0.5.0). Staff replies and resolution notify the member via push on their registered devices.
- **Admin panel** — triage queue (filter by category/status/assignee/member), on-behalf entry, assignment, comment replies, resolve, work-order linking, and routing configuration: the `/v1/tickets` and `/v1/ticket-routing` endpoints on the [Admin Panel API](/api/admin/public-api.md) (→ v0.5.0). Ticket handling sits under the general admin roles (Management, designated data-entry employees) — no dedicated capability.

# Decisions (2026-07-20)

- **Category list:** fixed enum `finance | maintenance | records | general` in v1 — routing configuration keys off category. A management-editable list (as document categories have) is a fast-follow.
- **Routing configuration:** one default assignee per category, management-configurable, plus manual reassignment. No rules engine.
- **SLA / escalation:** none in v1. Ticket age is visible in the triage queue and an ageing report; escalation happens offline.
- **On-behalf entry:** staff can raise a ticket for a member who phones or walks in — data-entry/management logins, recorded with `entered_by` (same pattern as offline payments).
- **Auto-close:** a resolved ticket closes automatically 7 days after resolution unless the member reopens it; the reopen window is those same 7 days.
- **Attachments are ticket-scoped, not registry documents:** the [document registry](/specifications/document-management.md) stays a management-curated corpus with mandatory category metadata; member-uploaded ticket photos carry none of that. Staff view attachments from the ticket itself.
