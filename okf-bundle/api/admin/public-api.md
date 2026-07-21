---
type: API Contract
title: Admin Panel API
description: REST contract for the admin panel — member & ownership administration, the asset registry, charges, the vendor work-order/invoice workflow, books of account, documents, member-request triage, communication, meetings & voting, and Tally export.
status: Draft
version: 0.8.0
owner: Manas Pradhan
timestamp: 2026-07-21T00:00:00Z
tags: [api, rest, admin, management]
resource: ./openapi.yaml
---

# Scope

The REST surface for the **admin panel** web app, used by Management, the EC, and designated employees ([Platforms](/specifications/platforms.md)). It administers everything the [mobile API](/api/mobile/public-api.md) only reads: members, projects, ownerships, the asset registry, charges, vendors, work orders, invoices, employees, the four books, documents, communication (bulletin posts and direct messages), meetings & voting (meetings, motions, elections, the resolutions register), and exports.

Machine-readable contract: [OpenAPI 3.1](openapi.yaml) (draft, authoritative once implementation starts).

# Conventions

Shared with the [Mobile Public API](/api/mobile/public-api.md): `/v1` path versioning, Cognito bearer JWTs, money as decimal INR strings, ISO 8601 UTC timestamps, cursor pagination (`{ items, next_cursor }`), and the `{ "error": { "code", "message" } }` error shape. Admin-specific:

| Concern | Convention |
|---|---|
| Authorization | Role claims in the JWT map to [Governance & Roles](/specifications/governance-and-roles.md) capabilities. Verification requires the **designated-verifier** capability; approval the **designated-approver** (EC) capability; money-posting endpoints (offline/invoice/salary payments, reversals) the **finance-recorder** capability (Treasurer by default, delegable). `403` with code `capability_required` otherwise. |
| Workflow actions | State transitions are explicit `POST` sub-resources (`/invoices/{id}/verify`, `/approve`, `/reject`) — never a `PATCH` of a status field. Each records the acting user for the audit trail. |
| Ledger immutability | Book entries are append-only. There is no update/delete; corrections are `POST …/reversal` ([Finance & Compliance](/specifications/finance-and-compliance.md)). |
| Idempotency | `Idempotency-Key` header required on money-posting endpoints: offline payments, invoice payments, salary payments, reversals. |

# Endpoints

## Members, projects, ownerships

| Method & path | Purpose |
|---|---|
| `GET`/`POST /v1/members` · `GET`/`PATCH /v1/members/{memberId}` | Member administration ([Domain Model](/specifications/domain-model.md) attributes) |
| `GET`/`POST /v1/projects` · `GET`/`PATCH /v1/projects/{projectId}` | Project administration |
| `GET`/`PUT /v1/projects/{projectId}/committee` | The project's **PC** — flat membership + designated chair; PUT is the EC appointment action, tenure history recorded like EC roles; 422 if a member is not active or owns nothing in the project ([Governance & Roles](/specifications/governance-and-roles.md)) |
| `GET`/`POST /v1/members/{memberId}/ownerships` | A member's ownerships; create assigns an existing **asset** by `asset_id` — 409 if already allotted ([Asset Management](/specifications/asset-management.md)) |
| `PATCH /v1/ownerships/{ownershipId}` | Amend informational fields (co-owner names) — never the owner or the asset; the label lives on the [Asset](/specifications/asset-management.md) |
| `POST /v1/ownerships/{ownershipId}/transfer` | Transfer to another member — closes this record, creates a new one, re-points the asset's `current_ownership`; 409 while charges are unsettled ([Domain Model](/specifications/domain-model.md)) |

## Assets ([Asset Management](/specifications/asset-management.md))

The first-class registry — assets exist before/without an ownership. Visibility: Management and EC see all projects; employees see only projects where they hold the **asset-view grant** (`403` code `capability_required` otherwise).

