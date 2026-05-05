---
description: One-time onboarding skill that connects a new team member's agent to the Ametyst ecosystem. Configures Slack + Notion MCP servers, clones general-protocols, symlinks rules/skills/agents into the user's .claude/, merges the Ametyst CLAUDE.md snippet into their CLAUDE.md, and (optionally) builds self-context. Use whenever a new person joins Ametyst, or says "onboard", "set up Ametyst", "connect to Ametyst", or "settings onboarding".
allowed-tools: Read, Write, Edit, Glob, Bash(gh *), Bash(mkdir *), Bash(ln *), Bash(node *), Bash(npm *), AskUserQuestion, mcp__slack__slack_post_message
---

## settings-onboarding

One-time skill. Connects a new agent to the Ametyst ecosystem by:
- Receiving Slack + Notion tokens from the admin
- Installing and configuring both MCP servers in `~/.claude/settings.json` (user-level, so they work across all the user's projects)
- Cloning `ametyst-business/general-protocols` into `~/Ametyst/protocols/general-protocols`
- Creating symlinks from the cloned repo into the user's `.claude/{rules,skills,agents}/`
- Merging the `general-protocols/CLAUDE.md` snippet (between `AMETYST:BLOCK:START/END` markers) into the user's `.claude/CLAUDE.md`
- Verifying Slack with a `#general` test post
- Optionally building self-context

After this skill, the user runs `/settings-become-teamspace-ic` for each area they have access to.

**Prerequisite:** the admin has shared a Slack Bot Token and a Notion Integration Secret, granted GitHub access to `ametyst-business`, and invited the bot to `#general`.

---

### Step 1 — Gather parameters

Use `AskUserQuestion`:

1. **Name** — first name (used throughout the setup)
2. **Working directory** — absolute path of the repo where the agent operates (default: current working directory). This is the repo whose `.claude/` will be patched.

---

### Step 2 — Receive tokens from admin

Tell the user:

```
Your admin has already created the Slack and Notion apps for you.
You should have received two tokens:

1. **Slack Bot Token** — starts with `xoxb-`
2. **Notion Integration Secret** — starts with `ntn_`

If you haven't received them yet, ask the admin to send them to you securely.

Do you have both tokens?
```

Use `AskUserQuestion` to wait for the user to confirm. If the user only has the Slack token (Notion not granted yet), allow them to skip Notion — the Notion MCP can be added later by `/settings-become-teamspace-ic` when they join their first Notion-using area.

---

### Step 3 — Prerequisites & MCP server configuration (human step)

Tell the user:

```
Now let's make sure you have the right tools installed and configure the MCP servers.

## 1. Check prerequisites

The MCP servers require Node.js (v18+) and npm. Let's check if you have them.

Open a terminal and run:

  node --version
  npm --version

### If both commands return a version number → skip to step 2.

### If Node.js / npm are NOT installed:

**macOS:**
  - If you have Homebrew: brew install node
  - If you don't have Homebrew:
    1. Install Homebrew first: /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    2. Then: brew install node
  - Alternative (no Homebrew): download the macOS installer from https://nodejs.org

**Windows:**
  - Download the Windows installer (.msi) from https://nodejs.org
  - Run it and follow the prompts (make sure "Add to PATH" is checked)
  - Restart your terminal after installation

**Linux:**
  - Ubuntu/Debian: sudo apt update && sudo apt install nodejs npm
  - Fedora: sudo dnf install nodejs npm
  - Or use nvm: curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    then: nvm install --lts

After installing, verify with: node --version && npm --version

## 2. Install the MCP server packages

Run this in your terminal:

  npm install -g @modelcontextprotocol/server-slack @notionhq/notion-mcp-server

This installs both MCP servers globally so they're available to Claude Code.

## 3. Configure settings.json

Open (or create) the file ~/.claude/settings.json and add the following
under the "mcpServers" key. If the file already exists, merge this into it.

{
  "mcpServers": {
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-paste-your-token-here",
        "SLACK_TEAM_ID": "T0A9Z6HAXDX"
      }
    },
    "notionApi": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "NOTION_TOKEN": "ntn_paste-your-token-here"
      }
    }
  }
}

Replace the placeholder tokens with the ones you received from the admin.
The SLACK_TEAM_ID is already set to the Ametyst workspace.

If the user skipped Notion in Step 2, omit the "notionApi" entry — they
can add it later via /settings-become-teamspace-ic.

After saving the file, restart Claude Code so the MCP servers load.
Tell me when you're done.
```

Use `AskUserQuestion` to wait for confirmation.

---

### Step 4 — Verify Slack connection

Post a test message to `#general` (channel ID `C0A9L8KDFEW`):

```
:wave: *New agent connecting — <Name>*
Onboarding in progress.
```

If the post fails with `not_in_channel`, ask the user to have the admin run `/invite @<bot>` in `#general`, then retry. Do NOT proceed until Slack is verified.

(Notion is not verified here because the General teamspace has no Notion databases. Notion access will be verified by `/settings-become-teamspace-ic` when the user joins their first Notion-using area.)

---

### Step 5 — Clone general-protocols

```bash
mkdir -p ~/Ametyst/protocols
gh repo clone ametyst-business/general-protocols ~/Ametyst/protocols/general-protocols 2>/dev/null || git -C ~/Ametyst/protocols/general-protocols pull --rebase --quiet
```

If clone fails (no access), tell the user to ask the admin to confirm GitHub access to `ametyst-business`.

---

### Step 6 — Symlink rules / skills / agents into user's `.claude/`

For each of `rules`, `skills`, `agents` in `~/Ametyst/protocols/general-protocols/`, iterate over its top-level entries (files or folders) and create a symlink in the corresponding subdir of `<working-dir>/.claude/`:

```bash
SRC="$HOME/Ametyst/protocols/general-protocols"
DST="<working-dir>/.claude"

for sub in rules skills agents; do
  mkdir -p "$DST/$sub"
  for entry in "$SRC/$sub"/*; do
    [ -e "$entry" ] || continue
    name=$(basename "$entry")
    [ "$name" = ".gitkeep" ] && continue
    target="$DST/$sub/$name"
    if [ -e "$target" ] && [ ! -L "$target" ]; then
      echo "SKIP: $target exists and is not a symlink"
      continue
    fi
    ln -sf "$entry" "$target"
  done
done
```

Report which symlinks were created.

---

### Step 7 — Merge CLAUDE.md snippet into user's CLAUDE.md

Read `~/Ametyst/protocols/general-protocols/CLAUDE.md`. Extract the block between `<!-- AMETYST:BLOCK:START -->` and `<!-- AMETYST:BLOCK:END -->`.

Then append this block as a **new section** at the bottom of `<working-dir>/.claude/CLAUDE.md` (create the file if it does not exist).

Idempotency: if `<!-- AMETYST:BLOCK:START -->` is already in the user's CLAUDE.md, replace the existing block (between markers) with the new one — do not duplicate.

---

### Step 8 — Optional self-context

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

### Step 9 — Welcome message

Show the user the message before posting, ask for confirmation, then post to `#general` (`C0A9L8KDFEW`):

```
:wave: *New agent online — <Name>*
General teamspace connected.
Run /settings-become-teamspace-ic to join an area.
```

---

### Step 10 — Recap

Tell the user:

```
Onboarding complete.

- Slack MCP configured + verified
- Notion MCP configured (verification deferred to first area join)
- general-protocols cloned to ~/Ametyst/protocols/general-protocols
- Symlinked into <working-dir>/.claude/{rules,skills,agents}/
- CLAUDE.md snippet merged
- Self-context: <created | skipped>

Next steps:
- For each area you have access to, run /settings-become-teamspace-ic
- Pull latest protocols anytime with: git -C ~/Ametyst/protocols/general-protocols pull
```
