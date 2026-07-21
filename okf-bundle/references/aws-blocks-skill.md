---
type: Reference
title: AWS Blocks Skill (from System2.dev)
description: The one System2.dev skill relevant to Estatly — working knowledge for building the mandated AWS Blocks backend — plus the evaluation of the skills not picked.
status: Draft
version: 0.1.0
owner: Manas Pradhan
timestamp: 2026-07-20T00:00:00Z
tags: [skills, aws-blocks, system2]
resource: ../../../../System2.dev/.claude/skills/aws-blocks/SKILL.md
---

# What it is

`System2.dev/.claude/skills/aws-blocks/` — a Claude Code skill covering **AWS Blocks**, the infrastructure-from-code toolkit the Estatly brief mandates for the backend. It documents the mental model (Scope, IFC layer, ApiNamespace, BlocksContext, conditional exports, CDK escape hatch), the Block catalog with AWS service mappings, local dev / sandbox / deploy commands, and the data-loss caution around immutable Block IDs. A deeper `reference.md` with per-Block APIs sits beside it.

- Docs: https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html
- Source: https://github.com/aws-devtools-labs/aws-blocks

# Why it's picked for Estatly

The original requirements brief states: *"Backend is to be developed in AWS using AWS Blocks."* This skill is the working knowledge for that mandate; the Block mapping in [AWS Blocks Backend](/architecture/aws-blocks-backend.md) is derived from it.

# How to use it here

When backend implementation starts, copy the skill folder into Estatly's own `.claude/skills/` (skills are per-workspace; Estatly sessions don't see System2.dev's) — or re-read it from System2.dev. Per the skill's own guidance, verify exact SDK signatures against the official docs before writing code; AWS Blocks is new and evolving.

# Skills evaluated and not picked

The other skills in `System2.dev/.claude/skills/` are AI-agent infrastructure for the System2 platform itself and have no counterpart in the Estatly brief:

| Skill | What it covers | Why not picked |
|---|---|---|
| `cognee` | Knowledge-graph AI memory | No AI/memory feature in the brief |
| `litellm` | LLM gateway/proxy | No LLM feature in the brief |
| `composio` / `composio-mcp` | Agent access to 1000+ SaaS apps | Estatly's third-party needs (payment gateway, Tally) are direct integrations, not agent tooling |
| `mirage-vfs` | Virtual filesystem for agents | System2-specific module |
| `loop-maker` | Self-running agent loops | Dev-workflow tooling, not product spec |

Revisit this table if Estatly's scope grows AI features (e.g. an accounting copilot) — `litellm` and `cognee` would become candidates.
