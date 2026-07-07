---
title: StandardLibrary - README
status: draft
version: 0.0.6
created: 2026-07-05
updated: 2026-07-06
---

# StandardLibrary

A library of programming standards. Nothing more, nothing less.

## Purpose

**StandardLibrary** is a **centralized source of reference information for programming standards**, covering both general (language-agnostic) standards and language-specific standards.

Its mission is to provide AI agents with the reference information they need about the programming standards of a specific language, so that downstream tools can generate the skills, commands, and workflows required for other agents to develop applications following those standards.

> **Golden Rule:** This repository contains **reference documentation only — nothing else**. No application code, no tooling, no executable configuration. It is purely a knowledge base. The only executable artifacts it ships are two commands: `/standard_research` and `/standard_validation`.

## How to Use This Repository (Client Quick Start)

Two commands cover the whole client workflow:

```
/standard_research <language> [agents] [points]
/standard_validation <language> [agents] [points]
```

- **`language`** (required): the language to research or validate.
- **`agents`** (optional, default `3`): number of sub-agents working in parallel.
- **`points`** (optional): a comma-separated list of section numbers from `common/ConstituentElements.md` (e.g., `1,3,8,11`) to restrict the run to just those sections instead of all 26.

### `/standard_research` — write the standards

Examples:

```
/standard_research java              # research Java standards with 3 parallel sub-agents (default)
/standard_research python 5          # research Python standards with 5 parallel sub-agents
/standard_research rust 2 1,3,8,11   # redo only points 1, 3, 8, 11 for Rust with 2 sub-agents
```

Running this command results in a **populated directory** (`java/` in the first example) full of programming-standards documentation for that language, organized by category, one file per section. `/standard_research` only ever researches and writes documents — it never assesses their accuracy; that's `/standard_validation`'s job (below).

1. **If a previous run was interrupted**, it detects the pending points in `<language>/research-tracking.md` and **resumes exactly where it left off**.
2. **If the language already exists** (fully researched) **and there's no `validation-tracking.md` yet**:
   - Without a `points` parameter, it reports the creation and last-review dates of the existing research, mentions that you can pass `points` to redo only specific sections, and asks whether to redo the **full** research. Decline → the command ends, nothing changes.
   - With a `points` parameter, it skips the question and redoes only those sections directly, updating the existing documents.
3. **If `<language>/validation-tracking.md` exists** (a prior `/standard_validation` run found things to fix), `/standard_research` targets the points listed in `points`, or — if `points` is omitted — every point still flagged in `validation-tracking.md`. For each, it reads that point's `Details` cell first and researches with those specific findings in mind. Once a point is fixed, its `validation-tracking.md` row is marked `✅` and `Details` is cleared.
4. **If the language does not exist**, it reads the research backbone ([`common/ConstituentElements.md`](common/ConstituentElements.md)), creates the tracking file, and dispatches **N parallel sub-agents** (2nd parameter, default 3) that research the 26 sections (or just the ones in `points`) and write one document per topic into the corresponding category folder.

### `/standard_validation` — check the standards

Examples:

```
/standard_validation java            # validate all of Java's existing documents with 3 sub-agents (default)
/standard_validation rust 2 1,3,8,11 # validate only points 1, 3, 8, 11 for Rust with 2 sub-agents
```

`/standard_validation` requires the language to already exist (run `/standard_research` first otherwise). It never edits a standard document — it only reads each one, compares it against the research backbone and current authoritative sources, and records findings in its own tracking file, `<language>/validation-tracking.md`. A subsequent `/standard_research` run for the same language automatically picks up and addresses whatever it finds (see point 3 above).

### The tracking files

Each language directory can contain two parallel tracking files, `research-tracking.md` and `validation-tracking.md`, sharing the same structure: a table with one row per point, columns **Status** (empty = pending, ✅ done/no issues, ❌ error/missing, ⚠️ warning/issues found), **Agent** (name of the sub-agent that did the work — an audit trail if the point is redone later by someone else), **Research Point** (short description), **Details** (the problem or, for validation, the full list of inconsistencies found — empty if OK), **Start**, and **End** timestamps. Sub-agents claim rows and update them in real time, which is what makes both processes **parallel, interruptible, and resumable**.

Both commands follow the detailed workflow defined in [`Instructions.md`](Instructions.md).

## What Comes Next: StandardBuilder

> **Important:** StandardLibrary is only the **library**. It stores reference documentation — it does not build anything.
>
> The next step is the independent project **[StandardBuilder]**. StandardBuilder reads this StandardLibrary and generates the actual output: the **skills, commands, workflows, and every tool** an AI agent needs to follow the development standards while building applications.
>
> Client flow: **1)** populate/consult standards here with `/standard_research` → **2)** go to the **StandardBuilder** repository to turn the selected standards into enforceable agent tooling.

## Key Files

| File | Role |
|------|------|
| [`README.md`](README.md) | This file. Context for any AI agent or human consulting the repository. |
| [`Instructions.md`](Instructions.md) | The step-by-step workflow both commands must follow, including the shared research/validation tracking-file mechanics. Read by `/standard_research` and `/standard_validation`. |
| [`common/ConstituentElements.md`](common/ConstituentElements.md) | The research backbone: defines the 26 sections every standard must cover. It never leaves this repo — it is only the guide for elaborating a standard. |
| [`.claude/commands/standard_research.md`](.claude/commands/standard_research.md) | The `/standard_research` command definition. |
| [`.claude/commands/standard_validation.md`](.claude/commands/standard_validation.md) | The `/standard_validation` command definition. |

## Repository Structure

