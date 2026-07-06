# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

**StandardLibrary** is a **documentation-only** knowledge base of programming standards. It contains no application code, no executable tooling, and no mandatory configurations — only Markdown reference documents. Its sole output mechanism is a single slash command: `/standard_research`.

The downstream consumer of this repository is the independent **StandardBuilder** project, which reads these standards and generates the skills, commands, and workflows that AI agents use when building applications.

## The Only Command

```
/standard_research <language>
```

Defined in `standard_research.md` at the repository root (intended location: `.claude/commands/standard_research.md`). When invoked, the agent **must read `Instructions.md` first** — every time — before doing anything else.

What the command does:
- If `<language>/` already exists: reports creation/last-review dates, asks the user whether to re-review, and stops if declined.
- If `<language>/` does not exist: reads `common/ConstituentElements.md` (the 26-section research backbone), researches each section for the target language using authoritative sources, and writes one document per section into `<language>/<category>/`.

## Document Authoring Rules

**Every file in this repository must start with this front matter:**

```yaml
---
title: {Document name}
status: draft
version: 0.0.1
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

The `/standard_research` command reads `created` and `updated` to report research age.

**File and folder naming:** lowercase, hyphen-separated (e.g., `naming-conventions.md`, `error-handling/`).

**One document per section** — never merge multiple backbone sections into a single file.

## Repository Architecture

```
/
├── Instructions.md              # Full workflow for /standard_research — read before acting
├── standard_research.md         # Slash command definition (→ .claude/commands/)
├── common/
│   └── ConstituentElements.md   # ⭐ Research backbone — 26 sections, NEVER moved or copied
├── rust/                        # Placeholder (empty)
└── <language>/                  # One directory per language, populated by /standard_research
    ├── common/                  # General rules (scope, naming, formatting, language rules, time/unicode)
    ├── documentation/           # Code docs and comments
    ├── error-handling/
    ├── security/
    ├── dependencies/
    ├── testing/
    ├── version-control/
    ├── automation/
    ├── build/                   # Build, packaging, containerization
    ├── performance/
    ├── concurrency/
    ├── governance/
    ├── api/                     # API design and data contracts
    ├── architecture/            # Project architecture and layout
    ├── ecosystem/
    ├── toolchain/
    ├── data/                    # Data standards and persistence
    ├── cicd/                    # CI/CD and deployment
    ├── accessibility/           # a11y (when applicable)
    └── messaging/               # Distributed messaging (when applicable)
```

## Precedence Rules

Specific overrides general: a rule in `<language>/security/` overrides the equivalent in `<language>/common/`, which overrides `common/`. This hierarchy is intentional — clients layer and override at any granularity.

## Hard Constraints

- `common/ConstituentElements.md` **never leaves this directory** — it is the guide, not content to be distributed.
- When documenting conventions where the community is split (e.g., GitFlow vs. Trunk-Based), **document all recognized alternatives side by side** — never pick a winner. Selection happens in StandardBuilder.
- If a backbone section does not apply to the target language, still create the document with an explicit note explaining why.
