---
name: debug-wirecell
description: Debug segmentation faults and mid-run crashes in the WireCell Toolkit processing chain (noise filtering, signal processing, imaging, clustering). Use when a WireCell job crashes, segfaults, or stops unexpectedly mid-chain.
---

# WireCell Segfault Debugger

## Context

The WireCell toolkit runs a chain per anode: **noise filtering → signal processing → imaging → clustering**. The chain can be run end-to-end or in partial steps. A crash can happen in the driving script, the jsonnet config, or deep inside C++ source code. Each case has a different approach.

---

## Phase 1 — Gather Basic Information and Reproduce

Ask the user to provide:

1. The full **running command**
2. A description of the **environment** (e.g., how the toolkit was set up/compiled, relevant env vars)
3. The **output they saw** — paste the full terminal output, log, or error message

### Step 1a — Reproduce the crash yourself

Once the user provides the command and environment, **attempt to reproduce the crash using the Bash tool** before doing any analysis.

Environment setup on wcgpu1:
- Always prepend `eval "$(direnv export bash 2>/dev/null)"` to every Bash command that needs WireCell tools. The CWD must be inside `/nfs/data/1/xning/wirecell-working/` for this to work.
- Use `/nfs/data/1/xning/tmp/` for all log/temp files — `/tmp/` is full on wcgpu1.
- Check the memory file at `/nfs/data/1/xning/.claude/projects/-nfs-data-1-xning-wirecell-working/memory/feedback_env_setup.md` for the latest env guidance if unsure.

Run the user's command via Bash and capture the output. Confirm one of:
- **Reproduced** — the same crash/error is seen. State clearly that you reproduced it and show the key lines.
- **Not reproduced** — the run succeeded or failed differently. Report what happened and ask the user to clarify the environment or input differences before proceeding.

Do not proceed to Phase 2 until the crash is reproduced or the user explicitly asks to skip reproduction.

### Step 1b — Identify crash location from output

From the output (user-provided or your own reproduction), extract:

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

### Isolating the crashing visitor inside a `cm_pipeline` (MABC loop)

If the crash happens inside the `MultiAlgBlobClustering` pipeline loop (the MABC loop iterates over `cm_pipeline` visitors defined in the jsonnet config), use the **comment-from-end** technique to binary-search which visitor is responsible:

1. Comment out visitors from the **end** of `cm_pipeline` only — never from the middle, because each visitor depends on the previous one's output.
2. Rerun. If the crash disappears, the removed block contains the crashing visitor.
3. Restore one visitor at a time from the end until the crash reappears. The step that reintroduces the crash is the crashing visitor.

**Key warning — deferred heap crashes:** glibc heap corruption crashes are deferred. The corrupting write happens inside one visitor, but `abort()` / SIGSEGV fires at the next `malloc`/`free` call in the **following** visitor. This means:
- The visitor that appears last in the debug log before the crash is **not** the guilty one.
- The guilty visitor is the one **just before** the last logged visitor.
- To confirm: comment out the suspected visitor; the crash should then move one step earlier (firing after the visitor before it).

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

### Print data values, not just milestones

**Critical rule:** When adding debug prints, always print the **actual data values** (coordinates, sizes, indices, flags), not just milestone messages like "reached here" or "size=N".

Milestone prints tell you that code *executed*. They cannot reveal **data corruption**, where code runs normally but produces garbage values. For heap-corruption bugs, the corrupting write typically happens inside a function that returns normally — the crash fires later when those garbage values are used.

**Wrong (milestone only):**
```cpp
std::cerr << "before sort npoints=" << points.size() << "\n";
std::sort(points.begin(), points.end(), cmp);
std::cerr << "after sort npoints=" << points.size() << "\n";
```

