---
type: Specification
title: Document Management
description: Society-, project-, and member-level document registry with metadata search — generalizing the brief's ledger-linked scanned-document storage.
status: Draft
version: 0.4.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [documents, storage, search]
---

# Requirement

**Added 2026-07-20 (not in the original brief):** the society needs to manage documents at the **Society**, **Project**, and **Member** levels, with **search capability**. The brief only mandates ledger-linked scanned documents ([Finance & Compliance](/specifications/finance-and-compliance.md)); this spec generalizes that into a single document registry that both serves.

# Document registry

One **Document** record per uploaded file, stored in the FileBucket (S3) already mapped in the [AWS Blocks backend](/architecture/aws-blocks-backend.md); the record lives in the relational store.

| Field | Notes |
|---|---|
| `document_id` | |
| `level` | `society \| project \| member` |
| `project_id` / `member_id` | Exactly one set for `project`/`member` level; both null for `society` |
| `title` | Human name; searchable |
| `category` | From a per-society configurable list (seeded below); searchable |
| `tags` | Free labels; searchable |
| `notes` | Optional free text; searchable |
| `file_name`, `mime_type`, `size`, `checksum` | File metadata; original filename searchable |
| `uploaded_by`, `uploaded_at` | Audit |
| `status` | `active \| archived` |

The **file is immutable** once uploaded — a correction is a new document plus archival of the old one, mirroring the reversal-only ledger convention. **Metadata** (`title`, `category`, `tags`, `notes`) stays editable by management, with changes audit-trailed. No delete: `archived` hides a document from default listings but preserves it (compliance evidence).

## Seed categories

| Level | Typical documents |
|---|---|
| Society | Bye-laws, registration certificate, AGM/EC meeting minutes, audit reports, insurance policies, statutory filings, circulars |
| Project | Sanctioned plans, completion/occupancy certificates, NOCs, vendor contracts (cross-link to [Work Orders](/specifications/vendor-workflow.md)) |
| Member | Share certificate, sale/transfer deeds, KYC, nomination forms, correspondence |

# Relationship to ledger-linked documents

Ledger attachments are **the same Document records** with an additional link: a ledger entry links to one or more documents, and that link is immutable once made ([Finance & Compliance](/specifications/finance-and-compliance.md)). A ledger-linked document can never be archived. One store, one registry, two entry points — no parallel "finance documents" silo.

# Search

- **v1 — metadata search:** full-text over `title`, `category`, `tags`, `notes`, and `file_name`, plus filters on `level`, linked entity, category, and upload date range. Implemented with **Postgres full-text search (`tsvector`)** in the existing Aurora Database Block — the per-society corpus is small (thousands of documents), so no search infrastructure (OpenSearch etc.) is warranted.
- **Content search (OCR)** over the scanned images/PDFs themselves is **deferred past v1** (decided 2026-07-20) — most documents are scans, so it requires OCR (e.g. Textract via `AsyncJob` + CDK). Revisit if metadata search proves insufficient in use; files stay in S3, so OCR can be backfilled over the existing corpus without migration.

# Access (decided 2026-07-20)

- **Upload & manage:** Management and designated data-entry employees, via the admin panel ([Platforms](/specifications/platforms.md)) — consistent with all other data entry. Members never upload.
- **Read (admin):** admin-panel roles see everything.
- **Read (PC, decided 2026-07-20):** members on a project's [PC](/specifications/governance-and-roles.md) read **all** of that project's documents — `member_visible` does not gate PC reads — via the read-only `/pc` surface of the [Mobile Public API](/api/mobile/public-api.md).
- **Member visibility:** a per-document **`member_visible` flag**, default **off**, set by management. When on — society-level docs are readable by all `active` members; project-level docs by members holding an ownership in that project; member-level docs by that member. This is the only visibility mechanism; no per-role ACLs.
- **Member surface:** not in v1 — document management ships admin-panel-only for members at large. A mobile **fast-follow** adds read-only access to member-visible documents ([Platforms](/specifications/platforms.md)). The flag ships in the admin contract now so the data is ready when that surface lands. Exception already in v1: the PC read surface above.

# API impact

**Revised 2026-07-20** in the [Admin Panel API](/api/admin/public-api.md) v0.2.0: `POST /v1/documents` generalized to registry uploads with level/entity metadata; added `GET /v1/documents` (full-text search + filters), metadata `GET`/`PATCH`, presigned `/download`, `/archive` + `/restore` (409 when ledger- or invoice-linked), and `GET`/`PUT /v1/document-categories`.

# Decisions (2026-07-20)

- **Category list is management-editable**, seeded with the categories above — modeled as `PUT /v1/document-categories` in the admin contract; removing a category still in use is rejected.
- **Mobile read access is a fast-follow, not v1** — v1 document management is admin-panel-only ([Platforms](/specifications/platforms.md)).
- **Member visibility is a per-document `member_visible` flag** (default off, management-set); per-level read scope as under Access. Resolves project-level visibility: owners in the project see a project document only when it is flagged — sensitive documents (e.g. vendor contracts) stay management-only by default.
- **OCR content search deferred past v1** — Postgres FTS metadata search is the v1 capability; backfillable later from the S3 corpus.
