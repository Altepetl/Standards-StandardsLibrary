---
title: StandardLibrary - Research Instructions
status: draft
version: 0.3.0
created: 2026-07-05
updated: 2026-07-06
---

# Instructions for Compiling Programming Standards

These are the instructions an AI agent must follow to research and compile the programming standards for a specific programming language inside **StandardLibrary**. They are read and executed by the `/standard_research` command.

## Context You Must Understand First

- **StandardLibrary is documentation only.** Your output is Markdown reference documents — never application code, tooling, or configuration.
- **`common/ConstituentElements.md` is your research backbone.** It defines the 26 sections every programming standard must cover, with examples. That file never leaves this repository and is never copied into language directories; it is only your guide.
- **One file per section.** Each researched topic becomes its own document, so clients can later pick, override, or discard pieces individually.
- **Specific overrides general.** Language-specific documents take precedence over the top-level `common/` documents; category documents inside a language take precedence over the language's own `common/`.
- **This repo is a catalog, not a mandate.** Document the recognized standards and their alternatives (e.g., GitFlow vs. Trunk-Based); do not pick a winner. The selection happens later, in the independent **StandardBuilder** project.

## Input

- **`language`** (required): the programming language to research (e.g., `java`, `python`, `csharp`). Normalize it to lowercase for the directory name.
- **`agents`** (optional, default: `3`): the number of sub-agents that will research sections **simultaneously**. Must be a positive integer. The research is not sequential — sections are independent and are distributed among the sub-agents.

## The Research Tracking File

Every research run is coordinated through a tracking file: **`<language>/research-tracking.md`**. It is the shared board that makes the process **interruptible, resumable, and parallelizable**: if the run is cut off, a new run picks up exactly where the previous one stopped by reading this file.

The file starts with the standard front matter, followed by one table row per research point taken from `common/ConstituentElements.md`:

```markdown
---
title: {language} - Research Tracking
status: draft
version: 0.0.1
created: {today, YYYY-MM-DD}
updated: {now, YYYY-MM-DD HH:MM}
---

# Research Tracking — {language}

| Status | Research Point | Details | Start | End |
|--------|----------------|---------|-------|-----|
|    | 1. Scope and Purpose |  |  |  |
| ✅ | 2. Naming Conventions |  | 2026-07-06 10:02 | 2026-07-06 10:07 |
| ⚠️ | 3. Formatting and Structure Style | Community split between two formatters; both documented side by side | 2026-07-06 10:02 | 2026-07-06 10:09 |
| ❌ | 4. Project Architecture and Directory Layout | Could not reach official docs (timeout); document not written | 2026-07-06 10:03 | 2026-07-06 10:04 |
|    | 5. Language Rules and Best Practices |  |  |  |
| ... | ... | ... | ... | ... |
```

### Column semantics

| Column | Meaning |
|---|---|
| **Status** | Empty = not researched yet. `✅` = research completed OK. `❌` = research failed (document missing or unusable). `⚠️` = research completed but with a problem worth reviewing. |
| **Research Point** | Short description of the point being researched — the section number and title from `ConstituentElements.md`. |
| **Details** | Long description of the problem found during the research. **Empty if the research is OK.** |
| **Start** | Timestamp (`YYYY-MM-DD HH:MM`) when a sub-agent began researching this point. |
| **End** | Timestamp when the sub-agent finished (success or failure). |

### State rules

- **Not started:** Status empty, Start empty.
- **In progress (claimed):** Start filled, End empty. A sub-agent writes the Start timestamp the moment it takes a point, so no other sub-agent takes it.
- **Finished:** End filled, Status is `✅`, `⚠️`, or `❌`.
- **Stale claim:** Start filled, End empty, but no sub-agent is actually running (the previous run was interrupted). On resume, stale claims are treated as *not started* again (clear Start before re-dispatching).
- Sub-agents must update their row **immediately** on start and on finish — never in batch at the end. The tracking file must reflect reality at any instant.
- Update the front matter `updated` field on every write.

### Concurrency note: single-writer variant

