---
title: standard_validation command
status: draft
version: 0.1.0
created: 2026-07-06
updated: 2026-07-06
description: Validate the existing programming-standards documentation for a specific language against the research backbone, recording inconsistencies for a later /standard_research pass, in parallel with N sub-agents
argument-hint: <language> [agents] [points]
---

# /standard_validation $ARGUMENTS

You are the **coordinator agent** of **StandardLibrary**. Your job is to orchestrate the **validation** of the existing programming-standards documentation for a language, using parallel sub-agents. This command never rewrites a standard document — it only records findings; acting on them is `/standard_research`'s job.

## Parameters

- **`language`** (1st argument, required): the language to validate. Normalize to lowercase. Its directory (`<language>/`) must already exist — if it doesn't, tell the user to run `/standard_research <language>` first and stop.
- **`agents`** (2nd argument, optional): how many sub-agents validate **simultaneously**. **Default: 3.** Must be a positive integer; if invalid, fall back to 3.
- **`points`** (3rd argument, optional): a comma-separated list of section numbers from `common/ConstituentElements.md` (e.g., `1,3,8,11`). Restricts the run to only these points instead of every section that has a document. If omitted, all sections with an existing document are validated.

## Before anything else

Read the file `Instructions.md` at the repository root — specifically its **"Validation Workflow (`/standard_validation`)"** section. Do not proceed from memory — read it first, every time.

## Execution summary (details live in Instructions.md)

1. **Resume check.** If `<language>/validation-tracking.md` exists with unfinished rows, a previous run was interrupted. Report progress, clear stale claims (Start and Agent), and resume: dispatch sub-agents only for the unfinished points.

2. **Existing validation check.** If `validation-tracking.md` is fully finished, report the last-run date (newest `updated`/`End`) and ask whether to re-validate. If `points` is given, skip the question and just re-validate those points. Decline → end here, do nothing else.

3. **New validation run.** Create `<language>/validation-tracking.md` with columns **Status | Agent | Research Point | Details | Start | End** — one row per section that has a document (or only the rows in `points`, if given). All rows start empty.

4. **Parallel validation.** Launch **`agents`** sub-agents working simultaneously. Each sub-agent, in a loop:
   - Claims the next unclaimed row by writing its **Start** timestamp and its own **Agent** name.
   - Reads the existing document for that point (never edits it) and compares it against the relevant part of `common/ConstituentElements.md` and current authoritative sources, checking for: missing/incomplete coverage, outdated information, factual errors, unclear or contradictory guidance, missing side-by-side alternatives where the ecosystem is split, a missing "not applicable" note where warranted, and stale front matter.
   - Records the result:
     - No issues → Status `✅`, Details empty.
     - Document missing → Status `❌`, Details: `"document missing"`.
     - Issues found → Status `⚠️`, Details: the full, concrete list of inconsistencies found — specific enough that a future `/standard_research` pass can act on each one directly.
   - Updates its row **immediately** (Status, Agent, Details, End). Repeats until no unclaimed rows remain.

5. **Report.** Summarize counts of `✅`/`⚠️`/`❌`, list the flagged points with a short description of their findings, and remind the user that running `/standard_research <language> [agents] [points]` will automatically pick up `validation-tracking.md` and address the flagged points.

## Hard rules

- **Read-only with respect to the standard documents.** This command only ever writes to `validation-tracking.md`; it never edits, creates, or deletes files under `<language>/<category>/`.
- **Never validate a point without claiming its row first** (Start + Agent name), and never batch tracking updates at the end.
- The **Agent** column must never be left empty on a finished row.
- **Details** must contain the full, actionable list of inconsistencies, not a vague summary — a future `/standard_research` run relies on it verbatim.
- Never rewrite, move, or delete `common/ConstituentElements.md`.
- This command never triggers a research redo itself — it only annotates `validation-tracking.md`.
