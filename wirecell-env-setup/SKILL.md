---
name: wirecell-env-setup
description: Load the WireCell direnv environment for Bash tool commands on wcgpu1 server. Use when running any WireCell tool (wire-cell, woodpecker, etc.) via the Bash tool on wcgpu1.
---

# WireCell Environment Setup (wcgpu1)

## Purpose
Activate the direnv-managed environment for `/nfs/data/1/xning/wirecell-working/` on the **wcgpu1** server so that WireCell tools (wire-cell, woodpecker, etc.) are available in Bash tool calls. This sets PATH, LD_LIBRARY_PATH, WIRECELL_PATH, PYTHONPATH, ROOTSYS, and other required variables.

## Step 1 — Prepend env activation to every Bash command

Every Bash tool call that needs WireCell tools must start with:

```bash
eval "$(direnv export bash 2>/dev/null)"
```

This must be run from within `/nfs/data/1/xning/wirecell-working/` or any subdirectory — direnv only exports when CWD is inside the project root.

Example:
```bash
eval "$(direnv export bash 2>/dev/null)" && cd wcp-porting-img/pdvd && ./run_img_clus.sh ../../data/run039324/evt1/ --debug > /nfs/data/1/xning/tmp/run.log 2>&1
```

## Step 2 — Use the correct tmp directory

Always use `/nfs/data/1/xning/tmp/` for temporary and log files.
**Never use `/tmp/`** — it is on a full filesystem on wcgpu1.

## Step 3 — Verify the environment is active (optional)

```bash
eval "$(direnv export bash 2>/dev/null)" && which wire-cell && echo "env OK"
```

## Rules (Always Follow)

- This skill applies **only on the wcgpu1 server** at `/nfs/data/1/xning/wirecell-working/`.
- Prepend `eval "$(direnv export bash 2>/dev/null)"` to every Bash command that invokes WireCell tools.
- CWD must be inside `/nfs/data/1/xning/wirecell-working/` when calling `direnv export bash`.
- Use `/nfs/data/1/xning/tmp/` for all temp/log files — never `/tmp/`.
- The env activation does not persist between Bash tool calls — repeat it each time.
