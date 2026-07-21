---
type: Reference
title: OKF Authoring Conventions for Estatly
description: Frontmatter vocabulary, directory layout, and linking rules every concept doc in this bundle follows.
tags: [okf, conventions, meta]
timestamp: 2026-07-20T00:00:00Z
---

# What this bundle is

`EST-Spec/okf-bundle/` is the canonical home for Estatly's authored spec knowledge, written in [OKF (Open Knowledge Format) v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) — plain markdown concept docs (YAML frontmatter + structural body) connected by ordinary markdown links. One concept = one file. The original requirements brief (`__init.md`, retired 2026-07-20) is fully absorbed into this bundle; [Product Overview](/specifications/product-overview.md) is now the source of truth for scope. Every spec doc here must trace back to it or carry an explicit **Open Questions** section for what the specs don't yet say.

The layout and frontmatter vocabulary follow the precedent set by `System2.dev/S2E-Spec/okf-bundle/` (see [[_conventions]] there), simplified for Estatly's current stage.

# Directory layout

```
okf-bundle/
├── index.md              # bundle root listing (okf_version: "0.1", no other frontmatter)
├── log.md                # change history, newest first
├── _conventions.md       # this file
├── specifications/       # narrative feature / domain specs
├── architecture/         # backend & platform architecture concepts
├── api/                  # one api/<surface>/ per client surface: REST narrative + openapi.yaml
└── references/           # pointers to external knowledge (skills, sibling workspaces)
```

Diagrams belong in `EST-Spec/diagrams/` (outside the bundle) and are referenced in place.

# Frontmatter

`type` is the only field OKF requires. This bundle also uses:

```yaml
---
type: Specification            # REQUIRED — see vocabulary below
title: Domain Model            # human display name
description: One sentence.     # feeds index files
status: Draft                  # Draft | Reviewed | Accepted | Frozen
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [domain, entities]
---
```

## `type` vocabulary (this bundle)

| `type`          | Used for                                                  |
|-----------------|-----------------------------------------------------------|
| `Specification` | Narrative feature / domain specs (`specifications/`).     |
| `Architecture`  | Backend & platform architecture concepts (`architecture/`). |
| `API Contract`  | REST narratives describing an `openapi.yaml` beside them (`api/<surface>/`). |
| `Reference`     | Pointers to external knowledge and meta docs (`references/`, this file). |

Consumers MUST tolerate unknown `type` values — extend as needed. As the spec matures, adopt further types from the System2.dev vocabulary (`Schema`, `Table`, `Sequence`, `Standard`).

# Linking rules

- **In-bundle** → absolute bundle-relative links from the bundle root: `[Domain Model](/specifications/domain-model.md)`.
- **Out-of-bundle** (EST-Design, sibling workspaces) → Obsidian `[[wikilinks]]`; OKF tolerates these as soft links.
- Broken links are valid — they mark not-yet-written knowledge.

# Conformance check

```bash
find . -name '*.md' ! -name 'index.md' ! -name 'log.md' \
  -exec sh -c 'head -1 "$1" | grep -q "^---$" || echo "NO FRONTMATTER: $1"' _ {} \;
```

A doc is conformant if it opens with `---`, has a closing `---`, parses as YAML, and has a non-empty `type`.
