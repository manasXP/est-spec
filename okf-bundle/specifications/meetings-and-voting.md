---
type: Specification
title: Meetings, Voting & Resolutions
description: In-app meetings, e-voting on resolutions at society and project level, EC and PC elections, and typed e-signing of resolutions.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-21T00:00:00Z
tags: [meetings, voting, elections, resolutions, governance]
---

# Purpose

**New requirement beyond the original brief** (added 2026-07-21). The society's formal decision-making moves in-app:

1. **Meetings** — General Body Meetings (society and project level) and EC/PC committee meetings are convened, noticed, and concluded in the system.
2. **Voting on motions** — members, EC, and PC members cast votes in-app; the **system is the ballot box**: rolls, tallies, quorum, and outcomes are computed by the system, not recorded after the fact.
3. **Elections** — EC office bearers and PC seats are elected in-app. This **amends** the 2026-07-20 "elections conducted offline, no election module" decisions in [Governance & Roles](/specifications/governance-and-roles.md) (EC appointment & tenure; PC appointment).
4. **Resolution signing** — passed resolutions are signed by their required signatories as a **typed e-acknowledgment** (append-only audit record; no signature images, no DSC — consistent with the receipts decision in [Finance & Compliance](/specifications/finance-and-compliance.md)).

# Meeting

One **Meeting** record per convened sitting. The meeting is the decision window, not a video call — it may stay `in_progress` across a multi-day voting period.

| Kind | Convened by | Voter roll (see below) | Typical use |
|---|---|---|---|
| `gbm` | EC, via the admin panel | All `active` members | AGM/SGM resolutions, EC elections |
| `project_gbm` | Management/EC, via the admin panel | `active` members holding an ownership in the project | Project resolutions, PC elections |
| `ec` | EC, via the admin panel | Current EC office bearers | EC resolutions |
| `pc` | Management/EC, via the admin panel | Current members of the project's PC | PC resolutions |

## Meeting entity (draft)

| Field | Notes |
|---|---|
| `meeting_id` | Unique |
| `kind` | `gbm \| project_gbm \| ec \| pc`; `project_id` required for `project_gbm`/`pc`, null otherwise |
| `title`, `agenda` | Agenda is free text; the votable items are the meeting's Motions and Election |
| `scheduled_at` | Statutory notice period (see Configuration) counts back from this |
| `notice_sent_at` | Set by `send-notice`; delivery rides on [Communication](/specifications/communication.md) |
| `status` | `draft → notice_sent → in_progress → concluded`, `cancelled` — see the state machine in [[ESTx-Glance/WORKFLOW|WORKFLOW]] |
| `convened_by` | The admin-panel actor |

## Lifecycle rules

- `send-notice` requires the configured notice period between now and `scheduled_at`; it publishes the notice (see Notices below) and records `notice_sent_at`.
- `start` moves the meeting to `in_progress`; motions and elections can only open for voting while their meeting is `in_progress`.
- `conclude` is rejected (409) while any motion is `open` or an election is not yet `declared`/`cancelled`.
- `cancel` is allowed before `start`; a cancelled noticed meeting re-notifies its audience.

# Voter rolls & eligibility

- **One member, one vote** (decided — see Decisions): voting weight is per member, never per asset. A member with three flats and a member with one `dividend` holding each cast one vote.
- **Roll snapshot:** when voting opens on a motion (or an election contest), the eligible roll is **frozen as of that moment** — the quorum denominator and the 403 boundary. Status or ownership changes after the snapshot do not add or remove voters.
- Rolls per meeting kind: `gbm` — all `active` members; `project_gbm` — `active` members holding a current ownership in that project (via `asset_id` → Asset → Project, [Domain Model](/specifications/domain-model.md)); `ec` — current office bearers; `pc` — current seat holders of the project's PC.
- `suspended`, `pending`, and `ceased` members are never on a roll. **Defaulters are not excluded** — the 2026-07-20 defaulters decision in [Governance & Roles](/specifications/governance-and-roles.md) stands: statutory restrictions on defaulters remain an offline matter.

# Motions & resolutions

A **Motion** is a votable item on a meeting's agenda; a passed motion is a **Resolution**.

## Motion entity (draft)

