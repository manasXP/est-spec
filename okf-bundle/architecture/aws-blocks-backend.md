---
type: Architecture
title: AWS Blocks Backend
description: Mapping Estatly's backend needs onto AWS Blocks — which Block serves which requirement, and the open architectural questions.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [aws, aws-blocks, backend, architecture]
---

# Context

The brief mandates: "Backend is to be developed in AWS using AWS Blocks." AWS Blocks is an infrastructure-from-code toolkit — each Block bundles cloud resources, a Lambda runtime, and a local implementation in one npm package; infrastructure is derived from the IFC layer (`aws-blocks/index.ts`), with no separate IaC. Working knowledge: the aws-blocks skill evaluation notes (formerly `references/`, retired 2026-07-21) — SDK findings verified against the real packages by STR-001, and written back into this document.

# Block mapping (draft)

| Estatly need | Block | AWS service | Notes |
|---|---|---|---|
| System of record — members, ownerships, work orders, invoices, **books of account** | `Database` | Aurora Serverless v2 | Relational + transactions are non-negotiable for double-entry ledgers; single-society deployment — no tenancy columns or RLS ([Product Overview](/specifications/product-overview.md)) |
| Scanned bills / vouchers / receipts / challans; society/project/member documents | `FileBucket` | S3 | Presigned upload/download; one registry serving ledger links and the document library ([Document Management](/specifications/document-management.md)) |
| Document metadata search | `Database` | Aurora Serverless v2 | Postgres full-text (`tsvector`) over document metadata — no separate search infrastructure ([Document Management](/specifications/document-management.md)) |
| Auth — members, management, employees, vendors | `AuthCognito` | Cognito | Production-grade (MFA); role claims per [Governance & Roles](/specifications/governance-and-roles.md) |
| Recurring maintenance-fee generation, due-date reminders | `CronJob` | EventBridge + Lambda | Monthly/quarterly charge runs |
| Tally export, GST report generation, payment-webhook reconciliation | `AsyncJob` | SQS + Lambda | Long-running / retryable background work ([Payments](/specifications/payments.md)) |
| Bills, receipts, approval notifications by email | `EmailClient` | SES | |
| App config & secrets (gateway keys) | `AppSetting` | SSM Parameter Store | |
| Observability | `Logger` / `Metrics` / `Dashboard` | CloudWatch | |
| Admin panel + landing site hosting | `Hosting` (CDK layer) | CloudFront + S3 | |

Outside the per-society stack sits the vendor-owned [Society Directory](/architecture/society-directory.md) — the one multi-society component, resolving invite links/codes to a deployment's endpoints; registering with it is a provisioning-runbook step.

Blocks evaluated and **not** needed for v1: `Agent` / `KnowledgeBase` (no AI features in the brief), `Realtime` (no live-collaboration requirement), `KVStore` / `DistributedTable` (relational store covers v1; add for caching only if measured need).

# Standing cautions (from the skill)

- **Block IDs are immutable once deployed** — renaming a stateful Block's ID (Database, FileBucket) recreates the resource and loses data. Fix IDs early: e.g. `estatly-db`, `estatly-documents` (`fullId` joins scope and block ID with a hyphen — verified against Blocks v0.2.3, STR-001).
- Keep all Blocks and the API in the IFC layer; drop to the CDK layer only where no Block fits (e.g. `Hosting`, payment-webhook ingress specifics).
- Local-first development: `npm run dev` runs with no AWS account; use `sandbox` only for behavior that differs on real AWS.

# Open architecture questions

- **Mobile clients vs `ApiNamespace`:** Blocks' type-safe RPC assumes a TypeScript frontend importing the API directly. The mobile apps are **decided native** (SwiftUI / Jetpack Compose with a shared KMP module — see [Platforms](/specifications/platforms.md)), so the member-facing surface needs an explicit **REST/HTTP contract** consumed by the KMP API client — drafted as [Mobile Public API](/api/mobile/public-api.md). **Policy decided 2026-07-20:** if Blocks can't serve plain HTTP routes natively, the REST surface (and the Razorpay webhook endpoint) goes through the **CDK escape hatch** (API Gateway + Lambda). Whether native support exists is a build-time verification item, not a blocker — the same applies to the admin panel's REST consumption ([Platforms](/specifications/platforms.md)).
- ~~Single-society vs multi-tenant~~ **resolved 2026-07-20:** one stack per society ([Product Overview](/specifications/product-overview.md)). The `Scope` name should carry the society (e.g. `estatly-<society>`), and provisioning a society is a repeatable deploy of the same IFC code — no tenancy machinery in the schema. Fleet concerns (upgrading N society stacks, per-society cost) become the operational question to plan for in EST-Deploy.
- Region **decided 2026-07-20: `ap-south-1` (Mumbai)** ([Product Overview](/specifications/product-overview.md)). Aurora Serverless v2, Cognito, and SES are available there; verify the full Blocks set at build time.