| Method & path | Purpose |
|---|---|
| `GET /v1/assets` | Registry scoped to the caller's visibility — filters `project_id`, `type`, `status`; includes current-owner identity |
| `POST /v1/assets` | Register an asset (Management) — created `available` or `society_retained`; `allotted` is only ever set by ownership assignment |
| `GET /v1/assets/{assetId}` | One asset with its current ownership and the owner history across close-and-create transfers |
| `PATCH /v1/assets/{assetId}` | Edit label/attributes/unowned status — `current_ownership` is never edited, only re-pointed by assignment/transfer; 409 on a status change while allotted |
| `GET`/`PUT /v1/employees/{employeeId}/asset-view-grants` | The employee's project list for asset view — audited Management action mirroring the verifier/finance-recorder pattern ([Governance & Roles](/specifications/governance-and-roles.md)); 422 if the employee has no admin account |

## Charges & member payments

| Method & path | Purpose |
|---|---|
| `GET`/`POST /v1/charges` | List/filter charges across members; raise a one-time asset charge |
| `GET`/`POST /v1/maintenance-schedules` · `PATCH …/{scheduleId}` | Recurring maintenance-fee definitions; the scheduled charge run generates charges from these ([AWS Blocks backend](/architecture/aws-blocks-backend.md), `CronJob`) |
| `POST /v1/charges/{chargeId}/offline-payment` | Record a cash/cheque receipt — posts to Cash/Bank Book + Payment Ledger and issues a receipt |

## Vendors, work orders, invoices ([Vendor Workflow](/specifications/vendor-workflow.md))

| Method & path | Purpose |
|---|---|
| `GET`/`POST /v1/vendors` · `GET`/`PATCH /v1/vendors/{vendorId}` | Vendor registry (incl. GSTIN) |
| `GET`/`POST /v1/work-orders` · `GET`/`PATCH /v1/work-orders/{workOrderId}` | Issue and track work orders |
| `POST /v1/work-orders/{workOrderId}/cancel` | Cancel an unfulfilled work order |
| `POST /v1/work-orders/{workOrderId}/invoices` | Enter a vendor invoice (full or partial) with its scanned document; optionally `resubmission_of` a finally rejected invoice — which closes the EC-override window on it |
| `GET /v1/invoices` · `GET /v1/invoices/{invoiceId}` | Invoice queue, filter by status |
| `POST /v1/invoices/{invoiceId}/verify` | Designated employee verifies |
| `POST /v1/invoices/{invoiceId}/approve` · `/reject` | Designated EC member approves/rejects; approved at a **majority** of the designated subset. One rejection is **terminal**; approvals on a rejected invoice (any EC member) supersede it at a majority of the **entire EC** |
| `POST /v1/invoices/{invoiceId}/payment` | Record the payment to the vendor from Bank/Cash Book |

## Employees

| Method & path | Purpose |
|---|---|
| `GET`/`POST /v1/employees` · `GET`/`PATCH /v1/employees/{employeeId}` | Employee records (non-members, salaried) |
| `POST /v1/employees/{employeeId}/salary-payments` | Record a salary payment — posts to the Expense Ledger |

## Books of account ([Finance & Compliance](/specifications/finance-and-compliance.md))

| Method & path | Purpose |
|---|---|
| `GET /v1/books/{book}/entries` · `GET …/entries/{entryId}` | Read a book: `book ∈ bank \| cash \| payment \| expense` |
| `POST /v1/books/{book}/entries/{entryId}/reversal` | Correction as a reversing entry (entries are never edited) |
| `POST /v1/books/{book}/entries/{entryId}/documents` | Link a registry document to a ledger entry (immutable once linked; blocks archival of the document) |

## Documents ([Document Management](/specifications/document-management.md))

One registry serving society-, project-, and member-level documents and the ledger/invoice attachments above. Files are immutable; metadata is editable and audited; archive, never delete. Upload and management sit under the general admin roles (Management, designated data-entry employees) — no dedicated capability. Each document carries a **`member_visible` flag** (default off) governing the future read-only member surface — a mobile fast-follow, not part of this contract ([Document Management](/specifications/document-management.md)).

