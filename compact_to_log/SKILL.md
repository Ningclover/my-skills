---
name: compact_to_log
description: Compact the current conversation into a concise, structured log file readable by a future agent in a new session. Use when the user says "compact this conversation", "save to log", or "compact to log".
---

# Compact to Log

## Purpose
Summarize the current conversation into a self-contained markdown log file that
a future Claude agent can read in a new session to understand what was done,
what decisions were made, what was guessed and whether it turned out to be true,
and what the current state of the work is.

## Step 1 — Determine the output path

- Default log directory: `agent_log/` under the current working directory.
- If the user specifies a different folder (as an argument or in the conversation), use that instead.
- Create the directory if it does not exist (`mkdir -p`).
- Filename format: `YYYY-MM-DD-<slug>.md` where `<slug>` is a 2-4 word kebab-case
  description of the session topic (e.g. `run-nf-command`, `vd-nf-debug`).
- Use today's date from the `currentDate` context.

## Step 2 — Write the log file

The log must be a single markdown file with these sections:

```markdown
# Session: <title>
Date: YYYY-MM-DD

## Goal
One paragraph: what the user wanted to accomplish.

## Files Created
For each new file: repo-relative path, purpose, key design decisions, important parameters.

## Files Modified
For each modified file: repo-relative path, what changed and why.

## Key Design Decisions
Bullet list of non-obvious choices made during the session, with rationale.

## Hypotheses & Findings
For each guess, hypothesis, or assumption made during the session:
- **Hypothesis**: what was guessed or assumed
- **Status**: confirmed true / confirmed false / unresolved
- **Evidence**: what confirmed or disproved it (brief)

## Test Results
What was tested, how, and what the outcome was. Include errors encountered
and how they were resolved.

## Current State
What is working, what is not, what remains to be done (if anything).

## Usage Examples
Concrete shell commands or code snippets showing how to use what was built.
```

### Content rules
- Be concise — bullet points over prose wherever possible.
- Use repo-relative paths (e.g. `wcp-porting-img/pdhd/wct-nf.jsonnet`), not absolute paths.
- Include exact command-line flags and their meanings for any CLI tools built.
- Record error messages and their resolutions verbatim when relevant.
- In "Hypotheses & Findings": include even unresolved guesses — a future agent needs
  to know what is still uncertain, not just what was confirmed.
- Omit exploratory dead-ends only if they reveal no constraint a future agent would need.

## Step 3 — Confirm to user

Tell the user the full path of the file written.

## Rules (Always Follow)
- Always use today's date (from `currentDate` context) in the filename and file header.
- Use repo-relative paths in the log body — never hardcode absolute paths.
- The log must be self-contained: a future agent reading only this file should have
  enough context to continue the work or reproduce the results.
- Include hypotheses with their confirmation status — "confirmed true/false/unresolved".
- Keep each section tight. A good log is 1-3 pages, not 10.