```
/
├── README.md
├── Instructions.md              # Workflow followed by /standard_research and /standard_validation
├── .claude/
│   └── commands/
│       ├── standard_research.md   # Researches and writes standards
│       └── standard_validation.md # Validates existing standards (never rewrites them)
├── common/                      # General standards, applicable to ALL languages
│   ├── ConstituentElements.md   # ⭐ Research backbone (guide only — never copied elsewhere)
│   ├── <one file per standard section>
│   └── ...
├── java/                        # Example: language-specific directory
│   ├── research-tracking.md     # Live research board: status, agent, details, start/end per point
│   ├── validation-tracking.md   # Live validation board (same structure); created by /standard_validation
│   ├── common/                  # General rules for Java
│   ├── database/                # Database standards for Java
│   ├── <other-category>/        # Additional categories as needed
│   └── ...
├── <other-language>/
│   └── ...
└── ...
```

### Structural rules

- **One directory per language.** Every programming language has its own subdirectory at the root of the repository (e.g., `java/`, `python/`, `csharp/`).
- **Subfolders per category inside each language.** Each language directory is divided into category subfolders. For example, for Java: general rules go in `java/common/`, database standards in `java/database/`, and so on.
- **One document per section.** Instead of a single large document with multiple sections, each section of a standard is a **separate file**. This granularity is intentional: it allows clients to pick, override, or discard individual pieces of a standard without touching the rest.
- **Document tracking header.** Every file in this project starts with a metadata front matter block to track changes:

  ```yaml
  ---
  title: {Document name}
  status: draft
  version: 0.0.1
  created: 2026-07-05
  updated: 2026-07-05
  ---
  ```

  The `/standard_research` command uses the `created` and `updated` fields to report the age of existing research.

- **Tracking-file header.** `research-tracking.md` and `validation-tracking.md` use the same front matter block, plus a table with columns **Status | Agent | Research Point | Details | Start | End** — full column semantics are defined in [`Instructions.md`](Instructions.md).

## The Research Backbone: `common/ConstituentElements.md`

The file [`common/ConstituentElements.md`](common/ConstituentElements.md) is the **cornerstone document** of this repository. It defines the constituent elements that any robust programming standard must include — 26 sections covering scope, naming, formatting, project layout, language rules, documentation, error handling, API design, security, dependencies, testing, version control, automation, build & packaging, performance, concurrency, governance, ecosystem conventions, development toolchain, date/time/numeric formats, internationalization and localization, character encoding and Unicode, data standards and persistence, CI/CD and deployment, accessibility, and distributed messaging — each with concrete examples.

**This file never goes anywhere.** It is not copied into language directories or client projects. It is exclusively the **guide for elaborating a standard**:

1. **As the research checklist.** `/standard_research` iterates over each of its 26 sections and researches how that topic applies to the target language (e.g., for the *Naming Conventions* section in Python, research PEP 8 naming rules).
2. **As the structure blueprint.** Each section maps to a **separate document** in the language's directory. For example, for Java:
   - Section 2 (Naming Conventions) → `java/common/naming-conventions.md`
   - Section 8 (API Design and Data Contracts) → `java/api/api-design-and-data-contracts.md`
   - Section 9 (Security and Privacy) → `java/security/security-and-privacy.md`
   - Section 11 (Testing) → `java/testing/testing-and-quality.md`
   - Section 18 (Ecosystem Conventions) → `java/ecosystem/ecosystem-conventions.md`
   - Section 19 (Development Toolchain) → `java/toolchain/development-toolchain.md`
3. **As the completeness validator.** A language's standard set is complete only when every applicable section has a corresponding language-specific document (or an explicit note stating why a section does not apply, e.g., manual memory management in garbage-collected languages).

## Precedence Rules

- **Specific standards override general standards.** A rule defined in a language-specific directory takes precedence over the equivalent rule in the top-level `common/` directory. Likewise, more specific categories within a language can override the language's own `common/` rules.
- The intended workflow for a client is: take **all** the standards of a language as a base, then **override, delete, or add** only what is needed (this selection and build step happens in **StandardBuilder**, not here).

## Scope and Constraints

- ✅ Reference documentation about programming standards (general and per-language).
- ✅ One file per standard section, organized in category subfolders per language.
- ✅ Two commands: `/standard_research <language> [agents] [points]`, which writes/updates the documentation with N parallel sub-agents (default 3), resumable via `research-tracking.md`; and `/standard_validation <language> [agents] [points]`, which checks existing documentation for issues without rewriting it, resumable via `validation-tracking.md`.
- ❌ No application code.
- ❌ No mandatory, fixed standard — everything here is a **reference catalog**, not a mandate.
- ❌ No generation of skills, commands, or workflows for development — that is **StandardBuilder**'s job.
- ❌ `/standard_research` never validates content accuracy; `/standard_validation` never rewrites a document — each command has exactly one job.

## Summary for AI Agents Consuming This Repository

If you are an AI agent reading this repository:

- Treat everything here as **reference material** about programming standards.
- To populate or refresh a language's standards, run `/standard_research <language> [agents] [points]` — the workflow is defined in `Instructions.md`. If a run is interrupted, re-running the command resumes from `<language>/research-tracking.md`. Pass `points` (comma-separated section numbers) to redo only specific sections.
- To check existing standards for inconsistencies without rewriting them, run `/standard_validation <language> [agents] [points]`. Its findings land in `<language>/validation-tracking.md`; a subsequent `/standard_research` run for that language automatically reads and addresses them.
- `common/ConstituentElements.md` is the research backbone: it defines every section a standard must cover, and it stays in this repository.
- Language-specific and category-specific documents **override** more general ones.
- To turn these standards into skills, commands, and workflows for development agents, go to the independent **StandardBuilder** repository — that is the next step, not this repo's job.
