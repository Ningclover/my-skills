---
name: add_to_wiki
description: Scan the current conversation for confirmed code understanding and explanations, then write or update pages in the target wiki. Automatically selects between ~/my_wirecell_wiki and ~/my_optic_wiki based on conversation content, or accepts an explicit path argument.
---

# Add to Wiki

## Purpose

Extract **confirmed code understanding** from the current conversation and persist it into a structured wiki. Only include knowledge that is certain — algorithm explanations, data structure descriptions, control flow, API semantics. Exclude: debug logs, debug print statements, speculative bug hypotheses, guesses, temporary investigation notes, and anything prefixed with "I think" or "likely".

---

## Step 0 — Select the target wiki

**Before anything else**, determine which wiki the content belongs to. The two available wikis are:

| Wiki | Path | Domain |
|------|------|--------|
| `my_wirecell_wiki` | `~/my_wirecell_wiki` | WireCell toolkit — LArTPC signal processing, clustering, imaging, noise filtering, ROI, simulation |
| `my_optic_wiki` | `~/my_optic_wiki` | Optics / photon detection — optical simulation, photon sensors, light yield, PDS, DUNE-FD photon systems |

**Selection rules (in order):**

1. If the user provided an explicit wiki path as an argument, use that — skip the rest of this step.
2. Scan the conversation for domain signals:
   - WireCell signals: `WireCell`, `wirecell`, `clus`, `sigproc`, `img`, `sst`, `cluster`, `wire plane`, `TPC`, `ROI`, `deconvolution`, `noise filtering`, `blob`, `facade`
   - Optic signals: `optic`, `photon`, `optical`, `scintillation`, `PDS`, `SiPM`, `light yield`, `dune-fd optics`, `photon detector`
3. Pick the wiki whose signals dominate the conversation.
4. If the conversation contains signals from **both** domains, or if the domain is ambiguous, **ask the user** to confirm which wiki before proceeding.
5. State the chosen wiki to the user (one sentence) before continuing.

---

## Step 1 — Resolve the wiki path

- Use the wiki selected in Step 0.
- Check that the directory exists (`ls <wiki_path>/wiki/`).
  - If it does not exist: ask the user to provide a valid wiki path, or do nothing if they decline.
- Read `<wiki_path>/CLAUDE.md` to understand the wiki's schema, page conventions, and index structure before writing anything.

---

## Step 2 — Read the wiki index

Read `<wiki_path>/index.md` to understand:
- What pages already exist
- How the index is organized (sections and sub-sections)
- Where new content should be placed

---

## Step 3 — Extract knowledge from the conversation

Scan the entire conversation for **confirmed, certain** code understanding. Collect items in these categories:

### What to include
- **Function/method explanations** — what a function does, its inputs, outputs, side effects
- **Algorithm descriptions** — step-by-step logic of a non-trivial algorithm
- **Data structure semantics** — what fields mean, ownership, lifetime
- **Control flow rationale** — why a swap, branch, or ordering choice exists
- **API/interface semantics** — what parameters do, return value meaning, constraints
- **Cross-file relationships** — how functions in different files call each other

### What to exclude (strictly)
- Debug print statements added during investigation
- Hypotheses about bug causes that were not confirmed by output
- Guesses, speculation, "likely", "probably", "I think" statements
- Temporary investigation notes (crash logs, backtrace analysis, heap corruption theories)
- Step-by-step debugging procedures
- Anything that only makes sense in the context of the current debugging session

---

## Step 4 — Determine target pages

For each piece of extracted knowledge, decide:

1. **Does a relevant page already exist?** Check the index. If yes, plan to update it.
2. **Is there enough new content to justify a new page?** If the content belongs to a new topic not covered by any existing page, plan to create one.
3. **What section/sub-section in the index does it belong to?** (e.g., `clus → Clustering`, `clus → Pattern Recognition`, `sigproc → Noise Filtering`, etc.)

---

## Step 5 — Write or update pages

For each target page:

### If updating an existing page
- Read the existing page first.
- Add new content in the appropriate section. Do not duplicate what is already there.
- Update the `updated:` date in the YAML frontmatter.

### If creating a new page
- Use the wiki's filename convention (Title Case with spaces, e.g., `Cluster Merging.md`).
- Use the suggested page structure from `CLAUDE.md`:
  ```
  # Page Title
  Brief one-line description.

  ## Purpose
  ## Method
  ## Inputs / Outputs
  ## Configuration (if applicable)
  ## See also
  ## Sources
  ```
- Include appropriate YAML frontmatter (`tags`, `updated`).

### Content rules
- Be concise. Prefer bullet points over prose.
- Use backtick paths for code references: `` `src/clus/Facade_Cluster.cxx:2227` ``
- Use `[[Page Title]]` wikilinks for cross-references.
- Do not invent facts — only write what was explicitly explained or confirmed in the conversation.

---

## Step 6 — Update the index

- Add any new pages to `<wiki_path>/index.md` under the correct section.
- Each index entry: one line, format `- [[Page Title]] — one-line description`.
- Update the `updated:` date at the top of index.md.

---

## Step 7 — Create a session record

Create a session record page at `<wiki_path>/wiki/source-session-<YYYY-MM-DD>-<slug>.md`. This is the citable anchor for the knowledge written in steps 5–6. Use a short slug describing the investigation topic (e.g., `cluster-merging`, `roi-debug`).

```markdown
---
tags: [source]
type: conversation
date: <YYYY-MM-DD>
context: <one-line description of what was being investigated>
files_touched: [<list of source files read or modified during the session>]
updated: <YYYY-MM-DD>
---

# source-session-<YYYY-MM-DD>-<slug>

<Brief description of the session: what problem was being debugged, what experiment/module.>

## Confirmed findings
- <each piece of confirmed knowledge extracted> — see [[Page Title]]

## Context
<What code was being read, what crash or behavior was being investigated.>
```

Then, for each page written in step 5, ensure it has a `## Sources` section that includes `- [[source-session-<YYYY-MM-DD>-<slug>]]`.

Also add the new session record to `<wiki_path>/index.md` under the **Sources** section.

---

## Step 8 — Append to the log

Append to `<wiki_path>/log.md`:

```
## [YYYY-MM-DD] ingest | <short description>
- <bullet: what was added or updated>
- <bullet: pages created>
- session record: source-session-<YYYY-MM-DD>-<slug>.md
```

---

## Step 9 — Git push

After all writes are complete, push the wiki to remote:

```bash
cd <wiki_path> && git add -A && git commit -m "ingest: <short description> [YYYY-MM-DD]" && git push
```

Report the result (success or any error) to the user.

---

## Rules (Always Follow)

1. **Certain knowledge only.** If you are not sure something is correct based on the conversation, do not write it.
2. **No debug artifacts.** Never write debug print formats, GDB backtraces, or crash investigation notes into the wiki.
3. **No speculation.** Do not write "the crash is likely caused by..." or any unconfirmed hypothesis.
4. **Read before write.** Always read `CLAUDE.md` and `index.md` before writing any page. Always read an existing page before updating it.
5. **Minimal footprint.** Do not rewrite pages that don't need changing. Only update sections that have new content.
6. **Ask if unsure about placement.** If the index structure doesn't clearly indicate where content belongs, ask the user before creating a new section.
