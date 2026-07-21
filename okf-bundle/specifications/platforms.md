---
type: Specification
title: Platforms
description: The client surfaces — admin panel, landing site, iOS and Android mobile apps — and who uses which.
status: Draft
version: 0.11.0
owner: Manas Pradhan
timestamp: 2026-07-21T00:00:00Z
tags: [platforms, mobile, web]
---

# Surfaces (from the brief)

| Surface | Audience | Draft scope |
|---|---|---|
| **Admin panel** | Management, EC, designated employees | Members & ownership administration, **asset registry** — EC all-projects view, employee project-scoped view ([Asset Management](/specifications/asset-management.md), draft), work orders and invoice verification/approval, books of account, document management, member-request triage ([Member Requests](/specifications/member-requests.md)), **communication** — society bulletin posts and direct member messaging via email/WhatsApp ([Communication](/specifications/communication.md), draft), **meetings & resolutions** — convening and managing meetings, motions, and elections, EC voting and signing, the resolutions register ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md), draft), Tally export |
| **Landing site** | Public | Product marketing; likely the society-onboarding entry point |
| **Mobile app — iOS** | Members; EC and PC members additionally | Own ownerships & charges with **asset detail** ([Asset Management](/specifications/asset-management.md), draft), payment via gateway, receipts; **member requests** — raise & track tickets ([Member Requests](/specifications/member-requests.md)); **bulletin boards** — society + owned-project announcements ([Communication](/specifications/communication.md), draft); **meetings & voting** — GBM and project-GBM ballots, EC/PC meeting votes for seat holders, election ballots and self-nomination, signatory signing on the roll-scoped `/me/meetings` surface ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md), draft); **EC approval inbox** for designated approvers; **PC surface** for committee members (project reads incl. assets + project-board posting) |
| **Mobile app — Android** | Members; EC and PC members additionally | Same scope as iOS |

Backend serving all surfaces: [AWS Blocks backend](/architecture/aws-blocks-backend.md).

# Mobile technology (decided 2026-07-20)

| Concern | Choice |
|---|---|
| iOS UI | Native **SwiftUI** |
| Android UI | Native **Jetpack Compose** |
| Shared business logic | **Kotlin Multiplatform (KMP)** module consumed by both apps |
| Persistence | **SQLDelight** in the KMP module — one schema and one data layer shared by both platforms (native SQLite driver per platform) |
| HTTP client & serialization | **Ktor Client + kotlinx.serialization** in the KMP module (decided 2026-07-20) — one API client for both platforms against the [Mobile Public API](/api/mobile/public-api.md) |

Implication: business logic, persistence, and the backend API client all live in the shared KMP module; the platform apps are UI shells (SwiftUI / Compose) over it. The API client requires a REST/HTTP contract from the [AWS Blocks backend](/architecture/aws-blocks-backend.md).

# Draft positioning

- The **admin panel** is the workhorse — every workflow in [Vendor Workflow](/specifications/vendor-workflow.md) and [Finance & Compliance](/specifications/finance-and-compliance.md) lands here.
- The **mobile app** is deliberately narrow in v1: a member's view of their own standing plus [Payments](/specifications/payments.md). Communication — bulletin boards and direct member messaging — was added beyond the brief on 2026-07-20 ([Communication](/specifications/communication.md)); member-to-member community features remain out of scope.
- The **landing site** is static/marketing-first; it should not share the backend's attack surface.

# Decisions

- **Vendors get no surface** (decided 2026-07-20): vendor data — work orders, invoices, documents — is entered by employees via the admin panel. Vendors are records, not users.
- **EC approval inbox on mobile** (decided 2026-07-20): designated EC approvers can review and approve/reject vendor invoices from the mobile app — see the [Mobile Public API](/api/mobile/public-api.md) `/ec` surface. Same workflow semantics as the admin panel; approval requires a majority of the designated EC subset ([Governance & Roles](/specifications/governance-and-roles.md)) identically on both.

- **Society discovery on mobile** (decided 2026-07-20): **invite-link onboarding backed by a tiny central directory** ([Society Directory](/architecture/society-directory.md)). Management creates the member; the system sends an SMS/email universal link (`https://join.estatly.in/<token>`); the app resolves the token to the society's API host + Cognito config and caches the binding. Members never see a hostname. Fallbacks: QR code and a short society code, resolving through the same directory. Multi-society members hold multiple cached bindings with a society switcher.

- **Member requests on mobile & admin** (decided 2026-07-20): members raise, track, withdraw, and reopen tickets from the mobile app — the [Mobile Public API](/api/mobile/public-api.md) `/me/tickets` endpoints; triage, on-behalf entry, assignment, and resolution on the admin panel ([Admin Panel API](/api/admin/public-api.md)). Staff replies and resolution push notifications to the member's registered devices ([Member Requests](/specifications/member-requests.md)).

- **Member document access** (decided 2026-07-20): not in v1 mobile scope — [Document Management](/specifications/document-management.md) ships admin-panel-only for members at large. **Fast-follow:** read-only mobile access to member-visible documents (own member-level, society-level, and owned-project documents).

- **PC read surface on mobile** (decided 2026-07-20, amended same day): PC members read their project's record, committee roster, and **all** project documents from the mobile app — the [Mobile Public API](/api/mobile/public-api.md) `/pc` surface. Read-only **except one write**: posting to the PC's own project bulletin board ([Communication](/specifications/communication.md)); PC administration stays on the admin panel via existing roles ([Governance & Roles](/specifications/governance-and-roles.md)).

- **Communication composing surfaces** (decided 2026-07-20): composing is **admin-panel-only** in v1 — EC society posts and Management/EC direct messages ([Communication](/specifications/communication.md)). The sole mobile write is PC project-board posting (above); EC mobile composing is a fast-follow. Members read boards on mobile with push on new posts.

- **Admin panel stack** (decided 2026-07-20): **Next.js + TypeScript**, served via the AWS Blocks `Hosting` CDK block (CloudFront + S3). It consumes the REST [Admin Panel API](/api/admin/public-api.md) — one API style across all clients, contract-first — rather than Blocks' TS-native `ApiNamespace` RPC.

- **Landing site stack** (decided 2026-07-20): **static export from the same Next.js + TypeScript toolchain** as the admin panel, served via the `Hosting` CDK block (CloudFront + S3). One web stack, but no backend consumption — the static output keeps the marketing site off the backend's attack surface.

- **Meetings & voting surfaces** (decided 2026-07-21): all member voting happens on mobile via the roll-scoped [Mobile Public API](/api/mobile/public-api.md) **`/me/meetings`** surface — the member sees exactly the meetings whose roll they are on (GBMs, project GBMs of owned projects, EC/PC meetings for seat holders); **no new `/ec` or `/pc` writes**, preserving the read-only-except-one-write `/pc` decision. EC members additionally vote and sign on the admin panel. Convening is **admin-panel-only in v1** ([Meetings, Voting & Resolutions](/specifications/meetings-and-voting.md)); PC-chair mobile convening is a flagged fast-follow.

- **All surfaces are developed test-first** — per-layer TDD toolchains in [Development Process](/specifications/development-process.md) (decided 2026-07-20).
