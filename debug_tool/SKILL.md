---
name: debug_tool
description: General-purpose C++/simulation debugging skill. Use when a Geant4 simulation, physics framework, or any C++ program produces wrong output, silent failures, crashes, or unexpected zero/missing results. Works for edep-sim, WireCell, Geant4 applications, and similar simulation frameworks.
---

# General-Purpose Simulation Debugger

## Philosophy

**Output-driven only.** Never conclude a bug is found from code reading alone. Every hypothesis must be confirmed by actual program output — add debug prints, recompile, rerun, read the output, then decide.

**Confirm at the source, not at the symptom.** When something is missing (no hits, zero count, empty collection), instrument the place where the data is *created*, not just where it is *used*. The creation site tells you whether the data ever existed; the use site only tells you it wasn't there when you looked.

---

## Phase 1 — Gather and Reproduce

Ask the user to provide:
1. The full **running command**
2. **Environment** — how the code was built, relevant env vars, which machine
3. **Output they saw** — full terminal output or log

### Step 1a — Reproduce yourself

Run the command via Bash. Save output to a temp file (`/nfs/data/1/xning/tmp/` on wcgpu1 — `/tmp/` is full). Confirm:
- **Reproduced** — same failure seen. Show the key lines.
- **Not reproduced** — different result. Ask user to clarify before proceeding.

Do not proceed past Phase 1 until you can reproduce the failure, or the user explicitly says to skip.

### Step 1b — Classify the failure

Identify which category the failure belongs to:

| Category | Signs |
|----------|-------|
| **Silent wrong output** | Program exits 0, but results are empty/zero/missing |
| **Crash / segfault** | SIGSEGV, abort(), core dump, stack trace |
| **Command not found / config error** | Bad macro command, missing file, wrong argument |
| **Stale binary** | Code was changed, `make` was run, but the right library was not installed |

---

## Phase 2 — Check the Environment First

Before touching source code, rule out environment issues:

### 2a — Stale binary / library

The most common silent failure in C++ simulation frameworks. **The binary loads a shared library from an install location, not from the build directory.** Running `make` compiles into `build/`, but the executable still uses the old `install/lib/`.

**Check:**
```bash
ls -la build/lib/libfoo.so install/lib/libfoo.so  # timestamps should match after install
```

**Fix:** Always run `make install` (or equivalent) after rebuilding, not just `make`.

**Signs of stale binary:**
- Debug prints you added never appear in output
- Behavior unchanged after source edit
- Build timestamp in `build/` is newer than `install/`

### 2b — Command not found in config/macro

If a macro or config command is silently ignored or gives `COMMAND NOT FOUND`:
- Check if the command is available **after** initialization. Many physics process commands only exist post-`/run/initialize`.
- Check the exact command path — framework defaults may reset settings before your macro runs.

### 2c — Wrong working directory

Relative paths in config files resolve from the working directory at runtime, not the script location. If a file is not found, check where the binary is being run from.

---

## Phase 3 — Instrument at the Source

Once environment issues are ruled out, add debug prints to confirm data is created and flows correctly.

### Rule: instrument where data is BORN, not where it's used

If hits/counts/results are zero:
1. Find where those objects are **created or populated** (the producer).
2. Add a print immediately after creation confirming count and values.
3. Then trace forward to where they are consumed.

Never start by printing at the consumer — that only confirms absence, not cause.

### Rule: print data values, not just milestones

**Wrong (milestone only):**
```cpp
std::cout << "[DEBUG] reached ProcessHits\n";
std::cout << "[DEBUG] energyDeposit=" << energyDeposit << "\n";
```

**Right (confirm gate conditions):**
```cpp
std::cout << "[DEBUG] ProcessHits: particle=" << theTrack->GetDefinition()->GetParticleName()
          << " energyDeposit=" << energyDeposit/CLHEP::eV << " eV"
          << " totalEnergy=" << theTrack->GetTotalEnergy()/CLHEP::eV << " eV"
          << " stepStatus=" << theStep->GetPostStepPoint()->GetStepStatus()
          << std::endl;
```

The second form reveals *why* a gate condition fails, not just that it did.

### Rule: throttle prints to avoid log flood

For per-photon/per-step prints, use a static counter:
```cpp
static int dbgCount = 0;
if (dbgCount < 20) {
    ++dbgCount;
    std::cout << "[DEBUG] ..." << std::endl;
}
```

Without throttling, millions of lines make the log unreadable.

### Rule: tag all debug prints

Prefix every debug print with `[DEBUG]` or a unique tag. This makes them easy to grep and easy to remove cleanly:
```bash
grep "\[DEBUG\]" /nfs/data/1/xning/tmp/run.log | head -30
```

---

## Phase 4 — Debug Loop

Repeat until the root cause is confirmed by output:

