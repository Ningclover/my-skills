---
name: init_work
description: This skill should be used when the user says "init work", "init_work", "initialize work", "start work", "update my repos", or "pull my wikis". It runs git pull on personal knowledge repos (my_wirecell_wiki, my-skills, my_debug_log, my_optic_wiki) wherever they exist on the current server.
version: 1.0.0
---

# init_work Skill

Pull the latest changes from personal knowledge/wiki repos on the current server.

## Target Repositories

- `my_wirecell_wiki`
- `my-skills`
- `my_debug_log`
- `my_optic_wiki`

## Procedure

### Step 1 — Find the base directory

The base directory varies by server — discover it dynamically. Search these locations in order:

1. `$HOME`
2. Common data dirs: `/exp/dune/data/users/$USER`, `/dune/data/users/$USER`, `/data/users/$USER`
3. Any base dir the user mentioned earlier in this session

Use Bash to search:

```bash
for repo in my_wirecell_wiki my-skills my_debug_log my_optic_wiki; do
    found=$(find "$HOME" -maxdepth 3 -type d -name "$repo" 2>/dev/null | head -1)
    [ -n "$found" ] && echo "$repo: $found"
done

# Also check common data dirs
for base in /exp/dune/data/users/$USER /dune/data/users/$USER /data/users/$USER; do
    [ -d "$base" ] && echo "Checking $base:" && ls "$base" 2>/dev/null
done
```

If none of the repos are found after searching, **ask the user** for the base directory before proceeding.

### Step 2 — Pull only the repos that exist

For each repo found, run `git pull` inside it. Skip repos that are not found — missing repos on this server are expected and normal.

```bash
cd /path/to/found/repo && git pull
```

Run pulls sequentially so output stays readable.

### Step 3 — Report results

Give a brief summary:
- Which repos were pulled (up-to-date or new commits)
- Which repos were **not found** on this server (list them)

## Rules

- Never hardcode paths — always discover dynamically using `$HOME` and `$USER`.
- Do not commit, push, or modify files — pull only.
- If `git pull` fails (merge conflict, no remote, etc.), show the error and continue with the remaining repos.
- Do not warn or error about repos that are simply absent on this server — just skip them.
