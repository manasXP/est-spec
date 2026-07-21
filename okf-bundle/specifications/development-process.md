---
type: Specification
title: Development Process
description: TDD mandate and the per-layer test toolchains every Estatly codebase follows.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [process, tdd, testing]
---

# Mandate (decided 2026-07-20)

Beyond the brief — a standing directive from the owner: **all Estatly code is developed test-first** (red → green → refactor). Every feature starts from a failing test; no production code without a test that demands it. Test-case documentation lives in [[EST-TCC/README|EST-TCC]].

# Per-layer toolchains (decided 2026-07-20)

| Layer | Toolchain |
|---|---|
| Backend ([AWS Blocks](/architecture/aws-blocks-backend.md)) | **Vitest** unit tests against Blocks' local implementations (`npm run dev`, no AWS account); **contract tests** validating handlers against the OpenAPI specs in [`/api`](/api/index.md); ledger invariants (double-entry balance, GST rounding — [Finance & Compliance](/specifications/finance-and-compliance.md)) as property-style unit tests |
| KMP shared module ([Platforms](/specifications/platforms.md)) | **`kotlin.test`** in `commonTest` for business logic; **SQLDelight in-memory driver** for data-layer tests; **Ktor `MockEngine`** for API-client tests |
| iOS shell | **XCTest** — the SwiftUI shell stays thin; logic under test lives in the KMP module |
| Android shell | **Compose UI tests / JUnit** — same thin-shell principle |
| Admin panel ([Platforms](/specifications/platforms.md)) | **Vitest + React Testing Library** for components and logic; **Playwright** end-to-end for the core flows (work-order → invoice → approval, ledger entry, Tally export) |
| Landing site | Static output — covered by the admin panel's web toolchain where logic exists; no dedicated harness |

# Rationale

- The finance/ledger domain (double-entry books, GST, Tally export) demands verifiable correctness — regressions here are compliance failures, not mere bugs.
- Contract tests keep the contract-first REST decision ([Platforms](/specifications/platforms.md)) honest: the OpenAPI documents in [`/api`](/api/index.md) are executable checks, not just documentation.
- Blocks' local-first runtime and SQLDelight's in-memory driver make the test loop fast enough for red-green-refactor on every layer.