| Field | Notes |
|---|---|
| `motion_id`, `meeting_id` | A motion belongs to exactly one meeting |
| `title`, `text` | The resolution text. **Frozen at open-voting** with a SHA-256 `text_hash` — signatures reference this hash |
| `majority` | `simple` (yes > no) \| `special` (yes ≥ 2/3 of yes+no) — per-motion flag; which types statutorily require `special` is an Open Question |
| `status` | `draft → open → passed \| failed`, `withdrawn` — see [[ESTx-Glance/WORKFLOW|WORKFLOW]] |
| `voting_opened_at`, `voting_closed_at` | The voting window |
| Tally | `yes` / `no` / `abstain` counts; `outcome_reason` on `failed` (e.g. `quorum_not_met`, `majority_not_reached`) |

## Voting rules

- `open-voting` (meeting must be `in_progress`) freezes the text + hash and snapshots the roll.
- A vote is `yes | no | abstain`, **final once cast** — append-only, unique per (motion, voter); a second attempt is 409. Not-on-roll voters get 403.
- **Quorum** = ballots cast (including abstain) ≥ the configured fraction of the roll. Abstain counts toward quorum, never toward the majority.
- **Majority** is computed over yes+no only. A tie **fails** (no chair casting vote — see Open Questions). Below-quorum closes as `failed` with `outcome_reason: quorum_not_met`.
- Resolution votes are **open** (identity + choice auditable by Management/EC) — the e-equivalent of a show of hands. Election ballots are secret (below).
- `withdraw` is allowed from `draft` or `open` before close.

# Signing

- Signing is a **typed e-acknowledgment**: an authenticated required signatory invokes Sign; the system records an append-only **ResolutionSignature** — signatory, `role_at_signing`, `signed_at`, and the motion's frozen `text_hash`. No signature images, no DSC (see Open Questions on legal review).
- **Required signatories** are configurable per meeting kind (see Configuration). Defaults: `gbm`/`ec` resolutions — **President + General Secretary**; `project_gbm`/`pc` resolutions — the **PC Chair**.
- A resolution reaches `signed` only when **all** required signatories have signed. Only `passed` motions can be signed (409 otherwise); non-signatories get 403.
- Signed resolutions (and concluded-meeting minutes) are exportable as a PDF registered in the [document registry](/specifications/document-management.md) — society level under the seeded "AGM/EC meeting minutes" category, project level for `project_gbm`/`pc`. The registry holds the exported artifact; the resolution record itself (text, votes, signatures) is first-class data.

# Elections

One **Election** may attach to a meeting: EC elections to a `gbm`, PC elections to a `project_gbm`. An election contains **Contests**:

- **EC election** — one contest per office (President, Vice President, General Secretary, Treasurer, Executive Member × configured seat count).
- **PC election** — one multi-seat contest (`seat_count`; each voter selects up to `seat_count` candidates; top-N by votes are declared).

## Entities (draft)

| Entity | Notes |
|---|---|
| **Election** | `election_id`, `meeting_id`, `kind` (`ec \| pc`), `status`, `nominations_open_at/close_at`, `term_effective_from` |
| **Nomination** | Per contest: `candidate_member_id`, `nominated_by` (self via mobile, or Management on-behalf for walk-ins — the ticket `entered_by` pattern), `status` `submitted \| accepted \| withdrawn \| rejected` (rejected = eligibility failure: not `active`; for PC contests, no ownership in the project) |
| **ElectionVote** | One ballot per voter per contest, immutable once submitted. **Secret:** choices are stored but never exposed by any API — only tallies and the voter's own cast/not-cast flag |

## Lifecycle rules

- `draft → nominations_open → nominations_closed → voting_open → voting_closed → declared`; `cancelled` any time before `declared` — see [[ESTx-Glance/WORKFLOW|WORKFLOW]]. Voting opens only while the meeting is `in_progress`; the roll snapshot follows the same rules as motions.
- **Uncontested contest** (accepted candidates ≤ seats): skips voting — candidates are declared **elected unopposed** at `declare`.
- **Zero-candidate contest**: declared `vacant` — filled through the retained manual appointment path.
- **`declare` writes the tenure history**: declaration creates the same effective-from/to role-assignment records that manual EC/PC assignment writes ([Governance & Roles](/specifications/governance-and-roles.md)), effective `term_effective_from`, and closes outgoing tenures — in one transaction. Manual assignment remains for **casual vacancies** between elections.
- PC Chair selection after a PC election is an Open Question (recommended: EC designates the Chair from the elected members, preserving the existing chair-designation mechanics).

# Notices