| Method & path | Purpose |
|---|---|
| `GET /v1/documents` | Search: full-text `q` over title/category/tags/notes/filename (Postgres FTS) + filters on level, entity, category, upload dates, status |
| `POST /v1/documents` | Register a document (level + entity + metadata) → presigned upload URL (FileBucket); active once the upload completes |
| `GET /v1/documents/{documentId}` | Metadata plus ledger/invoice links |
| `PATCH /v1/documents/{documentId}` | Edit title/category/tags/notes only — file and level/entity binding are immutable |
| `GET /v1/documents/{documentId}/download` | Presigned download URL |
| `POST /v1/documents/{documentId}/archive` · `/restore` | Hide from default listings / bring back; 409 on archive if ledger- or invoice-linked |
| `GET`/`PUT /v1/document-categories` | The society's category list (management-editable); 409 on removing a category in use |

## Member requests ([Member Requests](/specifications/member-requests.md))

Triage and resolution of member tickets (finance / maintenance / records / general). Sits under the general admin roles (Management, designated data-entry employees) — no dedicated capability. Members raise and track tickets on the [Mobile Public API](/api/mobile/public-api.md) `/me/tickets` surface; staff replies and resolution notify the member by push.

| Method & path | Purpose |
|---|---|
| `GET /v1/tickets` | Triage queue — filters `category`, `status`, `assignee_id`, `member_id`; includes ticket age for the ageing view |
| `POST /v1/tickets` | Raise a ticket **on a member's behalf** (walk-in/phone) — the acting user is recorded as `entered_by` |
| `GET /v1/tickets/{ticketId}` | Full detail: comment thread, attachments, work-order link, status history |
| `POST /v1/tickets/{ticketId}/assign` | Assign/reassign to an employee admin account (default comes from category routing) |
| `POST /v1/tickets/{ticketId}/comments` | Staff reply on the thread |
| `POST /v1/tickets/{ticketId}/resolve` | Resolve with a mandatory `resolution_note`; auto-closes after 7 days unless the member reopens |
| `PUT /v1/tickets/{ticketId}/work-order` | Set/clear the informational work-order link on a maintenance ticket ([Vendor Workflow](/specifications/vendor-workflow.md) money paths unchanged) |
| `GET /v1/tickets/{ticketId}/attachments/{attachmentId}` | Presigned download of a member-uploaded attachment (ticket-scoped, not a registry document) |
| `GET`/`PUT /v1/ticket-routing` | Per-category default assignee (management); 422 if an assignee has no admin account |

## Communication ([Communication](/specifications/communication.md))

Bulletin-post composing and direct member messaging. Society posts require an **EC role**; direct messages require **Management or an EC role** — `403` code `capability_required` otherwise. Project-board posts are created by their PC on the [Mobile Public API](/api/mobile/public-api.md) `/pc` surface, never here; this contract only lists and moderates them.

| Method & path | Purpose |
|---|---|
| `GET /v1/bulletin-posts` | All posts across boards — filters `scope`, `project_id`, `include_archived` |
| `POST /v1/bulletin-posts` | Post to the **society board** (EC members); attachments reference registry documents; publishing pushes a notification to all members with app access |
| `GET /v1/bulletin-posts/{postId}` | One post |
| `PATCH /v1/bulletin-posts/{postId}` | Edit a society post (EC members; editor and `edited_at` audited); 409 for project or archived posts |
| `POST /v1/bulletin-posts/{postId}/archive` | Archive any post, society or project (Management/EC moderation) — hidden from member boards, never deleted |
| `GET /v1/messages` | Sent direct messages; filter `member_id` |
| `POST /v1/messages` | Send a direct message to selected member(s) via email and/or WhatsApp — WhatsApp only to members with `whatsapp_opt_in`, others fall back to email-only recorded as a `skipped` delivery entry; delivery is async |
| `GET /v1/messages/{messageId}` | One message with the per-recipient/per-channel delivery log (`queued → sent → delivered / failed`, or `skipped`) |

The `Member` schemas gain **`whatsapp_opt_in`** — normally set by the member at onboarding (mobile `PUT /me/whatsapp-consent`); editable here for offline written consent.

## Meetings, voting & resolutions ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md))

Convening and running meetings, motions, and elections. Convening `gbm`/`ec` meetings requires an **EC role**; `project_gbm`/`pc` meetings Management or an EC role — `403` code `capability_required` otherwise. **Voting and signing are authorized by the frozen roll and the signatory configuration, not capability claims** — an EC member on the roll votes here or on the mobile `/me/meetings` surface interchangeably; the same 403/409 semantics apply on both. All state transitions are explicit `POST` sub-resources per the convention above.

