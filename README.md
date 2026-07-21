# EST-Spec — Specifications

Specifications for Estatly, a single-society housing-society management SaaS.

## Contents

- `okf-bundle/` — the canonical spec knowledge bundle in Open Knowledge Format. Start at [[EST-Spec/okf-bundle/index|the bundle index]]; authoring rules are in [[_conventions]].
  - `specifications/` — narrative feature and domain specs
  - `architecture/` — backend (AWS Blocks) and platform architecture
  - `api/` — API contracts, one folder per client surface (admin, mobile)
  - `references/` — pointers to external knowledge
- `diagrams/` — ERDs of the data model: a 5-tab LucidChart set in `diagrams/lucidchart/` (SVG/PNG exports; view-only live-document link in its `ERD.md`)

## How to review

This repo is open for spec review. Suggested path through the bundle:

1. **Start:** `okf-bundle/index.md` — the bundle map.
2. **Scope:** `okf-bundle/specifications/product-overview.md` — what Estatly is (and isn't).
3. **Domain:** `okf-bundle/specifications/domain-model.md` — society, members, ownerships, finance.
4. **Data model:** the five ERD tabs in `diagrams/lucidchart/svg/` (start with `1-governance-membership.svg`); live LucidChart document linked from `diagrams/lucidchart/ERD.md`.
5. Then follow whatever pulls you in: `architecture/` (AWS Blocks backend, single stack per society) or `api/` (admin and mobile REST contracts).

Feedback we're looking for:

- **Gaps** — scenarios the specs don't cover (Indian housing-society edge cases especially welcome).
- **Contradictions** — places where two documents disagree.
- **Unclear decisions** — anything where you can't tell *why* a choice was made.
- **Compliance** — GST, Tally export, and co-operative-society rules that look wrong or incomplete.

Please file feedback as [GitHub Issues](https://github.com/manasXP/est-spec/issues), one issue per concern, naming the file (and section) it's about. Notes on conventions: files are plain markdown with YAML frontmatter (Open Knowledge Format); `[[wiki links]]` between documents don't render as links on GitHub — treat them as file references.

## Notes

- The original requirements brief (`__init.md`) is retired; the bundle is the source of truth for scope, starting from `specifications/product-overview.md`.
- New spec concepts go into the OKF bundle, one concept per file, following `_conventions.md`.