Meeting notices ride on [Communication](/specifications/communication.md): `send-notice` publishes a **pinned bulletin post** to the meeting's audience board (society board for `gbm`/`ec`, project board for `project_gbm`/`pc`) with a push notification, and optionally a direct message (email/WhatsApp). The per-recipient **delivery log is the notice audit trail** — "the member was informed". Statutory notice forms remain whatever the law requires; this channel does not replace them.

# Configuration

Per-society meeting configuration (state-parameterized like compliance values — [Finance & Compliance](/specifications/finance-and-compliance.md)); admin contract `GET`/`PUT /v1/meeting-config` (the `/ticket-routing` pattern):

| Setting | Placeholder default | Notes |
|---|---|---|
| Notice period — `gbm`/`project_gbm` | 14 days | Statutory values per pilot-state bye-laws — Open Question |
| Notice period — `ec`/`pc` | 3 days | |
| Quorum — `gbm`/`project_gbm` | 1/3 of the roll | |
| Quorum — `ec`/`pc` | Majority of the committee | |
| Required signatories per meeting kind | President + General Secretary (`gbm`/`ec`); PC Chair (`project_gbm`/`pc`) | |

# Surfaces

Per [Platforms](/specifications/platforms.md):

- **Admin panel** — convene and manage meetings, motions, and elections; EC voting and signing; the resolutions register; meeting configuration. [Admin Panel API](/api/admin/public-api.md) `/v1/meetings`, `/v1/motions`, `/v1/resolutions`, `/v1/elections`, `/v1/meeting-config` (contract updated 2026-07-21).
- **Mobile app** — all member voting happens on a roll-scoped **`/me/meetings`** surface: the member sees exactly the meetings whose roll they are on (GBMs, project GBMs of owned projects, EC/PC meetings for seat holders). Vote on motions, cast election ballots, self-nominate, and sign (signatories are members). **No new `/ec` or `/pc` writes** — the `/pc` "read-only except one write" decision stands. [Mobile Public API](/api/mobile/public-api.md) `/me/meetings`, `/me/motions`, `/me/elections` (contract updated 2026-07-21).

# Decisions (2026-07-21)

- **In-app e-voting — the system is the ballot box**: votes are cast in the system by the voters themselves; rolls, quorum, tallies, and outcomes are computed, not transcribed. (A hybrid/offline capture mode is out of scope v1 — see Open Questions.)
- **Elections in scope**: EC and PC elections run in-app — amending the 2026-07-20 "no election module" decisions in [Governance & Roles](/specifications/governance-and-roles.md). Election declaration writes the same tenure-history records as manual assignment; manual assignment remains for casual vacancies; the EC retains PC dissolution.
- **Signing is a typed e-acknowledgment**: append-only signature records against the frozen resolution text hash. No signature images, no Aadhaar eSign/DSC in v1.
- **Voter rolls — active members, one vote each**: society roll = all `active` members; project roll = `active` members owning in the project; committee rolls = current seat holders. Defaulters are not excluded (statutory restrictions stay offline).
- **Votes are final once cast** (409 on re-vote); rolls are frozen snapshots at open-voting.
- **Resolution votes open, election ballots secret**: motion votes are auditable (show-of-hands equivalent); election choices are never exposed by any API.

# Open Questions

- **Statutory quorum fractions & notice periods** for the pilot state's co-operative act — the Configuration defaults above are placeholders. Resolve with the pilot-society commitment.
- **Election ballot secrecy depth** — recommended: choices stored but never exposed via API; cryptographic unlinkability (choice unlinkable even in the database) is out of scope v1.
- **Tie-breaking** — recommended: no chair casting vote; a tie fails and the motion is re-tabled.
- **Proxy voting** — recommended: out of scope v1; statutory proxy forms stay offline and offline outcomes are minuted via the [document registry](/specifications/document-management.md).
- **PC Chair selection post-election** — recommended: the EC designates the Chair from the elected members.
- **Which resolution types require a special (2/3) majority** — the per-motion `majority` flag ships; the statutory list is state bye-law dependent.
- **PC-chair mobile convening** of `pc`/`project_gbm` meetings — recommended: fast-follow; v1 convening is admin-panel-only.
- **Legal validity of e-votes and typed e-signatures** under the pilot state's co-operative act — flag for legal review; the append-only audit trail + frozen text hash is designed to support whatever evidentiary standard applies.
- **Hybrid meetings** — capturing in-room votes for members not on the app — recommended: out of scope v1; e-ballots only, with physical-meeting outcomes minuted as documents.