| Method & path | Purpose |
|---|---|
| `GET`/`POST /v1/meetings` | List (filters `kind`, `project_id`, `status`) / convene a meeting (`kind ∈ gbm \| project_gbm \| ec \| pc`; `project_id` required for project kinds) |
| `GET`/`PATCH /v1/meetings/{meetingId}` | Detail with motions, election summary, quorum progress against the frozen roll; PATCH while `draft` only (409 after notice) |
| `POST /v1/meetings/{meetingId}/send-notice` | Publish the notice — pinned bulletin post + push + optional direct message ([Communication](/specifications/communication.md)); 422 if the configured notice period cannot be met before `scheduled_at` |
| `POST /v1/meetings/{meetingId}/start` · `/conclude` · `/cancel` | Lifecycle; `conclude` 409 while any motion is `open` or an election is undeclared; `cancel` before start only, re-notifies a noticed audience |
| `GET`/`POST /v1/meetings/{meetingId}/motions` | Motions on the agenda / add one (title, text, `majority ∈ simple \| special`) |
| `GET`/`PATCH /v1/motions/{motionId}` | Detail with tally and vote list (motion votes are open); PATCH while `draft` only |
| `POST /v1/motions/{motionId}/open-voting` | Freezes text + SHA-256 hash, snapshots the voter roll; 409 unless the meeting is `in_progress` |
| `POST /v1/motions/{motionId}/close-voting` | Computes the outcome — quorum from ballots cast (incl. abstain), majority over yes+no, tie fails; records `outcome_reason` |
| `POST /v1/motions/{motionId}/withdraw` | From `draft` or `open`, before close |
| `POST /v1/motions/{motionId}/votes` | Cast the **caller's own** vote (`yes \| no \| abstain`) — 403 if not on the frozen roll, 409 on duplicate or when not `open` |
| `POST /v1/motions/{motionId}/sign` | Typed e-acknowledgment by a required signatory — records signatory, role, and the frozen `text_hash`; 403 unless a required signatory, 409 unless `passed`; `signed` once all required signatories have signed |
| `GET /v1/resolutions` | The resolutions register — passed/signed motions across meetings; filters `meeting_kind`, `project_id`, `status`, date range |
| `POST /v1/motions/{motionId}/minutes-export` | Export the signed resolution/minutes PDF into the [document registry](/specifications/document-management.md) ("AGM/EC meeting minutes" category; project level for project kinds) |
| `GET`/`POST /v1/meetings/{meetingId}/election` | The meeting's election (one per meeting: `ec` on a `gbm`, `pc` on a `project_gbm`) with its contests |
| `GET /v1/elections/{electionId}` | Detail — contests, accepted nominations, tallies once declared; **ballot choices are never exposed** |
| `POST /v1/elections/{electionId}/open-nominations` · `/close-nominations` · `/open-voting` · `/close-voting` · `/cancel` | Lifecycle; voting opens only while the meeting is `in_progress`; uncontested contests skip voting |
| `POST /v1/elections/{electionId}/declare` | Declares results and **writes the tenure-history role assignments** (effective `term_effective_from`, closes outgoing tenures) in one transaction; vacant contests fall back to manual appointment |
| `GET`/`POST /v1/elections/{electionId}/nominations` | Nominations per contest; POST is Management **on-behalf** entry (walk-ins, `nominated_by` recorded — the ticket `entered_by` pattern); 422 on eligibility failure (not `active`; for `pc`, no ownership in the project) |
| `POST /v1/nominations/{nominationId}/withdraw` | Before nominations close |
| `GET`/`PUT /v1/meeting-config` | Per-society notice periods, quorum fractions, and required signatories per meeting kind (management; the `/ticket-routing` pattern) |

## Exports

| Method & path | Purpose |
|---|---|
| `POST /v1/exports/tally` | Start a Tally export for a period (async job) |
| `GET /v1/exports/{exportId}` | Job status |
| `GET /v1/exports/{exportId}/download` | Presigned URL to the export artifact |

# Resolved questions & notes

