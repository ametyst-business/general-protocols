---
description: One-time onboarding skill that connects a new team member's agent to the Ametyst ecosystem. Clones general-protocols, symlinks rules/skills/agents into the user's .claude/, merges the Ametyst CLAUDE.md snippet into their CLAUDE.md, configures Slack MCP, and (optionally) builds self-context. Use whenever a new person joins Ametyst, or says "onboard", "set up Ametyst", "connect to Ametyst", or "settings onboarding".
allowed-tools: Read, Write, Edit, Glob, Bash(gh *), Bash(mkdir *), Bash(ln *), Bash(node *), Bash(npm *), AskUserQuestion, mcp__slack__slack_post_message
---

## settings-onboarding

One-time skill. Connects a new agent to the Ametyst ecosystem by:
- Cloning `ametyst-business/general-protocols` into `~/Ametyst/protocols/general-protocols`
- Creating symlinks from the cloned repo into the user's `.claude/{rules,skills,agents}/`
- Merging the `general-protocols/CLAUDE.md` snippet (between `AMETYST:BLOCK:START/END` markers) into the user's `.claude/CLAUDE.md`
- Configuring Slack MCP and verifying with a `#general` test post
- Optionally building self-context

After this skill, the user runs `/settings-become-teamspace-ic` for each area they have access to.

**Prerequisite:** admin has shared a Slack Bot Token, granted GitHub access to `ametyst-business`, and invited the bot to `#general`.

---

### Step 1 — Gather parameters

Use `AskUserQuestion`:

1. **Name** — first name
2. **Working directory** — absolute path of the repo where the agent operates (default: current working directory). This is the repo whose `.claude/` will be patched.

---

### Step 2 — Slack MCP setup

Tell the user:

```
Your admin should have sent you a Slack Bot Token (starts with `xoxb-`).
If you haven't received it, ask them.

I'll add the Slack MCP server to <working-dir>/.claude/settings.json. If
you already have a settings.json, I'll merge — your other MCP servers stay.
```

Wait for the token. Then patch `<working-dir>/.claude/settings.json` adding under `mcpServers`:

```json
{
  "slack": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-slack"],
    "env": {
      "SLACK_BOT_TOKEN": "<token>",
      "SLACK_TEAM_ID": "T0A9Z6HAXDX"
    }
  }
}
```

If `settings.json` does not exist, create it with this content. If it exists with other `mcpServers`, merge — do not overwrite.

Tell the user to restart Claude Code so the MCP loads, then confirm.

---

### Step 3 — Verify Slack

Post a test message to `#general` (channel ID `C0A9L8KDFEW`):

```
:wave: *New agent connecting — <Name>*
Onboarding in progress.
```

If the post fails with `not_in_channel`, ask the user to have the admin run `/invite @<bot>` in `#general`, then retry. Do NOT proceed until Slack is verified.

---

### Step 4 — Clone general-protocols

```bash
mkdir -p ~/Ametyst/protocols
gh repo clone ametyst-business/general-protocols ~/Ametyst/protocols/general-protocols 2>/dev/null || git -C ~/Ametyst/protocols/general-protocols pull --rebase --quiet
```

If clone fails (no access), tell the user to ask the admin to confirm GitHub access to `ametyst-business`.

---

### Step 5 — Symlink rules / skills / agents into user's `.claude/`

For each of `rules`, `skills`, `agents` in `~/Ametyst/protocols/general-protocols/`, iterate over its top-level entries (files or folders) and create a symlink in the corresponding subdir of `<working-dir>/.claude/`:

```bash
SRC="$HOME/Ametyst/protocols/general-protocols"
DST="<working-dir>/.claude"

for sub in rules skills agents; do
  mkdir -p "$DST/$sub"
  for entry in "$SRC/$sub"/*; do
    [ -e "$entry" ] || continue
    name=$(basename "$entry")
    target="$DST/$sub/$name"
    if [ -e "$target" ] && [ ! -L "$target" ]; then
      echo "SKIP: $target exists and is not a symlink"
      continue
    fi
    ln -sf "$entry" "$target"
  done
done
```

Skip `.gitkeep` files. Report which symlinks were created.

---

### Step 6 — Merge CLAUDE.md snippet into user's CLAUDE.md

Read `~/Ametyst/protocols/general-protocols/CLAUDE.md`. Extract the block between `<!-- AMETYST:BLOCK:START -->` and `<!-- AMETYST:BLOCK:END -->`.

Then append this block as a **new section** at the bottom of `<working-dir>/.claude/CLAUDE.md` (create the file if it does not exist).

Idempotency: if `<!-- AMETYST:BLOCK:START -->` is already in the user's CLAUDE.md, replace the existing block (between markers) with the new one — do not duplicate.

---

### Step 7 — Optional self-context

Use `AskUserQuestion`: "Do you want to build a self-context file now? (recommended on first onboarding; skip if you already have one in `context/self-context/`)"

**If yes:**

Ask in two rounds:

**Round 1 — Identity:**
1. Role at Ametyst, background, expertise
2. Mission — what drives them
3. Main area of focus right now

**Round 2 — Working style:**
4. Working preferences, habits, principles
5. What they dislike in AI-agent collaboration
6. Specific instructions for their agent

Write `<working-dir>/context/self-context/<Name>.md`:

```markdown
# <Name>

## Who they are
<answer 1>

## Mission
<answer 2>

## What they work on
<answer 3>

## How they work
<answer 4>

## What they dislike
<answer 5>

## How to work with them
<answer 6>
```

**If no:** skip — no file written.

---

### Step 8 — Welcome message

Show the user the message before posting, ask for confirmation, then post to `#general` (`C0A9L8KDFEW`):

```
:wave: *New agent online — <Name>*
General teamspace connected.
Run /settings-become-teamspace-ic to join an area.
```

---

### Step 9 — Recap

Tell the user:

```
Onboarding complete.

- Slack MCP configured + verified
- general-protocols cloned to ~/Ametyst/protocols/general-protocols
- Symlinked into <working-dir>/.claude/{rules,skills,agents}/
- CLAUDE.md snippet merged
- Self-context: <created | skipped>

Next steps:
- For each area you have access to, run /settings-become-teamspace-ic
- Pull latest protocols anytime with: git -C ~/Ametyst/protocols/general-protocols pull
```
