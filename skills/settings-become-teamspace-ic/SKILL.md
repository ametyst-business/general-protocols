---
description: Connect the agent to a new Ametyst teamspace as IC (or member). Clones the area's protocols repo, symlinks rules/skills/agents, appends the area block to CLAUDE.md, registers the area's wiki, updates communication-routing.md, and verifies Notion + Slack access. Use whenever you've been given access to an area and need to wire your agent up. Triggered by "become ic", "join teamspace", "connect strategy", "connect governance", "connect product", "connect gtm", or similar.
allowed-tools: Read, Write, Edit, Glob, Bash(gh *), Bash(mkdir *), Bash(ln *), Bash(git *), AskUserQuestion, mcp__notionApi__API-query-data-source, mcp__slack__slack_post_message
---

## settings-become-teamspace-ic

Connect the agent to one Ametyst teamspace. Run after `/settings-onboarding`. Repeatable for each area.

What it does:
- Clones `ametyst-business/<area>-protocols` into `~/Ametyst/protocols/<area>-protocols`
- Symlinks `rules/`, `skills/`, `agents/` entries into the user's `.claude/`
- Appends the area's CLAUDE.md snippet (between `AMETYST:AREA:<area>:START/END`) into the user's `.claude/CLAUDE.md`
- Adds a row to the "Connected wikis" table inside `AMETYST:BLOCK`
- Adds the area's row to `.claude/rules/communication-routing.md`
- Adds a `git pull` line to "Morning startup" inside `AMETYST:BLOCK`
- Verifies Notion access (test query) and Slack access (test post in the area channel)

**Prerequisite:** `/settings-onboarding` already run; admin has granted access to the area's Notion teamspace, Slack channel, and GitHub repo.

---

### Step 1 — Pick area and check access

Use `AskUserQuestion`: "Which area? (strategy, governance, product, gtm)"

Verify the user has GitHub access:

```bash
gh repo view ametyst-business/<area>-protocols --json name 2>&1
```

If access is denied, tell the user to ask the admin to add them to the repo and stop here. Do NOT proceed.

Check if already connected: if `~/Ametyst/protocols/<area>-protocols/.git` exists, ask whether to re-connect (refresh symlinks + re-merge CLAUDE.md block) or abort.

---

### Step 2 — Clone the area's protocols repo

```bash
mkdir -p ~/Ametyst/protocols
gh repo clone ametyst-business/<area>-protocols ~/Ametyst/protocols/<area>-protocols 2>/dev/null \
  || git -C ~/Ametyst/protocols/<area>-protocols pull --rebase --quiet
```

---

### Step 3 — Symlink rules / skills / agents

For each of `rules`, `skills`, `agents` in `~/Ametyst/protocols/<area>-protocols/`, symlink each top-level entry into `<working-dir>/.claude/<sub>/`:

```bash
SRC="$HOME/Ametyst/protocols/<area>-protocols"
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

### Step 4 — Merge area CLAUDE.md snippet

Read `~/Ametyst/protocols/<area>-protocols/CLAUDE.md`. Extract the block between `<!-- AMETYST:AREA:<area>:START -->` and `<!-- AMETYST:AREA:<area>:END -->`.

Append the block to `<working-dir>/.claude/CLAUDE.md` after the existing `<!-- AMETYST:BLOCK:END -->` marker.

Idempotency: if `<!-- AMETYST:AREA:<area>:START -->` is already present, replace the existing block (between markers) with the new one.

---

### Step 5 — Add row to "Connected wikis" table

Inside the user's `.claude/CLAUDE.md`, find the "Connected wikis" table (inside `AMETYST:BLOCK`) and append a row:

```
| <area> | github | ametyst-business/<area>-wiki |
```

If the row already exists for `<area>`, do nothing.

---

### Step 6 — Update communication-routing.md

Read `<working-dir>/.claude/rules/communication-routing.md`. Add a row to the routing table for the area, mapping the area's Slack channel and Notion databases to `.claude/rules/<area>-teamspace-communication/`.

Read the area's `notion-protocol.md` (just symlinked) to extract the database list, and add a row like:

```
| #<area>, <area> Notion databases | `.claude/rules/<area>-teamspace-communication/` |
```

If the row already exists, do nothing.

---

### Step 7 — Add `git pull` line to Morning startup

Inside `AMETYST:BLOCK` in user's CLAUDE.md, find the "Morning startup" section and append:

```
- `git -C ~/Ametyst/protocols/<area>-protocols pull --rebase --quiet`
```

If the line already exists, do nothing.

---

### Step 8 — Verify Notion + Slack

**Notion:** read the just-symlinked `<working-dir>/.claude/rules/<area>-teamspace-communication/notion-protocol.md`, find the Tasks database `data_source_id`, and run a test query with `Team` filter on `<Area>`. If it fails, tell the user the integration is missing the connection on the area backend page — admin must add it.

**Slack:** post a test message to the area channel:

```
:wave: *<Name> joining as IC of <Area>*
Symlinks installed, protocols loaded.
```

Channel IDs (lookup from area's `slack-protocol.md`):
- strategy → `C0AHN1ZB18T`
- governance → `C0AKQDJEFFH`
- product → (read from protocol)
- gtm → (read from protocol)

If post fails (`not_in_channel`), ask the admin to invite the bot.

---

### Step 9 — Recap

Tell the user:

```
Connected to <Area> teamspace.

- Cloned: ~/Ametyst/protocols/<area>-protocols
- Symlinks: rules/skills/agents installed
- CLAUDE.md: AMETYST:AREA:<area> block appended
- Connected wikis: row added (<area> → ametyst-business/<area>-wiki)
- Communication routing: row added
- Morning startup: git pull line added
- Notion: <verified | failed>
- Slack: <verified | failed>

Pull latest anytime with: git -C ~/Ametyst/protocols/<area>-protocols pull
```