With N sub-agents writing to the same Markdown file, concurrent writes could theoretically collide (two agents saving the file at the same moment, one overwriting the other's row update). The per-row claim protocol makes this unlikely in practice, but if collisions are observed (lost claims, two agents researching the same point, rows reverting to a previous state), switch to the **single-writer variant**:

- The **coordinator (main agent) is the only one that writes** `research-tracking.md`.
- Sub-agents do not touch the file. Instead, they **report events to the coordinator**: *"claiming point X"*, *"finished point X with status ✅/⚠️/❌ and these details"*.
- The coordinator serializes all updates: it assigns points to sub-agents (instead of letting them self-claim), writes the Start timestamp when dispatching, and writes Status/Details/End when the sub-agent reports back.
- Everything else stays identical: same table, same column semantics, same state rules, same resumability — only the writer changes.

This variant trades a small coordination overhead for guaranteed write consistency. Prefer it when running with a high number of sub-agents or when the execution environment does not guarantee atomic file writes.

## Workflow

### Step 1 — Check for existing research (and resume if interrupted)

1. Look for a directory named after the language at the repository root (e.g., `java/`).
2. **If it exists and `<language>/research-tracking.md` has unfinished rows** (empty status or stale claims), a previous run was interrupted. Inform the user how many points are done/pending, then **resume**: clear stale claims and go straight to Step 4 to research only the unfinished points (including `❌` rows; ask the user whether `⚠️` rows should also be redone).
3. **If it exists and the tracking file shows all rows finished:**
   - Read the front matter (`created`, `updated`) of the documents inside it and report to the user:
     - The **creation date** of the research (oldest `created` value).
     - The **last review date** (newest `updated` value).
   - Ask the user: *"Standards for this language already exist. Do you want your AI agent to review and improve the existing research? AI agents evolve quickly, so a re-review usually sharpens what is already compiled."*
   - **If the user declines → end the command. Do nothing else.**
   - **If the user accepts →** perform a review pass: **reset the tracking file** (all rows back to not-started, so the review is also interruptible and parallel), then follow Steps 4–5 in review mode — each sub-agent re-reads the existing document for its point, verifies it against `common/ConstituentElements.md`, researches what changed in the ecosystem since the `updated` date, improves the document, bumps its `version`, and refreshes `updated`. Then go to Step 6.
4. **If it does not exist:** continue with Step 2.

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

Create `<language>/research-tracking.md` as specified in *The Research Tracking File* above, with **one row per section** of `ConstituentElements.md` (26 rows, plus one row for each extra language-specific category you decide to add). All rows start empty.

### Step 4 — Research each section in parallel with sub-agents

Launch **`agents`** sub-agents (default 3) that work **simultaneously**. Coordination protocol:

1. Each sub-agent takes the next unclaimed row (empty Status and empty Start), writes its Start timestamp to claim it, researches that point, writes the corresponding standard document (Step 5 rules), and then updates the row: Status (`✅`/`⚠️`/`❌`), Details (only if there is a problem), and End timestamp.
2. Sub-agents keep taking rows until none remain unclaimed.
3. A sub-agent failure must never leave a row half-updated silently: on any error, set Status `❌`, describe the error in Details, and set End.
4. Sections are independent — no ordering between them is required.

For **every** section, the researching sub-agent must:

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

### Step 6 — Final validation

When no unfinished rows remain, the coordinator (main agent) verifies completeness:

- [ ] The tracking file has no empty-status rows and no stale claims; every row has Start and End.
- [ ] Every `✅`/`⚠️` row has its corresponding document (or an explicit "not applicable" document); every `❌` row is reported to the user for a re-run.
- [ ] Every document has the tracking front matter with correct dates and version.
- [ ] Every document contains language-specific examples, not generic placeholders.
- [ ] No application code, tooling, or configuration was generated — documentation only.
- [ ] File and folder names are lowercase, hyphen-separated, and consistent with the mapping above.

Report a summary to the user: language, number of documents created or updated, count of `✅`/`⚠️`/`❌` rows, categories generated, and a reminder that the next step is the **StandardBuilder** repository. If any `❌` rows remain, tell the user they can simply re-run `/standard_research <language>` to resume only the failed/pending points.

## What You Must NOT Do

- ❌ Do not copy `ConstituentElements.md` into the language directory — it stays where it is.
- ❌ Do not merge multiple sections into a single document.
- ❌ Do not choose a single "winning" convention when the ecosystem has several recognized alternatives — document them all.
- ❌ Do not generate skills, commands, workflows, or any development tooling — that is **StandardBuilder**'s job, in its own repository.
- ❌ Do not research a point without first claiming its row in `research-tracking.md`, and never update the tracking file in batch at the end — it must reflect reality at any instant so the process can be cut and resumed.
