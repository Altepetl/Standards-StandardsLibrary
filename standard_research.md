---
title: standard_research command
status: draft
version: 0.2.0
created: 2026-07-05
updated: 2026-07-06
description: Research and compile the programming standards for a specific language into StandardLibrary, in parallel with N sub-agents
argument-hint: <language> [agents]
---

# /standard_research $ARGUMENTS

You are the **coordinator agent** of **StandardLibrary**. Your job is to orchestrate the research and compilation of the programming standards for a language, using parallel sub-agents.

## Parameters

- **`language`** (1st argument, required): the language to research. If missing, ask the user and stop until you have it. Normalize to lowercase (e.g., `Java` → `java`).
- **`agents`** (2nd argument, optional): how many sub-agents research **simultaneously**. **Default: 3.** Must be a positive integer; if invalid, fall back to 3.

## Before anything else

Read the file `Instructions.md` at the repository root. It contains the complete workflow, including the specification of the research tracking file and the sub-agent coordination protocol. Do not proceed from memory — read it first, every time.

## Execution summary (details live in Instructions.md)

1. **Resume check.** If `<language>/research-tracking.md` exists with unfinished rows (empty status, or claimed rows with Start but no End), a previous run was interrupted. Report progress (done/pending counts), clear stale claims, and **resume**: dispatch sub-agents only for the unfinished points (including `❌` rows; ask about redoing `⚠️` rows). This is the whole point of the tracking file — the process can be cut and restarted where it left off.

2. **Existing research check.** If the language directory exists and its tracking file is fully finished, report the **creation date** (oldest `created`) and **last review date** (newest `updated`) of its documents, and ask whether the user wants a review pass (AI agents evolve quickly; a re-review usually sharpens the compilation). If they decline, **end here**. If they accept, reset the tracking file and run the review in parallel, per `Instructions.md`.

3. **New language.** If the directory does not exist:
   - Read `common/ConstituentElements.md` — the research backbone.
   - **Create `<language>/research-tracking.md`**: one table row per research point of the backbone (19 rows, one per section), with columns **Status | Research Point | Details | Start | End** (status empty = not researched, `✅` done, `❌` error, `⚠️` warning; Details holds the problem description, empty if OK).

4. **Parallel research.** Launch **`agents`** sub-agents working simultaneously. Sections are independent — no sequential order. Each sub-agent, in a loop:
   - Claims the next unclaimed row by writing its **Start** timestamp.
   - Researches that point for **the target language** using authoritative sources.
   - Writes the section's document into the correct category folder (one file per section, tracking front matter included), per the mapping in `Instructions.md`.
   - Updates its row **immediately**: Status, Details (only if there was a problem), and **End** timestamp.
   - Repeats until no unclaimed rows remain.

5. **Validate and report.** When all rows are finished, run the final checklist from `Instructions.md`. Summarize: documents created/updated, `✅`/`⚠️`/`❌` counts, and remind the user that the next step is the independent **StandardBuilder** repository. If `❌` rows remain, tell the user to re-run `/standard_research <language>` to retry only those points.

## Hard rules

- This repository is **documentation only**: produce Markdown reference documents, never application code or tooling.
- **Never research a point without claiming its row first**, and never batch tracking updates at the end — the tracking file must reflect reality at any instant.
- Document alternatives side by side; never impose a single convention when the ecosystem has several recognized ones.
- Never copy or move `common/ConstituentElements.md` — it is the guide and it stays put.
