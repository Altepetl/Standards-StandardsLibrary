---
title: StandardLibrary - README
status: draft
version: 0.0.5
created: 2026-07-05
updated: 2026-07-06
---

# StandardLibrary

A library of programming standards. Nothing more, nothing less.

## Purpose

**StandardLibrary** is a **centralized source of reference information for programming standards**, covering both general (language-agnostic) standards and language-specific standards.

Its mission is to provide AI agents with the reference information they need about the programming standards of a specific language, so that downstream tools can generate the skills, commands, and workflows required for other agents to develop applications following those standards.

> **Golden Rule:** This repository contains **reference documentation only — nothing else**. No application code, no tooling, no executable configuration. It is purely a knowledge base. The only executable artifact it ships is a single research command: `/standard_research`.

## How to Use This Repository (Client Quick Start)

This is the **only thing** a client needs to do:

```
/standard_research <language> [agents]
```

Examples:

```
/standard_research java        # research Java standards with 3 parallel sub-agents (default)
/standard_research python 5    # research Python standards with 5 parallel sub-agents
```

Running this command results in a **populated directory** (`java/` in the first example) full of programming-standards documentation for that language, organized by category, one file per section.

That's it. The command handles everything else:

1. **If a previous run was interrupted**, it detects the pending points in `<language>/research-tracking.md` and **resumes exactly where it left off**.
2. **If the language already exists** (fully researched), it shows the creation date and last review date of the existing research, and asks whether you want your AI agent to review and improve it (AI agents evolve fast — a re-review usually sharpens the existing research). If you decline, the command simply ends.
3. **If the language does not exist**, it reads the research backbone ([`common/ConstituentElements.md`](common/ConstituentElements.md)), creates the tracking file, and dispatches **N parallel sub-agents** (2nd parameter, default 3) that research the 26 sections defined there and write one document per topic into the corresponding category folder.

### The research tracking file

Each language directory contains a `research-tracking.md`: a table with one row per research point, with columns **Status** (empty = pending, ✅ done, ❌ error, ⚠️ warning), **Research Point** (short description), **Details** (the problem found, empty if OK), **Start**, and **End** timestamps. Sub-agents claim rows and update them in real time, which is what makes the process **parallel, interruptible, and resumable**.

The command follows the detailed workflow defined in [`Instructions.md`](Instructions.md).

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
| [`Instructions.md`](Instructions.md) | The step-by-step workflow the research agent must follow. Read by `/standard_research`. |
| [`common/ConstituentElements.md`](common/ConstituentElements.md) | The research backbone: defines the 26 sections every standard must cover. It never leaves this repo — it is only the guide for elaborating a standard. |
| [`.claude/commands/standard_research.md`](.claude/commands/standard_research.md) | The `/standard_research` command definition. |

## Repository Structure

```
/
├── README.md
├── Instructions.md              # Workflow followed by /standard_research
├── .claude/
│   └── commands/
│       └── standard_research.md # The only command in this repository
├── common/                      # General standards, applicable to ALL languages
│   ├── ConstituentElements.md   # ⭐ Research backbone (guide only — never copied elsewhere)
│   ├── <one file per standard section>
│   └── ...
├── java/                        # Example: language-specific directory
│   ├── research-tracking.md     # Live research board: status, details, start/end per point
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
- ✅ One single command: `/standard_research <language> [agents]`, which populates the documentation with N parallel sub-agents (default 3), resumable via each language's `research-tracking.md`.
- ❌ No application code.
- ❌ No mandatory, fixed standard — everything here is a **reference catalog**, not a mandate.
- ❌ No generation of skills, commands, or workflows for development — that is **StandardBuilder**'s job.

## Summary for AI Agents Consuming This Repository

If you are an AI agent reading this repository:

- Treat everything here as **reference material** about programming standards.
- To populate or refresh a language's standards, run `/standard_research <language> [agents]` — the workflow is defined in `Instructions.md`. If a run is interrupted, re-running the command resumes from `<language>/research-tracking.md`.
- `common/ConstituentElements.md` is the research backbone: it defines every section a standard must cover, and it stays in this repository.
- Language-specific and category-specific documents **override** more general ones.
- To turn these standards into skills, commands, and workflows for development agents, go to the independent **StandardBuilder** repository — that is the next step, not this repo's job.