- ~~Ownership transfer~~ **resolved 2026-07-20:** `POST /v1/ownerships/{id}/transfer` — settle-before-transfer, close-and-create semantics per the [Domain Model](/specifications/domain-model.md); `PATCH` is informational-only.
- ~~GST reports~~ **resolved 2026-07-20:** deferred past v1 — GSTIN is optional per-society config ([Finance & Compliance](/specifications/finance-and-compliance.md)); `POST /v1/exports/gst` is reserved for when a GST-registered society needs return-ready reports.
- ~~Approval quorum~~ **resolved 2026-07-20:** approval requires a **majority of the designated EC subset**; `/approve` records one member's approval and `approval_progress` reports the count. A **single rejection is terminal** (`rejected`, accumulated approvals discarded); the sole recourse is an **EC override** — `/approve` calls on the rejected invoice accept any EC member and supersede the rejection at a majority of the **entire EC** ([Governance & Roles](/specifications/governance-and-roles.md)).
- ~~Vendor self-service~~ **resolved 2026-07-20:** vendors get no surface — work orders and invoices are entered by employees via the admin panel ([Platforms](/specifications/platforms.md)). This contract is the only entry point for vendor data.
- ~~Documents surface~~ **revised 2026-07-20 (v0.2.0):** `POST /v1/documents` generalized from a finance-scan upload into the [Document Management](/specifications/document-management.md) registry — level/entity metadata on upload, search listing, download, audited metadata edits, archive/restore, and a management-editable category list replacing the old fixed `kind` enum (`DocumentLink` now carries `title`/`category`).
- ~~Member document visibility~~ **decided 2026-07-20 (v0.3.0):** per-document `member_visible` flag (default off) on the Document schemas; the member-facing read surface itself is a mobile fast-follow, deliberately absent from this contract ([Document Management](/specifications/document-management.md)).
- ~~Project Committee~~ **decided 2026-07-20 (v0.4.0):** `GET`/`PUT /v1/projects/{projectId}/committee` records the PC (flat membership + chair, EC-appointed). PCs carry **no financial authority and no dedicated capability in v1** — PC members act through existing admin roles, so no other endpoint changes ([Governance & Roles](/specifications/governance-and-roles.md)).
- ~~Member requests~~ **decided 2026-07-20 (v0.5.0):** ticketing surface added per [Member Requests](/specifications/member-requests.md) — triage queue, on-behalf entry with `entered_by`, assignment over category-based routing (`/v1/ticket-routing`), staff replies, resolve with mandatory note (7-day auto-close), informational work-order linking. Ticket state changes follow the explicit-`POST`-sub-resource convention; tickets are never deleted.
- ~~Communication~~ **decided 2026-07-20 (v0.6.0):** society bulletin posting (EC), post moderation/archive, and direct member messaging via email/WhatsApp with a per-recipient/per-channel delivery log, per [Communication](/specifications/communication.md). WhatsApp goes through the society's pre-approved notice template and only to opted-in members; `whatsapp_opt_in` added to the Member schemas. Project-board posting lives on the mobile `/pc` surface, not here.
- ~~Assets~~ **decided 2026-07-20 (v0.7.0):** first-class asset registry per [Asset Management](/specifications/asset-management.md) — `/v1/assets` with visibility-scoped reads (Management/EC all projects, granted employees theirs), owner history on the detail view, and audited edits. The `Ownership` schemas reference **`asset_id`** (embedded `asset_type`/`asset_label` dropped); ownership create assigns an existing asset and transfer re-points the asset's `current_ownership`. Employee grants via `GET`/`PUT /v1/employees/{employeeId}/asset-view-grants`.
- ~~Meetings & voting~~ **added 2026-07-21 (v0.8.0):** meetings, motions, elections, and the resolutions register per [Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md) — the system is the ballot box. Open-voting freezes the resolution text (SHA-256 hash) and snapshots the roll; votes are final (409 on re-vote); election `declare` writes the tenure-history records the manual `PUT …/committee` path also writes (that endpoint remains for casual vacancies and dissolution); signing is a typed e-acknowledgment against the frozen hash. Amends the 2026-07-20 offline-elections decisions in [Governance & Roles](/specifications/governance-and-roles.md).
- Whether admin auth requires MFA / IP restriction is a deployment decision, not contract-level.
