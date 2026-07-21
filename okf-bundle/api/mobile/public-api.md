---
type: API Contract
title: Mobile Public API
description: REST contract for the mobile app — member profile, ownerships with asset detail, charges, payment initiation, receipts, member request tickets, bulletin boards, meetings & voting, the EC approval inbox, and the PC surface.
status: Draft
version: 0.9.0
owner: Manas Pradhan
timestamp: 2026-07-21T00:00:00Z
tags: [api, rest, mobile, members]
resource: ./openapi.yaml
---

# Scope

The REST surface consumed by the shared **KMP API client** in the iOS/Android apps (see [Platforms](/specifications/platforms.md)). Three surfaces in one contract:

- **`/me`** — a member's own standing, payments, request tickets ([Member Requests](/specifications/member-requests.md), decided 2026-07-20), the bulletin feed ([Communication](/specifications/communication.md), decided 2026-07-20), and **meetings & voting** ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md), decided 2026-07-21) — roll-scoped, so the same surface serves GBM ballots, project-GBM ballots, and EC/PC meeting votes for seat holders — available to every authenticated member.
- **`/ec`** — the **EC approval inbox** (decided 2026-07-20): EC members review and approve/reject vendor invoices on the go. Normal approve/reject requires the designated-approver capability; **override approvals on rejected invoices are open to any EC member** (`403` code `capability_required` otherwise). Same workflow semantics as the [Admin Panel API](/api/admin/public-api.md).
- **`/pc`** — the **PC surface** (decided 2026-07-20, amended same day): members sitting on a [Project Committee](/specifications/governance-and-roles.md) read their project's record, committee roster, **all** of that project's [documents](/specifications/document-management.md) — `member_visible` does not gate PC reads — and its full [asset registry](/specifications/asset-management.md). Read-only **except one write** — posting to the PC's own project bulletin board ([Communication](/specifications/communication.md)); other PC administration (uploads, metadata, flags) stays on the admin panel through existing roles.

Society administration otherwise stays on the admin panel.

Machine-readable contract: [OpenAPI 3.1](openapi.yaml) (draft, authoritative once implementation starts).

# Conventions

| Concern | Convention |
|---|---|
| Base path | `/v1` — breaking changes bump the path version |
| Auth | `Authorization: Bearer <JWT>` issued by Cognito ([AWS Blocks backend](/architecture/aws-blocks-backend.md), `AuthCognito`). The app authenticates against Cognito directly; this API only validates tokens. Every endpoint is scoped to the authenticated member — there are no member-ID path parameters to traverse. |
| Base URL discovery | The app learns `api_base_url` + Cognito config by resolving its invite link / society code against the [Society Directory](/architecture/society-directory.md) at onboarding, then caches the binding. This contract has no discovery endpoints. |
| Money | Decimal **string** in INR (e.g. `"1250.00"`) — never floats |
| Timestamps | ISO 8601 UTC |
| Pagination | Cursor-based: `?cursor=&limit=` → `{ items, next_cursor }` (`next_cursor` null on the last page) |
| Errors | `{ "error": { "code": "<machine_code>", "message": "<human text>" } }` with conventional HTTP status |
| Idempotency | `Idempotency-Key` header required on `POST /me/payments` |
| App version gate | The app sends `X-App-Version` on every request. If it is below the deployment's `min_supported_version` configuration (unset by default), the request fails **`426 Upgrade Required`** (code `upgrade_required`) and the KMP client intercepts it into an upgrade screen. Handling ships in the first app build — enforcement can then be turned on per society at any time |

# Endpoints

| Method & path | Purpose |
|---|---|
| `GET /v1/me` | Member profile (`member_id`, `name`, `email`, `phone`, `address`, `joining_date`, `member_status`) |
| `GET /v1/me/ownerships` | The member's assets across projects, each ownership embedding its asset's detail — type, label, attributes, status ([Asset Management](/specifications/asset-management.md)) |
| `GET /v1/me/charges` | Charges (one-time + recurring maintenance); filter `?status=due\|paid\|all` |
| `GET /v1/me/charges/{chargeId}` | One charge with its payment/receipt trail |
| `POST /v1/me/payments` | Initiate payment for one or more due charges → returns a provider-neutral payment intent for the gateway SDK ([Payments](/specifications/payments.md)) |
| `GET /v1/me/payments/{paymentId}` | Payment status (`initiated → pending → succeeded \| failed`) — poll after the gateway SDK returns; the ledger posts only on webhook confirmation |
| `GET /v1/me/receipts` | Receipts issued to the member |
| `GET /v1/me/receipts/{receiptId}/document` | Short-lived presigned URL to the receipt document (FileBucket) |
| `POST /v1/me/devices` | Register this device for push notifications (due-date reminders) |
| `DELETE /v1/me/devices/{deviceId}` | Deregister on logout |
| `PUT /v1/me/whatsapp-consent` | Set the member's WhatsApp messaging consent flag ([Communication](/specifications/communication.md)) — without it, direct messages arrive by email only |

