---
title: StandardLibrary - Research Instructions
status: draft
version: 0.4.0
created: 2026-07-05
updated: 2026-07-06
---

# Instructions for Compiling Programming Standards

These are the instructions an AI agent must follow to research and compile — or validate — the programming standards for a specific programming language inside **StandardLibrary**. They are read and executed by both the `/standard_research` and `/standard_validation` commands.

## Context You Must Understand First

- **StandardLibrary is documentation only.** Your output is Markdown reference documents — never application code, tooling, or configuration.
- **`common/ConstituentElements.md` is your research backbone.** It defines the 26 sections every programming standard must cover, with examples. That file never leaves this repository and is never copied into language directories; it is only your guide.
- **One file per section.** Each researched topic becomes its own document, so clients can later pick, override, or discard pieces individually.
- **Specific overrides general.** Language-specific documents take precedence over the top-level `common/` documents; category documents inside a language take precedence over the language's own `common/`.
- **This repo is a catalog, not a mandate.** Document the recognized standards and their alternatives (e.g., GitFlow vs. Trunk-Based); do not pick a winner. The selection happens later, in the independent **StandardBuilder** project.
- **Research and validation are separate jobs.** `/standard_research` only researches and writes standard documents — it never assesses or reports on the accuracy of existing documents. Assessing accuracy and recording inconsistencies is the exclusive job of `/standard_validation`, which never rewrites a standard document; it only annotates `validation-tracking.md`.

## Input

- **`language`** (required): the programming language to research or validate (e.g., `java`, `python`, `csharp`). Normalize it to lowercase for the directory name.
- **`agents`** (optional, default: `3`): the number of sub-agents that work **simultaneously**. Must be a positive integer. The work is not sequential — points are independent and are distributed among the sub-agents.
- **`points`** (optional, 3rd argument): a comma-separated list of section numbers from `common/ConstituentElements.md` (e.g., `1,3,8,11`). Restricts the run to only these points instead of all 26. Applies to both commands: for `/standard_research` it re-researches only these sections; for `/standard_validation` it validates only these sections. If omitted, the run considers all applicable points, subject to the resume/existing-research rules below.

## The Research/Validation Tracking File

Every run is coordinated through a tracking file: **`<language>/research-tracking.md`** for `/standard_research`, and **`<language>/validation-tracking.md`** for `/standard_validation`. Both share the exact same structure and mechanics — it is the shared board that makes the process **interruptible, resumable, and parallelizable**: if the run is cut off, a new run picks up exactly where the previous one stopped by reading this file.

The file starts with the standard front matter, followed by one table row per point taken from `common/ConstituentElements.md` (or per point listed in the `points` parameter, when given):

```markdown
---
title: {language} - Research Tracking
status: draft
version: 0.0.1
created: {today, YYYY-MM-DD}
updated: {now, YYYY-MM-DD HH:MM}
---

# Research Tracking — {language}

| Status | Agent | Research Point | Details | Start | End |
|--------|-------|----------------|---------|-------|-----|
|    |  | 1. Scope and Purpose |  |  |  |
| ✅ | Sub-agent 1 | 2. Naming Conventions |  | 2026-07-06 10:02 | 2026-07-06 10:07 |
| ⚠️ | Sub-agent 2 | 3. Formatting and Structure Style | Community split between two formatters; both documented side by side | 2026-07-06 10:02 | 2026-07-06 10:09 |
| ❌ | Sub-agent 3 | 4. Project Architecture and Directory Layout | Could not reach official docs (timeout); document not written | 2026-07-06 10:03 | 2026-07-06 10:04 |
|    |  | 5. Language Rules and Best Practices |  |  |  |
| ... | ... | ... | ... | ... | ... |
```

`validation-tracking.md` uses the identical column layout. Its `Details` column, however, holds the **full list of inconsistencies found** rather than a research problem note (see *Validation Workflow* below).

### Column semantics

