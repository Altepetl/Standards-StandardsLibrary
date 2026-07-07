---
title: standard_research command
status: draft
version: 0.4.0
created: 2026-07-05
updated: 2026-07-06
description: Research and compile the programming standards for a specific language into StandardLibrary, in parallel with N sub-agents
argument-hint: <language> [agents] [points]
---

# /standard_research $ARGUMENTS

You are the **coordinator agent** of **StandardLibrary**. Your job is to orchestrate the research and compilation of the programming standards for a language, using parallel sub-agents. This command only researches and writes standard documents — it never assesses or reports on the accuracy of existing ones; that is the job of `/standard_validation`.

## Parameters

- **`language`** (1st argument, required): the language to research. If missing, ask the user and stop until you have it. Normalize to lowercase (e.g., `Java` → `java`).
- **`agents`** (2nd argument, optional): how many sub-agents research **simultaneously**. **Default: 3.** Must be a positive integer; if invalid, fall back to 3.
- **`points`** (3rd argument, optional): a comma-separated list of section numbers from `common/ConstituentElements.md` (e.g., `1,3,8,11`). Restricts the run to only these points instead of all 26 — use it to redo research for specific sections without a full re-run. Example: `/standard_research rust 2 1,3,8,11` redoes points 1, 3, 8, and 11 for Rust using 2 parallel sub-agents. If omitted, all applicable points are considered.

## Before anything else

Read the file `Instructions.md` at the repository root. It contains the complete workflow, including the specification of the research tracking file and the sub-agent coordination protocol. Do not proceed from memory — read it first, every time.

## Execution summary (details live in Instructions.md)

1. **Resume check.** If `<language>/research-tracking.md` exists with unfinished rows (empty status, or claimed rows with Start but no End), a previous run was interrupted. Report progress (done/pending counts), clear stale claims (Start and Agent), and **resume**: dispatch sub-agents only for the unfinished points (including `❌` rows; ask about redoing `⚠️` rows). This is the whole point of the tracking file — the process can be cut and restarted where it left off.

2. **Existing research check.** If the language directory exists and its tracking file is fully finished, behavior depends on whether `<language>/validation-tracking.md` exists:
   - **No `validation-tracking.md`:**
     - No `points` given → report the **creation date** (oldest `created`) and **last review date** (newest `updated`), mention that a `points` parameter can be passed to redo only specific sections, then ask whether to redo the **full** research. Decline → **end here, do nothing else**. Accept → reset the whole tracking file and redo all 26 points, updating existing documents.
     - `points` given → skip the question; reset just those rows and redo only those points, updating existing documents.
   - **`validation-tracking.md` exists:** the target points are those in `points` if given, otherwise every point in `validation-tracking.md` not marked `✅`. For each, read that point's `Details` cell in `validation-tracking.md` first and use it to drive the redo — this is a feedback-driven fix, not a blind re-research. Once a point's document is rewritten, set its `validation-tracking.md` row to `✅` and clear `Details`.

3. **New language.** If the directory does not exist:
   - Read `common/ConstituentElements.md` — the research backbone.
   - **Create `<language>/research-tracking.md`**: one row per research point of the backbone (26 rows, one per section, or only the rows in `points` if given), with columns **Status | Agent | Research Point | Details | Start | End** (status empty = not researched, `✅` done, `❌` error, `⚠️` warning; Agent = name of the sub-agent that did the work; Details holds the problem description, empty if OK).

4. **Parallel research.** Launch **`agents`** sub-agents working simultaneously. Points are independent — no sequential order. Each sub-agent, in a loop:
   - Claims the next unclaimed row by writing its **Start** timestamp and its **own Agent name**.
   - Researches that point for **the target language** using authoritative sources (incorporating any validation `Details` fed to it, per Step 2 above).
   - Writes the section's document into the correct category folder (one file per section, tracking front matter included), per the mapping in `Instructions.md`.
   - Updates its row **immediately**: Status, Agent, Details (only if there was a problem), and **End** timestamp.
   - Repeats until no unclaimed rows remain.

5. **Finish and report.** When all rows are finished, run the completion checklist from `Instructions.md`. Summarize: documents created/updated, `✅`/`⚠️`/`❌` counts, and remind the user that the next step is the independent **StandardBuilder** repository. If `❌` rows remain, tell the user to re-run `/standard_research <language> [agents] [points]` to retry only those points.

## Hard rules

- This repository is **documentation only**: produce Markdown reference documents, never application code or tooling.
- **Never research a point without claiming its row first** (Start + Agent name), and never batch tracking updates at the end — the tracking file must reflect reality at any instant.
- The **Agent** column must never be left empty on a finished row — every sub-agent must record its own name/label there.
- Document alternatives side by side; never impose a single convention when the ecosystem has several recognized ones.
- Never copy or move `common/ConstituentElements.md` — it is the guide and it stays put.
- This command never performs content validation or writes inconsistency findings — use `/standard_validation` for that.