## Member requests ([Member Requests](/specifications/member-requests.md))

| Method & path | Purpose |
|---|---|
| `GET /v1/me/tickets` | The member's own tickets; filter `?status=` |
| `POST /v1/me/tickets` | Raise a ticket — category (`finance \| maintenance \| records \| general`), subject, description |
| `GET /v1/me/tickets/{ticketId}` | One ticket with its comment thread and attachments |
| `POST /v1/me/tickets/{ticketId}/comments` | Reply on the thread (409 once the ticket is closed or withdrawn) |
| `POST /v1/me/tickets/{ticketId}/attachments` | Attach a file (photo, disputed receipt) → presigned upload URL |
| `GET /v1/me/tickets/{ticketId}/attachments/{attachmentId}` | Presigned download URL for an attachment |
| `POST /v1/me/tickets/{ticketId}/withdraw` | Withdraw before resolution — terminal |
| `POST /v1/me/tickets/{ticketId}/reopen` | Reopen a resolved ticket within the 7-day auto-close window |

Staff replies and resolution push a notification to the member's registered devices. Triage, assignment, and resolution happen on the [Admin Panel API](/api/admin/public-api.md).

## Bulletin boards ([Communication](/specifications/communication.md))

| Method & path | Purpose |
|---|---|
| `GET /v1/me/bulletin` | Feed across the society board and the boards of projects the member owns in; filters `scope`, `project_id`; pinned first, then newest |
| `GET /v1/me/bulletin/{postId}` | One post — the deep-link target of the new-post push notification |
| `GET /v1/me/bulletin/{postId}/attachments/{documentId}` | Presigned download of a post attachment (registry document) |

Boards are read-only for members at large — no comments or reactions; composing happens on the [Admin Panel API](/api/admin/public-api.md) (EC, society board) and the `/pc` write below (PC, project board).

## Meetings & voting ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md))

Roll-scoped: a member sees exactly the meetings whose voter roll they are on (or will be on) — GBMs, project GBMs of owned projects, and EC/PC meetings for seat holders. Voting and signing are authorized by the frozen roll and signatory configuration; there are no meeting writes on `/ec` or `/pc`.

| Method & path | Purpose |
|---|---|
| `GET /v1/me/meetings` | The member's meetings; filter `?status=` |
| `GET /v1/me/meetings/{meetingId}` | Agenda, motions with status and tallies (motion votes are open; election tallies only once declared), election summary, and the caller's cast/not-cast state per item |
| `POST /v1/me/motions/{motionId}/votes` | Cast `yes \| no \| abstain` — final once cast; 403 if not on the frozen roll, 409 on duplicate or when voting is not open |
| `POST /v1/me/motions/{motionId}/sign` | Typed e-acknowledgment by a required signatory (President/General Secretary/PC Chair are members); 403 unless a required signatory, 409 unless `passed` |
| `GET /v1/me/elections/{electionId}` | Contests, accepted candidates, and the caller's voted flags — **ballot choices are never returned by any endpoint** |
| `POST /v1/me/elections/{electionId}/nominations` | **Self**-nominate during the nomination window; 422 if ineligible (not `active`; for PC contests, no ownership in the project) |
| `POST /v1/me/nominations/{nominationId}/withdraw` | Withdraw own nomination before nominations close |
| `POST /v1/me/elections/{electionId}/votes` | Cast the ballot — one immutable submission per contest (up to `seat_count` choices in a multi-seat contest); 403 not on roll, 409 duplicate/not open |

## EC approval inbox (designated approvers only)

| Method & path | Purpose |
|---|---|
| `GET /v1/ec/invoices` | Invoices awaiting the caller's approval (verified queue); filter `?status=` |
| `GET /v1/ec/invoices/{invoiceId}` | One invoice with amounts, work-order context, and action history |
| `GET /v1/ec/invoices/{invoiceId}/document` | Presigned URL to the scanned invoice — review before deciding |
| `POST /v1/ec/invoices/{invoiceId}/approve` | Record the caller's approval — approved at a **majority of the designated subset**; on a rejected invoice this is an **EC override** (any EC member; majority of the **entire EC** supersedes the rejection). `approval_progress` reports the count against the active threshold |
| `POST /v1/ec/invoices/{invoiceId}/reject` | Reject with a mandatory reason — **terminal**; discards accumulated approvals (recourse: EC override via approve) |