| Column | Meaning |
|---|---|
| **Status** | Empty = not processed yet. `✅` = completed OK (research: document written cleanly; validation: no inconsistencies found). `❌` = failed (research: document missing/unusable; validation: document missing entirely). `⚠️` = completed but with a problem worth reviewing (research: a noteworthy caveat; validation: inconsistencies were found). |
| **Agent** | Name/label of the sub-agent that claimed and completed this row (e.g., `Sub-agent 2`). This is the audit trail: if in the future only this specific point is redone, the new agent's name overwrites the old one here, so it's always clear who produced the current content. |
| **Research Point** | Short description of the point — the section number and title from `ConstituentElements.md`. |
| **Details** | For research: the problem found, empty if OK. For validation: the full list of inconsistencies found, empty if none. |
| **Start** | Timestamp (`YYYY-MM-DD HH:MM`) when a sub-agent began working on this point. |
| **End** | Timestamp when the sub-agent finished (success or failure). |

### State rules

- **Not started:** Status empty, Agent empty, Start empty.
- **In progress (claimed):** Start and Agent filled, End empty. A sub-agent writes its Start timestamp **and its own Agent name** the moment it takes a point, so no other sub-agent takes it, and so it's visible who is working on it.
- **Finished:** End filled, Status is `✅`, `⚠️`, or `❌`, Agent is never empty.
- **Stale claim:** Start filled, End empty, but no sub-agent is actually running (the previous run was interrupted). On resume, stale claims are treated as *not started* again (clear Start and Agent before re-dispatching).
- Sub-agents must update their row **immediately** on start and on finish — never in batch at the end. The tracking file must reflect reality at any instant.
- Update the front matter `updated` field on every write.

### Concurrency note: single-writer variant