**Right (dump actual values):**
```cpp
std::cerr << "before sort npoints=" << points.size() << "\n";
std::sort(points.begin(), points.end(), cmp);
std::cerr << "after sort npoints=" << points.size() << "\n";
for (size_t i = 0; i < points.size(); ++i) {
    const auto& p = points[i].first;
    std::cerr << "  [" << i << "] x=" << p.x()/cm << " y=" << p.y()/cm << " z=" << p.z()/cm << " cm\n";
}
```

Only the second form would have revealed garbage coordinates like `x=2.37e-322 y=0 z=4.94e-324` caused by undefined behavior in `std::sort`.

### When heap probes pass but the crash persists

`malloc(1)/free(1)` heap probes validate **glibc's allocator metadata** (chunk size fields, freelist links, `prev_inuse` bits). They do **not** validate the contents of live allocations. If probes pass everywhere but the crash still occurs:

- The corruption is **inside an allocated buffer** — glibc sees the chunk as "in use" and does not check its contents.
- Switch from control-flow probes to **data-value dumps**: print the actual contents of suspect vectors, arrays, and structs at key points.
- Look for: subnormal floats (`1e-320` range), zero values where nonzero is expected, pointer-sized integers, repeated patterns — all signs of uninitialized or corrupted memory.

### STL algorithm UB: a class of silent data corruption

`std::sort`, `std::unique`, `std::lower_bound`, and similar STL algorithms require their comparators to satisfy **strict weak ordering** (irreflexivity, asymmetry, transitivity). Passing a broken comparator is **undefined behavior** — the algorithm may:

- Return normally with corrupted element values
- Write garbage into adjacent memory
- Infinite-loop or access out-of-bounds indices

The corruption happens silently inside the algorithm. No heap probe fires. The function returns normally. The crash occurs later when the corrupted data is used.

**What to check when you see garbage values in a container that was filled with valid data:**
1. Did any `std::sort` / `std::unique` run on it with a custom comparator?
2. Does the comparator correctly implement strict weak ordering? In particular: when `a.x == b.x`, does it return `false` (not fall through to compare `y`)? The broken pattern is:
   ```cpp
   // BROKEN — falls through when x values are equal
   if (x() < rhs.x()) return true;
   // else if (x() > rhs.x()) return false;  ← missing!
   else if (y() < rhs.y()) return true;
   ```
3. Read the comparator's full definition — including any `operator<` it calls transitively (e.g., on `geo_point_t` / `D3Vector`).

Note: `D3Vector::operator<` in WireCell (`toolkit/util/inc/WireCellUtil/D3Vector.h`) had exactly this bug — the `else if (x() > rhs.x()) return false` and `else if (y() > rhs.y()) return false` guards were commented out, making every sort that used it UB when any two points shared an x or y coordinate.

---

## Phase 5 — Confirm the Bug

Once a likely root cause is identified:

1. Add print statements around the suspected code to confirm the hypothesis.
2. Recompile and rerun.
3. Only conclude the bug is found after the output **confirms** it — do not rely on code reading alone.

---

## Rules (Always Follow)

1. **Output-driven only.** Always base bug localization on actual debug output, not just static code analysis or deduction. If output is insufficient, add more print statements, recompile, and rerun.

2. **Print data, not just milestones.** After any algorithmic operation on a container (sort, erase, unique, transform), print the actual element values — not just the size. Data corruption is invisible to milestone prints.

3. **Confirm before concluding.** When you think you found the bug, add print statements around it, recompile, and run again to confirm before declaring success.

4. **Comment, never delete.** When modifying code during debugging, always comment out the old version — do not delete it. This preserves context and makes it easy to revert.

5. **Heap probes ≠ data integrity.** Passing heap probes means glibc's allocator bookkeeping is intact. It says nothing about the values inside live allocations. When probes pass but crashes persist, switch to dumping data values.

6. **When crash location drifts between runs, suspect deferred corruption.** If adding or removing a probe changes where the crash fires, the actual write site is earlier than where glibc detects it. Focus on data values at the last function that ran successfully before the crash started drifting.