Approve/reject here and on the [Admin Panel API](/api/admin/public-api.md) hit the same workflow — an invoice approved on mobile is approved everywhere, and both record the acting EC member in the audit trail.

## PC surface (PC members only)

| Method & path | Purpose |
|---|---|
| `GET /v1/pc/projects` | Projects where the caller sits on the PC, with the committee roster and the caller's chair status |
| `GET /v1/pc/projects/{projectId}/documents` | The project's active documents — metadata search `q` + `category` filter, same search semantics as the admin registry ([Document Management](/specifications/document-management.md)); `403` code `capability_required` unless the caller is on that project's PC |
| `GET /v1/pc/documents/{documentId}/download` | Presigned URL to the file; `403` unless the document belongs to a project whose PC the caller sits on |
| `GET /v1/pc/projects/{projectId}/assets` | The project's full asset registry — including unowned (available / society-retained) assets — with current-owner identity ([Asset Management](/specifications/asset-management.md)); filters `type`, `status`; `403` unless the caller is on that project's PC |
| `POST /v1/pc/projects/{projectId}/posts` | Post to the PC's project bulletin board — the surface's **single write** ([Communication](/specifications/communication.md)); attachments reference existing project-level registry documents; publishing pushes to the project's owners |
| `PATCH /v1/pc/posts/{postId}` | Edit a post on the PC's own board (any current PC member; editor and `edited_at` audited); 409 once archived |

Read-only except the project-board posting write ([Governance & Roles](/specifications/governance-and-roles.md) amendment, 2026-07-20): all other PC administration happens on the admin panel via existing roles. Post archival is admin-panel moderation, not a `/pc` action.

# Payment flow (client's view)

1. `POST /v1/me/payments` with `charge_ids` and an `Idempotency-Key` → `201` with `{ payment_id, provider, provider_params }`.
2. Hand `provider_params` to the third-party gateway SDK in-app.
3. Poll `GET /v1/me/payments/{payment_id}` until `succeeded`/`failed`. Success is **only** ever set server-side from the gateway webhook — the client result is advisory ([Payments](/specifications/payments.md)).
4. On success, the receipt appears under `/v1/me/receipts`.

# Decisions (2026-07-20)

- **Sign-in:** Cognito **phone-OTP primary** (phone is a required member attribute), email as recovery/secondary channel.
- **Society/notice endpoints:** excluded from v1 — not in the brief.
- **Push notifications:** in v1 scope for **due-date reminders** ([Payments](/specifications/payments.md) member-initiated collection depends on them), **ticket updates** (staff reply, resolution), **new bulletin posts** ([Communication](/specifications/communication.md)), and **meeting events** — notice sent, voting opened, results declared, each deep-linking to the meeting detail ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md), added 2026-07-21). Device registration endpoints are part of this contract.
- **Member requests** (decided 2026-07-20): `/me/tickets` per [Member Requests](/specifications/member-requests.md) — fixed category enum, withdraw/reopen member actions, ticket-scoped attachments via presigned upload (not registry documents), 7-day reopen window before auto-close.
- **Communication** (decided 2026-07-20): `/me/bulletin` feed (society + owned-project boards, pinned first, archived hidden), `PUT /me/whatsapp-consent` for the opt-in flag, and the `/pc` posting write per [Communication](/specifications/communication.md). Direct messages have **no mobile surface** — they arrive as email/WhatsApp, not in-app.
- **Assets** (decided 2026-07-20): the `Ownership` schema drops the embedded `asset_type`/`asset_label` for an embedded `asset` object per [Asset Management](/specifications/asset-management.md); `/pc` gains the project assets read. EC members get **no mobile asset surface in v1** — the all-projects view is admin-panel-only; a mobile `/ec` assets read is a fast-follow.
- **Forced upgrade** (decided 2026-07-20): the `X-App-Version` / `426 Upgrade Required` gate above is **reserved in the v1 contract** so the mechanism exists in every installed app from day one; *enforcement* (setting `min_supported_version`) is a per-society operational act ([[EST-Deploy/release-and-rollback|Release & Rollback]]). Rationale: apps that predate the mechanism can never be force-upgraded.
- **Meetings & voting** (decided 2026-07-21): all member voting rides on the roll-scoped `/me/meetings` surface per [Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md) — no new `/ec` or `/pc` writes, preserving the read-only-except-one-write `/pc` decision. Votes are final once cast; election ballot choices are stored but never exposed by any endpoint; signing is a typed e-acknowledgment against the frozen resolution hash. Convening has no mobile surface in v1 (admin panel only).