1. **Identify** — from current output, which gate/condition is preventing the data from flowing?
2. **Hypothesize** — what value or state is wrong, and why?
3. **Instrument** — add print(s) that will confirm or refute the hypothesis. Print the actual values involved.
4. **Rebuild and install** — `make -j4 install` (or equivalent). Verify the install timestamp updated.
5. **Run** — save output to `/nfs/data/1/xning/tmp/`.
6. **Read** — grep the log for your debug tags. Decide: confirmed, refuted, or need more data.
7. **Return to 1.**

Never skip step 4 (install). Never skip step 6 (read the output before deciding).

### Tracing a null pointer / failed dynamic_cast

When a cast returns null and causes a crash or silent skip:
```cpp
SomeType* ptr = dynamic_cast<SomeType*>(base);
// DEBUG: confirm cast succeeded
std::cout << "[DEBUG] ptr=" << ptr << " (null=failed cast)" << std::endl;
if (ptr) {
    ptr->DoThing();
    std::cout << "[DEBUG] DoThing() called OK" << std::endl;
}
```

Common cause: the object's runtime type doesn't match the cast target — wrong registration order, or the object was constructed in a context where the derived type wasn't set yet.

### Tracing silent kills / filters

When objects (photons, tracks, hits) are being silently discarded:
1. Find the filter/gate function (stacking action, selection cut, guard condition).
2. Print the decision variable at the gate:
```cpp
std::cout << "[DEBUG] classifying particle=" << particle->GetParticleName()
          << " killFlag=" << fKillFlag << std::endl;
```
3. Confirm whether the gate is open or closed, and what value controls it.

### Tracing missing initialization

When a flag or value is stuck at its default (e.g., `fKillOpticalPhotons` stays `true`):
1. Find where the flag is *supposed* to be changed.
2. Confirm the code that changes it is actually executed:
```cpp
std::cout << "[DEBUG] SetKillOpticalPhotons(false) called" << std::endl;
fKillOpticalPhotons = false;
```
3. If the print never appears, the initialization path was never reached — investigate why (wrong order, wrong thread, null pointer preventing the call).

---

## Phase 5 — Crash Debugging (Segfault / Abort)

### Deferred heap corruption

glibc heap corruption crashes are **deferred**. The corrupting write happens in function A, but `abort()` fires at the next `malloc`/`free` in function B (which is innocent).

Signs:
- Crash location drifts when you add/remove debug prints
- ASAN reports `heap-buffer-overflow` at a different line than the segfault

Strategy:
- The function that ran just **before** the crash is the likely culprit, not the one that crashed.
- Add data-value prints (not just milestones) after each algorithmic operation in the suspected function.
- Look for: subnormal floats (`1e-320` range), zeros where nonzero is expected, pointer-sized integers — signs of uninitialized or overwritten memory.

### STL algorithm undefined behavior

`std::sort`, `std::unique`, `std::lower_bound` require comparators satisfying **strict weak ordering**. A broken comparator causes silent data corruption — the sort returns normally but the container holds garbage.

**Broken pattern:**
```cpp
// WRONG — falls through when x values are equal
if (a.x < b.x) return true;
// missing: else if (a.x > b.x) return false;
else if (a.y < b.y) return true;
```

When you see garbage values in a container that was filled with valid data, always ask: did any sort/unique run on it with a custom comparator?

### Heap probes ≠ data integrity

`malloc(1)/free(1)` heap probes validate glibc's *allocator metadata*, not the *contents* of live buffers. If probes pass but the crash persists, the corruption is inside an allocated object — switch from probes to data-value dumps.

---

## Phase 6 — Confirm the Fix

Before declaring the bug fixed:

1. Remove all debug prints (or comment them out with `// DEBUG:` prefix so they're easy to restore).
2. Rebuild and install cleanly.
3. Run the full original command and confirm the failure is gone.
4. Verify the correct output appears (not just absence of crash).

---

## Rules (Always Follow)

1. **Output-driven only.** Base all conclusions on actual program output. Code reading alone is not enough.

2. **Instrument at the source.** When data is missing, find where it's created and confirm it was created correctly, before chasing the consumer.

3. **Print values, not milestones.** Always print the actual data (flags, counts, energies, coordinates) that control the behavior you're investigating.

4. **Throttle per-step prints.** Use a static counter to limit output to the first N occurrences.

5. **Make install, not just make.** Always verify the installed library/binary is updated after a rebuild. Check timestamps.

6. **Confirm before concluding.** When you think you found the root cause, add prints that directly confirm it — then recompile, rerun, and verify the output matches the hypothesis.

7. **Comment, never delete.** When modifying code during debugging, comment out the old version. Never delete it — it preserves context and makes reverting easy.

8. **Deferred = upstream.** If crash location drifts when you add/remove probes, the real corruption site is earlier than where the crash fires.

9. **Save logs to `/nfs/data/1/xning/tmp/`.** The `/tmp/` directory is full on wcgpu1.