With N sub-agents writing to the same Markdown file, concurrent writes could theoretically collide (two agents saving the file at the same moment, one overwriting the other's row update). The per-row claim protocol makes this unlikely in practice, but if collisions are observed (lost claims, two agents on the same point, rows reverting to a previous state), switch to the **single-writer variant**:

- The **coordinator (main agent) is the only one that writes** the tracking file.
- Sub-agents do not touch the file. Instead, they **report events to the coordinator**: *"claiming point X as \<agent name\>"*, *"finished point X with status ✅/⚠️/❌, agent \<agent name\>, and these details"*.
- The coordinator serializes all updates: it assigns points to sub-agents (instead of letting them self-claim), writes the Start timestamp and Agent name when dispatching, and writes Status/Details/End when the sub-agent reports back.
- Everything else stays identical: same table, same column semantics, same state rules, same resumability — only the writer changes.

This variant trades a small coordination overhead for guaranteed write consistency. Prefer it when running with a high number of sub-agents or when the execution environment does not guarantee atomic file writes.

## Research Workflow (`/standard_research`)

### Step 1 — Check for existing research (and resume if interrupted)

1. Look for a directory named after the language at the repository root (e.g., `java/`).
2. **If it exists and `<language>/research-tracking.md` has unfinished rows** (empty status or stale claims), a previous run was interrupted. Inform the user how many points are done/pending, then **resume**: clear stale claims (Start and Agent) and go straight to Step 4 to research only the unfinished points (including `❌` rows; ask the user whether `⚠️` rows should also be redone). This takes priority over everything below.
3. **If it exists and the tracking file shows all rows finished** (a completed prior research), decide what to do based on the `points` parameter and on whether `<language>/validation-tracking.md` exists:

   **A. No `validation-tracking.md` yet:**
   - If **no `points` parameter** was given:
     - Report the **creation date** (oldest `created`) and **last review date** (newest `updated`) of the existing documents.
     - Tell the user they may instead pass a third parameter — a comma-separated list of section numbers — to re-research only specific points without a full re-run.
     - Ask: *"Standards for this language already exist. Do you want to redo the full research for this language, updating the existing documents?"*
       - **If yes** → reset every row of `research-tracking.md` to not-started (clear Status, Agent, Details, Start, End) and go to Step 4 for all 26 points.
       - **If no** → end the command. Do nothing else.
   - If a **`points` parameter was given** → skip the question above. Reset only the rows listed in `points` (clear Status, Agent, Details, Start, End for those rows) and go straight to Step 4 for just those points. No confirmation is needed — the explicit parameter is the user's confirmation.

   **B. `validation-tracking.md` exists:**
   - Determine the target points:
     - If a `points` parameter was given, the targets are exactly those points.
     - If not, the targets are every row in `validation-tracking.md` whose Status is not `✅` (every point still flagged with inconsistencies). If none are flagged, tell the user there is nothing pending per the last validation and stop (they can still force a full re-research by passing every point number explicitly via `points`).
   - Reset the corresponding rows of `research-tracking.md` (clear Status, Agent, Details, Start, End).
   - **Before dispatching a sub-agent to a target point, read that point's `Details` cell in `validation-tracking.md`.** Feed those observations into the sub-agent's research task as required corrections: the redo is not "research from scratch again", it is "research again and specifically address these findings."
   - Go to Step 4 for the target points, in this feedback-driven mode.
   - **On completion of each such point**, in addition to updating `research-tracking.md` as usual: set that row's Status in `validation-tracking.md` to `✅` and **clear its `Details` cell** — the finding has now been addressed.
4. **If it does not exist:** continue with Step 2. If a `points` parameter was given for a brand-new language, only create rows and research those specific points instead of all 26; otherwise research all 26.

### Step 2 — Read the research backbone

Read `common/ConstituentElements.md` in full. Its 26 sections are your research checklist:

1. Scope and Purpose
2. Naming Conventions
3. Formatting and Structure Style
4. Project Architecture and Directory Layout
5. Language Rules and Best Practices
6. Documentation and Comments
7. Error Handling, Exceptions, and Logging
8. API Design and Data Contracts
9. Security and Privacy
10. Dependency Management
11. Testing and Quality Validation
12. Version Control and Workflow
13. Automation Tools (Linters and Formatters)
14. Build, Packaging, and Containerization
15. Performance and Resource Efficiency
16. Concurrency and Thread Safety
17. Governance and Standard Updates
18. Ecosystem Conventions
19. Development Toolchain
20. Date, Time, Time Zones, and Numeric Formats
21. Internationalization and Localization
22. Character Encoding and Unicode
23. Data Standards and Persistence
24. CI/CD and Deployment
25. Accessibility (a11y)
26. Distributed Messaging and Asynchronous Communication

### Step 3 — Create the research tracking file

Create `<language>/research-tracking.md` as specified in *The Research/Validation Tracking File* above, with **one row per section** of `ConstituentElements.md` (26 rows, plus one row for each extra language-specific category you decide to add) — or only the rows listed in `points`, when that parameter is given. All rows start empty, including Agent.

### Step 4 — Research each point in parallel with sub-agents

Launch **`agents`** sub-agents (default 3) that work **simultaneously**. Coordination protocol:

1. Each sub-agent takes the next unclaimed row (empty Status and empty Start), writes its **Start timestamp and its own Agent name/label** to claim it, researches that point, writes the corresponding standard document (Step 5 rules), and then updates the row: Status (`✅`/`⚠️`/`❌`), Agent (kept/confirmed), Details (only if there is a problem), and End timestamp.
2. If this run was triggered by the feedback-driven mode of Step 1.B, the sub-agent must also incorporate the observations read from `validation-tracking.md`'s `Details` cell for that point, and perform the corresponding `validation-tracking.md` update (Status `✅`, Details cleared) once the document is rewritten.
3. Sub-agents keep taking rows until none remain unclaimed.
4. A sub-agent failure must never leave a row half-updated silently: on any error, set Status `❌`, describe the error in Details, and set End. Agent must still be recorded.
5. Points are independent — no ordering between them is required.
6. **The Agent column must never be left empty on a finished row** — it is the audit trail of who produced the current content for that point.

For **every** point, the researching sub-agent must:

- Research how that topic applies specifically to the target language.
- Prioritize **authoritative sources**: official language documentation, widely adopted style guides (e.g., PEP 8, Google Style Guides, PSR, Effective Go, Microsoft C# conventions), recognized security references (OWASP, CERT), and the language's dominant tooling.
- When the community is split, **document the main alternatives side by side** with their trade-offs (remember: catalog, not mandate).
- Include concrete ✅/❌ code examples in the target language, following the style used in `ConstituentElements.md`.
- If a section does not apply to the language (e.g., manual memory management in a garbage-collected language), still create the document with a brief explicit note stating why it does not apply.

### Step 5 — Write one document per topic in the corresponding folder

1. Create the language directory and its category subfolders. Suggested baseline mapping (adjust categories if the language demands it):

   | Backbone section | Target file |
   |---|---|
   | 1. Scope and Purpose | `<language>/common/scope-and-purpose.md` |
   | 2. Naming Conventions | `<language>/common/naming-conventions.md` |
   | 3. Formatting and Structure Style | `<language>/common/formatting-and-structure.md` |
   | 4. Project Architecture and Directory Layout | `<language>/architecture/project-architecture-and-layout.md` |
   | 5. Language Rules and Best Practices | `<language>/common/language-rules-and-best-practices.md` |
   | 6. Documentation and Comments | `<language>/documentation/documentation-and-comments.md` |
   | 7. Error Handling, Exceptions, and Logging | `<language>/error-handling/error-handling-and-logging.md` |
   | 8. API Design and Data Contracts | `<language>/api/api-design-and-data-contracts.md` |
   | 9. Security and Privacy | `<language>/security/security-and-privacy.md` |
   | 10. Dependency Management | `<language>/dependencies/dependency-management.md` |
   | 11. Testing and Quality Validation | `<language>/testing/testing-and-quality.md` |
   | 12. Version Control and Workflow | `<language>/version-control/version-control-and-workflow.md` |
   | 13. Automation Tools | `<language>/automation/linters-and-formatters.md` |
   | 14. Build, Packaging, and Containerization | `<language>/build/build-packaging-and-containerization.md` |
   | 15. Performance and Resource Efficiency | `<language>/performance/performance-and-resources.md` |
   | 16. Concurrency and Thread Safety | `<language>/concurrency/concurrency-and-thread-safety.md` |
   | 17. Governance and Standard Updates | `<language>/governance/governance-and-updates.md` |
   | 18. Ecosystem Conventions | `<language>/ecosystem/ecosystem-conventions.md` |
   | 19. Development Toolchain | `<language>/toolchain/development-toolchain.md` |
   | 20. Date, Time, Time Zones, and Numeric Formats | `<language>/common/date-time-and-numeric-formats.md` |
   | 21. Internationalization and Localization | `<language>/common/internationalization-and-localization.md` |
   | 22. Character Encoding and Unicode | `<language>/common/character-encoding-and-unicode.md` |
   | 23. Data Standards and Persistence | `<language>/data/data-standards-and-persistence.md` |
   | 24. CI/CD and Deployment | `<language>/cicd/cicd-and-deployment.md` |
   | 25. Accessibility (a11y) | `<language>/accessibility/accessibility.md` |
   | 26. Distributed Messaging and Asynchronous Communication | `<language>/messaging/distributed-messaging.md` |

2. Additional language-relevant categories are welcome when justified (e.g., `<language>/database/` for data-access standards, `<language>/frontend/` for UI-heavy languages). Each extra category also follows the one-file-per-section rule.
3. **Every generated file must start with the tracking front matter:**

   ```yaml
   ---
   title: {Document name}
   status: draft
   version: 0.0.1
   created: {today, YYYY-MM-DD}
   updated: {today, YYYY-MM-DD}
   ---
   ```

   On a redo (full, `points`-scoped, or validation-feedback-driven), keep the original `created` date and bump `version` and `updated`.

### Step 6 — Completion check

When no unfinished rows remain, the coordinator (main agent) verifies completeness:

- [ ] The tracking file has no empty-status rows and no stale claims; every row has Start, Agent, and End.
- [ ] Every `✅`/`⚠️` row has its corresponding document (or an explicit "not applicable" document); every `❌` row is reported to the user for a re-run.
- [ ] Every document has the tracking front matter with correct dates and version.
- [ ] Every document contains language-specific examples, not generic placeholders.
- [ ] No application code, tooling, or configuration was generated — documentation only.
- [ ] File and folder names are lowercase, hyphen-separated, and consistent with the mapping above.
- [ ] If this run was triggered by validation feedback (Step 1.B), every addressed row's entry in `validation-tracking.md` is `✅` with an empty `Details` cell.

Report a summary to the user: language, number of documents created or updated, count of `✅`/`⚠️`/`❌` rows, categories generated, and a reminder that the next step is the independent **StandardBuilder** repository. If any `❌` rows remain, tell the user they can simply re-run `/standard_research <language> [agents] [points]` to resume only the failed/pending points.

## Validation Workflow (`/standard_validation`)

`/standard_validation <language> [agents] [points]` shares the same parameters, tracking-file mechanics (claim by Start + Agent, parallel sub-agents, resumability), and column structure as research — but it runs against its own tracking file, **`<language>/validation-tracking.md`**, and it never modifies a standard document.

Requires `<language>/` to already exist. If it doesn't, tell the user to run `/standard_research <language>` first and stop.

### Differences from `/standard_research`

- **Read-only with respect to the standard documents.** A validating sub-agent never edits `<language>/<category>/*.md` — it only reads it (or notices it's missing).
- **What each sub-agent does per claimed point:** read the existing document for that section and compare it against the relevant part of `common/ConstituentElements.md` and against current authoritative sources, checking for:
  - missing or incomplete coverage of the section;
  - outdated information (APIs, versions, deprecated tooling, superseded style guides);
  - factual errors;
  - unclear or contradictory guidance;
  - missing side-by-side alternatives where the ecosystem is split;
  - a missing explicit "not applicable" note where a section legitimately doesn't apply;
  - stale or malformed front matter.
- **Recording results** in `validation-tracking.md`:
  - No issues found → Status `✅`, Details empty.
  - Document missing entirely → Status `❌`, Details: `"document missing"`.
  - Issues found → Status `⚠️`, Details: the **full list of inconsistencies found**, written concretely enough that a future `/standard_research` re-run can act on each one without re-deriving the analysis.
  - Agent, Start, End follow the exact same claim/finish rules as research (Step 4 above).
- **Existing-run behavior mirrors Step 1** of the research workflow: if `validation-tracking.md` has unfinished rows, resume. If it is fully finished, report the last-run date and ask whether to re-validate — unless `points` is given, in which case just re-validate those points without asking.
- **This command never triggers a research redo itself.** It only ever populates/updates `validation-tracking.md`. Acting on its findings is the job of a subsequent `/standard_research <language> [agents] [points]` run, which picks up `validation-tracking.md` automatically per Step 1.B above.
- **Report:** summarize counts of `✅`/`⚠️`/`❌`, list the flagged points, and remind the user that running `/standard_research` for this language will pick up and address these findings automatically.

## What You Must NOT Do

- ❌ Do not copy `ConstituentElements.md` into the language directory — it stays where it is.
- ❌ Do not merge multiple sections into a single document.
- ❌ Do not choose a single "winning" convention when the ecosystem has several recognized alternatives — document them all.
- ❌ Do not generate skills, commands, workflows, or any development tooling — that is **StandardBuilder**'s job, in its own repository.
- ❌ Do not research or validate a point without first claiming its row (Start + Agent) in the tracking file, and never update the tracking file in batch at the end — it must reflect reality at any instant so the process can be cut and resumed.
- ❌ Do not leave a finished row's **Agent** column empty — every completed row must record who did the work.
- ❌ `/standard_research` must never assess accuracy or write inconsistency findings — that is exclusively `/standard_validation`'s job. `/standard_validation` must never rewrite a standard document — that is exclusively `/standard_research`'s job.
