---
description: One-time onboarding skill that connects a new team member's agent to the Ametyst ecosystem. Configures Slack + Notion MCP servers (global in ~/.claude/settings.json or project-only in <working-dir>/.claude/settings.local.json — user's choice), clones general-protocols, symlinks rules/skills/agents into the user's .claude/, merges the Ametyst CLAUDE.md snippet into their CLAUDE.md, and (optionally) builds self-context. Use whenever a new person joins Ametyst, or says "onboard", "set up Ametyst", "connect to Ametyst", or "settings onboarding".
allowed-tools: Read, Write, Edit, Glob, Bash(gh *), Bash(mkdir *), Bash(ln *), Bash(node *), Bash(npm *), Bash(git *), Bash(grep *), Bash(printf *), AskUserQuestion, mcp__slack__slack_post_message
---

## settings-onboarding

One-time skill. Connects a new agent to the Ametyst ecosystem by:
- Receiving Slack + Notion tokens from the admin
- Installing and configuring both MCP servers — the user picks the scope:
  - **Global** in `~/.claude/settings.json` (default; works across all the user's projects)
  - **Project-only** in `<working-dir>/.claude/settings.local.json` (gitignored, so tokens stay out of git)
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
3. **MCP install scope** — where to install the Slack + Notion MCP servers:
   - **Global** (`~/.claude/settings.json`) — recommended. MCP servers are available across **all** the user's projects. The file lives in the user's home directory and is never committed to any repo.
   - **Project-only** (`<working-dir>/.claude/settings.local.json`) — MCP servers only load when Claude Code runs inside this repo. Use this if the user wants to keep tokens scoped to this single project (e.g. they juggle multiple workspaces with different Slack/Notion accounts).

   **Important:** for the project-only option we use `settings.local.json` (NOT `settings.json`). `settings.local.json` is **convention-only** for being local — Claude Code does NOT auto-add it to `.gitignore`. So if the user picks project scope, the skill MUST verify (and patch) the repo's `.gitignore` in Step 3 before writing tokens. Never write MCP tokens into a project's `settings.json` — that file is committed to git and would leak secrets.

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

**Step 3.0 — Gitignore guard (project scope only).**

If the user picked **project scope** in Step 1, the agent MUST run this check BEFORE giving the user the configuration instructions below:

```bash
WORKDIR="<working-dir>"
PATTERN=".claude/settings.local.json"

# Does the repo's gitignore already cover settings.local.json?
if git -C "$WORKDIR" check-ignore -q "$PATTERN" 2>/dev/null; then
  # It's ignored — but check WHY. If only by a global gitignore (e.g. ~/.config/git/ignore),
  # that protection won't survive on other machines. Confirm the repo's own .gitignore covers it.
  if ! grep -qE "(^|/)\.claude/settings\.local\.json$|(^|/)\*\*/\.claude/settings\.local\.json$|(^|/)settings\.local\.json$" "$WORKDIR/.gitignore" 2>/dev/null; then
    echo "PATCH: settings.local.json is ignored only by a global gitignore — adding entry to repo .gitignore for portability."
    printf '\n# Claude Code local settings (contains MCP tokens — never commit)\n.claude/settings.local.json\n' >> "$WORKDIR/.gitignore"
  fi
else
  # Not ignored at all — must patch the repo's .gitignore.
  echo "PATCH: adding .claude/settings.local.json to repo .gitignore."
  printf '\n# Claude Code local settings (contains MCP tokens — never commit)\n.claude/settings.local.json\n' >> "$WORKDIR/.gitignore"
fi

# Verify
git -C "$WORKDIR" check-ignore -v "$PATTERN" || {
  echo "ERROR: settings.local.json is still not ignored. Stop. Do not write tokens."
  exit 1
}
```

Tell the user (only when patched):

```
I added `.claude/settings.local.json` to your repo's .gitignore so your
Slack/Notion tokens stay out of git. Commit that .gitignore change before
pushing — otherwise the protection only exists locally.
```

**If the agent CAN'T verify the gitignore protection (check fails), STOP** — do not proceed to write tokens. Surface the error and ask the user to add the entry manually, then retry.

For **global scope**, skip Step 3.0 entirely — `~/.claude/settings.json` lives outside any repo, so there's nothing to gitignore.

---

Then tell the user:

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

> **Heads-up — known deprecation warning (Slack).** During install npm will
> print `npm warn deprecated @modelcontextprotocol/server-slack@2025.4.25:
> Package no longer supported.` This is **expected and safe to ignore for now**
> — the package still works and is what the rest of this skill (and
> `/settings-become-teamspace-ic`) targets. The Anthropic-maintained Slack MCP
> was archived; Ametyst is tracking actively-maintained alternatives (e.g.
> `slack-mcp-server` by korotovsky) and will swap the install command here
> when the migration is ready. If you're onboarding a new contributor and the
> warning shows up, reassure them: no action needed.

## 3. Configure the settings file

Open (or create) the settings file at the path chosen in Step 1, and add
the following under the "mcpServers" key. If the file already exists, merge
this into it (don't overwrite existing keys like "permissions").

- **Global scope:** `~/.claude/settings.json`
- **Project scope:** `<working-dir>/.claude/settings.local.json`
  (NOT `settings.json` — `settings.local.json` is gitignored so your
  tokens never get committed.)

The JSON schema is **identical** in both files — only the path differs:

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

The Ametyst protocols live under `~/Ametyst/protocols/` — an umbrella directory containing one cloned repo per area. This step adds the first one (`general-protocols/`); `/settings-become-teamspace-ic` later adds `<area>-protocols/` per area joined. Skills, rules, agents, and guides are then symlinked from these clones into the user's `.claude/`, so a `git pull` in any clone updates everyone at once.

```bash
mkdir -p ~/Ametyst/protocols
gh repo clone ametyst-business/general-protocols ~/Ametyst/protocols/general-protocols 2>/dev/null \
  || git -C ~/Ametyst/protocols/general-protocols pull --rebase --quiet
```

If clone fails (no access), tell the user to ask the admin to confirm GitHub access to `ametyst-business`.

---

### Step 6 — Symlink rules / skills / agents / guides into user's `.claude/`

**Invariants** — these MUST hold after this step:
- `.claude/rules/`, `.claude/skills/`, `.claude/agents/`, `.claude/guides/` are **real directories**, never top-level symlinks. A top-level symlink (e.g. `.claude/skills` → `~/Ametyst/protocols/general-protocols/skills/`) breaks IDE display (VS Code/Cursor don't show nested symlinks) and pollutes the shared protocols repo when `/settings-become-teamspace-ic` later adds area entries inside them.
- Each entry inside is an individual symlink pointing directly at the source — never symlink-inside-symlink (which causes circular `ln -s` behavior).
- Guides go under `.claude/guides/<source>/` (here `general/`) — never flat in `.claude/guides/`. This keeps provenance clear once multiple area-protocols are connected.
- Symlink targets have no trailing slash. Some shells expand globs with one (`for entry in dir/*` may yield `dir/foo/`); `ln -s target/ link` is silently treated as a directory ref by macOS, losing symlink metadata so VS Code/Cursor stop showing the arrow → on the link. We strip with `${entry%/}`.
- **`rules/communication-routing.md` is per-user state, NOT a protocol file.** It must be **copied** (real file), never symlinked. Reason: `/settings-become-teamspace-ic` mutates this file every time the user joins a new area; if it were a symlink, the Edit tool would follow it and write into the shared `~/Ametyst/protocols/general-protocols/rules/communication-routing.md` clone — dirtying upstream and causing `git pull --rebase` conflicts on the next protocol sync. Any other file flagged as per-user state (extend `PER_USER_RULES` below) must follow the same copy-not-symlink rule.

```bash
SRC="$HOME/Ametyst/protocols/general-protocols"
DST="<working-dir>/.claude"

# CONVERT: if a previous (buggy) install left .claude/<sub> as a top-level symlink,
# convert it into a real directory by re-symlinking its contents individually.
# Idempotent — no-op on a fresh install where these directories don't exist yet.
for sub in rules skills agents guides; do
  if [ -L "$DST/$sub" ]; then
    echo "CONVERT: $DST/$sub is a top-level symlink — converting to a real directory."
    real=$(readlink "$DST/$sub")
    real="${real%/}"
    rm "$DST/$sub"
    mkdir -p "$DST/$sub"
    for entry in "$real"/*; do
      [ -e "$entry" ] || continue
      name=$(basename "$entry")
      [ "$name" = ".gitkeep" ] && continue
      ln -sf "${entry%/}" "$DST/$sub/$name"
    done
  fi
done

# Files inside rules/ that are per-user state — copy, never symlink.
# Extend this list if more user-mutable rule files are introduced upstream.
PER_USER_RULES=("communication-routing.md")

is_per_user_rule() {
  local name="$1"
  for n in "${PER_USER_RULES[@]}"; do [ "$n" = "$name" ] && return 0; done
  return 1
}

# SYMLINK rules / skills / agents — flat under .claude/<sub>/.
# Exception: per-user files inside rules/ are COPIED (real file), not symlinked.
for sub in rules skills agents; do
  mkdir -p "$DST/$sub"
  for entry in "$SRC/$sub"/*; do
    [ -e "$entry" ] || continue
    name=$(basename "$entry")
    [ "$name" = ".gitkeep" ] && continue
    target="$DST/$sub/$name"

    # Per-user state in rules/: copy on first install, leave alone if real file exists.
    if [ "$sub" = "rules" ] && is_per_user_rule "$name"; then
      if [ -L "$target" ]; then
        echo "CONVERT: $target was a symlink (legacy install) — replacing with a copy so per-user edits stay local."
        rm "$target"
        cp "$entry" "$target"
      elif [ ! -e "$target" ]; then
        echo "COPY: $target (per-user state, copied from $entry)."
        cp "$entry" "$target"
      else
        echo "KEEP: $target exists as a real file — preserving user's per-user state."
      fi
      continue
    fi

    if [ -e "$target" ] && [ ! -L "$target" ]; then
      echo "SKIP: $target exists and is not a symlink"
      continue
    fi
    [ -L "$target" ] && rm "$target"
    ln -sf "${entry%/}" "$target"
  done
done

# SYMLINK guides — under .claude/guides/general/ (area-isolated).
mkdir -p "$DST/guides/general"
guides_count=0
for entry in "$SRC/guides"/*; do
  [ -e "$entry" ] || continue
  name=$(basename "$entry")
  [ "$name" = ".gitkeep" ] && continue
  target="$DST/guides/general/$name"
  if [ -e "$target" ] && [ ! -L "$target" ]; then
    echo "SKIP: $target exists and is not a symlink"
    continue
  fi
  [ -L "$target" ] && rm "$target"
  ln -sf "${entry%/}" "$target"
  guides_count=$((guides_count+1))
done

if [ "$guides_count" -eq 0 ]; then
  echo "NOTE: $DST/guides/general/ is empty — general-protocols has no guides yet (expected)."
fi
```

Report which symlinks were created. If any `CONVERT:` lines fired, also tell the user to inspect `~/Ametyst/protocols/general-protocols/` for stray entries that may have been written into the shared repo by the previous buggy install (and ask the admin to clean them up).

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

- Slack MCP configured + verified (scope: <global | project-only>, file: <path used>)
- Notion MCP configured (verification deferred to first area join)
- general-protocols cloned to ~/Ametyst/protocols/general-protocols
- Symlinked into <working-dir>/.claude/{rules,skills,agents,guides/general}/
- CLAUDE.md snippet merged
- Self-context: <created | skipped>

How the symlinks work (one-time explainer):
A symlink is a file that points at another file. Each entry under
.claude/{rules,skills,agents,guides}/ is a symlink into
~/Ametyst/protocols/<area>-protocols/. So when the admin pushes a
protocol update upstream, you pick it up by `git pull`-ing the cloned
repo — no copying, no manual sync. Don't edit the files inside .claude/
directly unless you intend to PR a change to the protocols repo: edits
write back to the clone.

Next steps:
- For each area you have access to, run /settings-become-teamspace-ic
- Pull latest protocols anytime with:
    git -C ~/Ametyst/protocols/general-protocols pull
```
