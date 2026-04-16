---
name: add_skill
description: Create a new Claude Code skill from a description or existing behavior, write it to ~/my-skills/<skill_name>/SKILL.md, and symlink it into ~/.claude/skills/ so it is immediately available.
---

# Add Skill

## Purpose

Turn a described behavior or workflow into a reusable Claude Code skill, install it, and confirm it is ready to use.

---

## Step 1 — Get the skill name and description

If the user has not provided them, ask:
- **Skill name**: short, lowercase with underscores (e.g. `add_to_wiki`, `bee_url_check`). This becomes the directory name and the slash command name.
- **What the skill should do**: a plain-language description of the behavior, including any defaults, options, and rules.

If the user says "make this a skill" referring to something already discussed in the conversation, extract the name and behavior from context.

---

## Step 2 — Write the SKILL.md

Create the file at:
```
~/my-skills/<skill_name>/SKILL.md
```

Use this structure:

```markdown
---
name: <skill_name>
description: <one-line description shown in the skill list and used for trigger matching>
---

# <Skill Title>

## Purpose
What the skill does and when to use it.

## Step 1 — ...
## Step 2 — ...
...

## Rules (Always Follow)
- Rule 1
- Rule 2
```

Guidelines for writing the skill:
- The `name:` in frontmatter must match the directory name exactly.
- The `description:` should be specific enough that Claude knows when to auto-trigger it.
- Steps should be concrete and actionable — written as instructions Claude will follow, not as documentation.
- Include defaults, fallbacks, and edge cases explicitly.
- Keep it concise. Bullet points over prose.

---

## Step 3 — Symlink into ~/.claude/skills/

```bash
ln -s ~/my-skills/<skill_name> ~/.claude/skills/<skill_name>
```

Check first that the symlink does not already exist. If it does, skip this step and note it.

---

## Step 4 — Confirm

Tell the user:
- The skill file path
- The symlink path
- The slash command to invoke it (e.g. `/<skill_name>`)
- That a session restart may be needed for the skill to appear in the skill list
