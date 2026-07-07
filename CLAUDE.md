# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

**StandardLibrary** is a **documentation-only** knowledge base of programming standards. It contains no application code, no executable tooling, and no mandatory configurations — only Markdown reference documents. Its only executable artifacts are two slash commands: `/standard_research` (writes the standards) and `/standard_validation` (checks existing standards without rewriting them).

The downstream consumer of this repository is the independent **StandardBuilder** project, which reads these standards and generates the skills, commands, and workflows that AI agents use when building applications.

## The Commands

```
/standard_research   <language> [agents] [points]
/standard_validation <language> [agents] [points]
```

Both are defined at the repository root (intended location: `.claude/commands/`). When invoked, the agent **must read `Instructions.md` first** — every time — before doing anything else. Both share the same parameters, tracking-file mechanics (claim by Start + Agent, parallel sub-agents, resumability), and column structure.

- `language` (required): normalized to lowercase for the directory name.
- `agents` (optional, default `3`): how many sub-agents work **simultaneously**. Points are independent, so work is parallelized across them rather than done sequentially.
- `points` (optional): a comma-separated list of section numbers from `common/ConstituentElements.md` (e.g., `1,3,8,11`) to restrict the run to just those sections instead of all 26.

### `/standard_research` — writes the standards

Researches and writes the standard documents. It never assesses the accuracy of existing documents — that is `/standard_validation`'s job.

- **Interrupted run:** if `<language>/research-tracking.md` has unfinished rows, resume — dispatch sub-agents only for the pending/`❌` points (ask before redoing `⚠️` ones).
- **Validation feedback:** if `<language>/validation-tracking.md` exists, `/standard_research` targets the points listed in `points`, or — if omitted — every point still flagged in `validation-tracking.md` (`⚠️`/`❌`). For each, it reads that point's `Details` cell first and researches with those findings in mind; once fixed, it marks that row `✅` and clears `Details`.
- **Existing, fully-researched language, no validation file:** report creation/last-review dates (from documents' front matter) and ask whether to redo the full research; stop if declined.
- **New language:** read `common/ConstituentElements.md` (the 26-section research backbone), create `<language>/research-tracking.md` with one row per section, then launch `agents` sub-agents that each claim an unclaimed row, research that section for the target language using authoritative sources, write the corresponding document, and update the row (Status/Details/End) immediately — never in batch.

### `/standard_validation` — checks the standards (never rewrites)

Requires `<language>/` to already exist (tell the user to run `/standard_research` first if it doesn't). It never edits a standard document — it reads each one, compares it against the research backbone and current authoritative sources, and records findings in its own tracking file, `<language>/validation-tracking.md`.

- No issues → Status `✅`, Details empty.
- Document missing entirely → Status `❌`, Details: `"document missing"`.
- Issues found → Status `⚠️`, Details: the full, concrete list of inconsistencies (outdated info, factual errors, missing side-by-side alternatives, missing "not applicable" note, etc.) — specific enough that a future `/standard_research` pass can act on each one directly.
- A subsequent `/standard_research` run for the same language automatically picks up and addresses whatever it finds.

### The tracking files

Each language directory can contain two parallel tracking files, sharing the same structure (a table with one row per point) and mechanics — they are the shared boards that make both processes **parallel, interruptible, and resumable**:

- **`<language>/research-tracking.md`** — drives `/standard_research`. Columns: **Status** (empty/`✅`/`❌`/`⚠️`), **Agent**, **Research Point**, **Details**, **Start**, **End**.
- **`<language>/validation-tracking.md`** — drives `/standard_validation`. Identical column layout; its **Details** cell holds the **full list of inconsistencies found** (empty if none) — the bridge that a subsequent `/standard_research` run reads to drive feedback-driven fixes.

A sub-agent writes its **Start** timestamp **and its own Agent name** to claim a row before doing the work, so a cut-off run can be resumed by treating any row with `Start` but no `End` as a stale claim (clear it, then re-dispatch). The **Agent** column is the audit trail: if a point is redone later, the new agent's name overwrites the old one. If concurrent writes to the file collide in practice, `Instructions.md` defines a single-writer fallback where only the coordinator agent writes the file and sub-agents report status back to it instead.

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
├── Instructions.md              # Full workflow for both commands — read before acting
├── standard_research.md         # /standard_research definition (→ .claude/commands/)
├── standard_validation.md       # /standard_validation definition (→ .claude/commands/)
├── common/
│   └── ConstituentElements.md   # ⭐ Research backbone — 26 sections, NEVER moved or copied
├── rust/                        # Placeholder (empty)
└── <language>/                  # One directory per language, populated by /standard_research
    ├── research-tracking.md     # Live research board driving resume/parallel research
    ├── validation-tracking.md   # Live validation board (same structure); created by /standard_validation
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
- `/standard_research` must never assess content accuracy or write inconsistency findings — that is exclusively `/standard_validation`'s job. `/standard_validation` must never rewrite a standard document — that is exclusively `/standard_research`'s job. Each command has exactly one job.
