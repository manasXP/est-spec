---
type: Specification
title: Communication
description: Bulletin boards at society (EC) and project (PC) level, and direct member messaging via email and WhatsApp.
status: Draft
version: 0.3.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [communication, bulletin, messaging, email, whatsapp]
---

# Purpose

**New requirement beyond the original brief** (added 2026-07-20). The communication surface has two parts:

1. **Bulletin boards** — the EC and each PC post announcements on boards existing at their level, visible to members.
2. **Direct messages** — individual member(s) can be messaged via **email** and **WhatsApp**.

Both are **one-way, society-to-member** channels in v1. A member's response is a ticket ([Member Requests](/specifications/member-requests.md)) — there is no reply, comment, or member-to-member messaging.

# Bulletin boards

| Board | Posted by | Visible to (draft) |
|---|---|---|
| **Society board** (one) | EC members | All members with app access (`active` and `suspended` — [Domain Model](/specifications/domain-model.md) lifecycle) |
| **Project board** (one per project) | That project's PC members | Members holding an ownership in the project, plus its PC members — mirrors project-document visibility in [Document Management](/specifications/document-management.md) |

## BulletinPost entity (draft)

| Field | Notes |
|---|---|
| `post_id` | Unique |
| `scope` | `society \| project`; project posts carry `project_id` |
| `author` | The individual EC/PC member who posted — not just "the EC" |
| `title`, `body` | Announcement content |
| `attachments` | Via the [document registry](/specifications/document-management.md) (e.g. a circular, a notice PDF) |
| `pinned` | Optional; keeps a post at the top of its board |
| `posted_at`, `edited_at` | Edits are audited; posts are **archived, never deleted** (bundle-wide rule) |

## Rules (draft)

- Posting authority follows committee membership: any EC member may post to the society board; any member of a project's PC may post to that project's board. Losing the seat ([Governance & Roles](/specifications/governance-and-roles.md) invariants) removes posting rights; existing posts remain with their author attribution.
- New posts trigger a **push notification** to the board's audience — device registration already exists for payment reminders ([Mobile Public API](/api/mobile/public-api.md)).
- Boards are announcement-only: no member comments or reactions in v1.

# Direct messages

- **Senders:** Management and EC, via the admin panel (decided — see Decisions). PCs cannot direct-message members in v1.
- **Recipients:** one member or an ad-hoc selection of members.
- **Channels:** email and WhatsApp, selected per message (either or both). The member's `email`/`phone` come from the member record.
- **Outbound only:** no in-app inbox in v1. Replies arrive outside the system, or as tickets.
- **Delivery log:** per recipient per channel (`queued → sent → delivered / failed`) — the audit trail for "the member was informed", which matters for notices.
- **Boundary:** ad-hoc messaging only. Automated payment reminders remain as decided in [Payments](/specifications/payments.md); statutory notices follow whatever form the law requires — this channel does not replace them.

## Channel infrastructure (draft)

| Channel | Draft approach |
|---|---|
| **Email** | AWS SES from the [AWS Blocks backend](/architecture/aws-blocks-backend.md) — natural fit, per-society sender identity |
| **WhatsApp** | WhatsApp Business Platform. Business-initiated messages require **pre-approved message templates** and member **opt-in** (Meta policy). Consumed through a **provider-neutral boundary** — the Razorpay pattern from [Payments](/specifications/payments.md); provider decided 2026-07-20: **Meta Cloud API direct** ([[EST-Design/whatsapp-provider|WhatsApp Provider]]) |

# Surfaces

Per [Platforms](/specifications/platforms.md):

- **Admin panel** — EC composes society posts; Management/EC compose direct messages and view the delivery log; post moderation (archive) for all boards. [Admin Panel API](/api/admin/public-api.md) `/bulletin-posts` and `/messages` (contract updated 2026-07-20).
- **Mobile app** — members read the society board and the boards of projects they own in; push on new posts; WhatsApp consent flag. PC members post to their project's board via the `/pc` surface's single write (see Decisions). [Mobile Public API](/api/mobile/public-api.md) `/me/bulletin`, `/me/whatsapp-consent`, and `/pc` posts (contract updated 2026-07-20).

# Decisions (2026-07-20)

- **PC posting surface**: the mobile `/pc` surface gains a **single narrow write** — create/edit posts on the PC's own project board — amending the strictly-read-only `/pc` decision in [Governance & Roles](/specifications/governance-and-roles.md) and [Platforms](/specifications/platforms.md). Everything else on `/pc` stays read-only, and PCs still get no admin-panel capability.
- **Direct-message senders**: Management and EC, via the admin panel. PCs cannot direct-message members in v1.
- **WhatsApp account model**: one **WhatsApp Business number per society**, provisioned alongside directory registration ([Society Directory](/architecture/society-directory.md)) — matching the per-society Razorpay merchant and single-society deployments. Provider decided 2026-07-20: **Meta Cloud API direct** — Estatly operates its own WABA with one number per society; the provider-neutral boundary stays as the escape hatch to a BSP (evaluation in [[EST-Design/whatsapp-provider|WhatsApp Provider]]). Number registration and credentials become provisioning-runbook steps ([[EST-Deploy/provisioning-runbook|Provisioning Runbook]]).
- **WhatsApp opt-in**: a **consent flag on the member record**, captured at onboarding. A message to a non-opted-in member is sent **email-only**, recorded as such in the delivery log. Email needs no opt-in for society notices.
- **EC composing surface**: admin panel only in v1; mobile composing (society posts, direct messages) is a fast-follow.
