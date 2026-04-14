---
name: debug-wirecell
description: Debug segmentation faults and mid-run crashes in the WireCell Toolkit processing chain (noise filtering, signal processing, imaging, clustering). Use when a WireCell job crashes, segfaults, or stops unexpectedly mid-chain.
---

# WireCell Segfault Debugger

## Context

The WireCell toolkit runs a chain per anode: **noise filtering → signal processing → imaging → clustering**. The chain can be run end-to-end or in partial steps. A crash can happen in the driving script, the jsonnet config, or deep inside C++ source code. Each case has a different approach.

---

## Phase 1 — Gather Basic Information

Ask the user to provide:

1. The full **running command**
2. A description of the **environment** (e.g., how the toolkit was set up/compiled, relevant env vars)
3. The **output they saw** — paste the full terminal output, log, or error message

Once the user provides this, extract the following from the output:

- **Which step in the chain crashed** — look for the last successfully completed step or the step name that appears in the traceback/log just before the crash. The chain order is: noise filtering → signal processing → imaging → clustering.
- **Which anode was being processed** — look for anode identifiers (e.g., `anode0`, `anode:0`, `APA 0`, or similar patterns) in the output near the crash point.

Summarize your findings and confirm with the user before proceeding to Phase 2. If the output does not contain enough information to determine the crash location or anode, ask the user to re-run with increased verbosity or provide additional logs.

---

## Phase 2 — Identify the Crash Layer

### Case A: Error in the Running Script or Jsonnet Config

Debug directly at the config level. Useful tools:
- `plot_jsonnet` — to inspect and validate jsonnet configurations

Resolve the config error before proceeding.

### Case B: Error Inside C++ Source Code

Proceed to Phase 3.

---

## Phase 3 — Reproduce and Isolate (C++ Crash)

1. **Ask the user** if they want to check the output from the previous step in the chain (i.e., the input file to the failing step) to rule out a corrupted input.

2. **Ask the user** if they want a simplified version of the running script that isolates just the failing step and target anode.

   - **If yes:** Generate a simplified script focused only on the problematic step and anode. Ask the user to run it and confirm it reproduces the crash.
     - If it does not reproduce: discuss with the user why, and iterate until reproduction is confirmed.
   - **If no:** Keep the original script, but suggest running in **debug mode**.

---

## Phase 4 — Debug Loop (Iterate Until Bug Is Located)

Repeat this loop until the exact crash location is found:

1. Based on current output, identify **which file and which line** the crash appears to originate from.
2. Read the relevant lines and trace correlating functions in other files.
3. **Add print/output debug statements** around the suspected area.
4. Recompile the toolkit.
5. Rerun the script and examine the new output.
6. Return to step 1 with the updated information.

**Communicate with the user throughout the loop.** If the user directs you to check a specific location, add debug output there as well.

---

## Phase 5 — Confirm the Bug

Once a likely root cause is identified:

1. Add print statements around the suspected code to confirm the hypothesis.
2. Recompile and rerun.
3. Only conclude the bug is found after the output **confirms** it — do not rely on code reading alone.

---

## Rules (Always Follow)

1. **Output-driven only.** Always base bug localization on actual debug output, not just static code analysis or deduction. If output is insufficient, add more print statements, recompile, and rerun.

2. **Confirm before concluding.** When you think you found the bug, add print statements around it, recompile, and run again to confirm before declaring success.

3. **Comment, never delete.** When modifying code during debugging, always comment out the old version — do not delete it. This preserves context and makes it easy to revert.
