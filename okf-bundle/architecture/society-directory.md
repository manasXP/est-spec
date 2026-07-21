---
type: Architecture
title: Society Directory
description: The one multi-society component — a tiny, read-only, PII-free lookup service that binds a member's app to their society's deployment via invite links.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [architecture, onboarding, directory, mobile]
---

# Purpose

With **single society per deployment** ([Product Overview](/specifications/product-overview.md)), one app-store binary must find the right society's backend. Decided 2026-07-20: **invite-link onboarding backed by this directory** — members never see a hostname ([Platforms](/specifications/platforms.md)).

# Resolution flow

1. Management creates/activates a member in their society's deployment ([Admin Panel API](/api/admin/public-api.md)).
2. That deployment registers an invite token with the directory and sends the member an SMS/email **universal link**: `https://join.estatly.in/<token>` (survives app install on iOS and Android).
3. On tap, the app resolves the token against the directory:

```
GET https://directory.estatly.in/v1/resolve/<token-or-code>
→ { society_name, api_base_url, cognito: { user_pool_id, client_id, region } }
```

4. The app caches the binding and proceeds into Cognito sign-up/OTP against that society's pool. All subsequent traffic goes directly to `api_base_url` — the directory is not on the request path.

**Fallbacks** resolving through the same endpoint: a **QR code** (posted at the society office) and a **short society code** (e.g. `GRNVLA`) typed once into the app.

**Multi-society members** hold multiple cached bindings; the app offers a society switcher.

# Design constraints

- **PII-free.** The directory maps tokens/codes → endpoints. It never stores member names, phones, or emails — no central member registry, preserving the isolation single-society deployments buy (and avoiding a DPDP-scoped central database).
- **Read-only and boring.** Public surface is `resolve` only. Writes (register society, mint invite tokens) are an authenticated vendor/provisioning concern.
- **Vendor-owned, not per-society.** One instance run by Estatly; registering a society in it is a step of the provisioning runbook ([AWS Blocks backend](/architecture/aws-blocks-backend.md)). Plausibly itself a tiny AWS Blocks app (`KVStore` + `ApiNamespace`-fronted HTTP route).
- Invite tokens are single-use and expiring; society codes are long-lived and public (they only reveal that a society exists, and Cognito still gates entry).

# Decisions (2026-07-20)

- **Token lifetime:** invite tokens are single-use with a **7-day expiry**. Re-inviting from the admin panel mints a fresh token and invalidates the outstanding one.
- **Admin panel:** does **not** use the directory — admins use fixed per-society URLs (a bookmark). The directory serves mobile onboarding only.
