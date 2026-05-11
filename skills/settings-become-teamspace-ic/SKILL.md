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

Use `AskUserQuestion`: "Which area? (strategy, product, gtm)"

> `governance` is intentionally **not** offered as an IC area here — it's an admin-only teamspace (legal, fundraising, cap table, IP). Governance access is granted directly by the admin via the admin onramp, not through this self-serve skill. If a contributor explicitly asks for governance, stop and tell them to coordinate with the admin.

Verify the user has GitHub access:

```bash
gh repo view ametyst-business/<area>-protocols --json name 2>&1
```

If access is denied, tell the user to ask the admin to add them to the repo and stop here. Do NOT proceed.

Check if already connected: if `~/Ametyst/protocols/<area>-protocols/.git` exists, ask whether to re-connect (refresh symlinks + re-merge CLAUDE.md block) or abort.

---

### Step 1.5 — Notion MCP fallback (only if missing)

The Notion MCP is normally installed by `/settings-onboarding`. If the user skipped it (or this is an upgrade from a pre-Notion state), install it now since every area-teamspace except General has Notion databases.

Check if `notionApi` is already configured. The user may have picked global or project scope during `/settings-onboarding`, so check **both** files:

```bash
WORKDIR="<working-dir>"
NOTION_PRESENT="missing"
if [ -f ~/.claude/settings.json ] && grep -q '"notionApi"' ~/.claude/settings.json 2>/dev/null; then
  NOTION_PRESENT="present (global ~/.claude/settings.json)"
elif [ -f "$WORKDIR/.claude/settings.local.json" ] && grep -q '"notionApi"' "$WORKDIR/.claude/settings.local.json" 2>/dev/null; then
  NOTION_PRESENT="present (project $WORKDIR/.claude/settings.local.json)"
fi
echo "$NOTION_PRESENT"
```

**If present:** skip this step entirely.

**If missing:** tell the user:

```
The Notion MCP server isn't configured yet. You need it to interact with
the <area> Notion databases.

Your admin should have sent you a Notion Integration Secret (starts with `ntn_`).
If you haven't received it yet, ask them.

## 1. Install the Notion MCP package

Run in your terminal:

  npm install -g @notionhq/notion-mcp-server

## 2. Configure your settings file

Open the same file you used during /settings-onboarding — that's
`~/.claude/settings.json` (global scope) or `<working-dir>/.claude/settings.local.json`
(project scope) — and add the following entry inside the "mcpServers" key
(alongside the existing "slack" entry):

    "notionApi": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "NOTION_TOKEN": "ntn_paste-your-token-here"
      }
    }

After saving, restart Claude Code so the new MCP server loads.
Tell me when you're done.
```

Use `AskUserQuestion` to wait for confirmation. Do NOT proceed until Notion MCP is configured.

---

### Step 2 — Clone the area's protocols repo

```bash
mkdir -p ~/Ametyst/protocols
gh repo clone ametyst-business/<area>-protocols ~/Ametyst/protocols/<area>-protocols 2>/dev/null \
  || git -C ~/Ametyst/protocols/<area>-protocols pull --rebase --quiet
```

---

### Step 3 — Symlink rules / skills / agents / guides

**Invariants** — these MUST hold after this step:
- `.claude/rules/`, `.claude/skills/`, `.claude/agents/`, `.claude/guides/` in the user's working dir are **real directories**, never top-level symlinks. If a previous (buggy) onboarding left one of these as a symlink into `~/Ametyst/protocols/general-protocols/<sub>/`, the area-specific entries would land **inside the shared general-protocols repo** — polluting it for everyone. The conversion block below detects and fixes this before any new symlink is created.
- Each entry inside is an individual symlink pointing directly at the source — never symlink-inside-symlink (which produces circular self-references like `admin-ingest-and-share-wiki/admin-ingest-and-share-wiki → ...`).
- Guides go under `.claude/guides/<area>/` (e.g. `strategy/`) — never flat. With multiple areas connected, flat guides mix file provenance and become unreadable.
- Symlink targets have no trailing slash. Some shells (zsh on macOS depending on globbing options) expand globs with one; `ln -s target/ link` is silently treated as a directory ref by macOS, dropping symlink metadata so VS Code/Cursor don't show the arrow → on the link. Strip with `${entry%/}`.

```bash
SRC="$HOME/Ametyst/protocols/<area>-protocols"
DST="<working-dir>/.claude"
AREA="<area>"

# CONVERT: if a previous (buggy) install left .claude/<sub> as a top-level symlink,
# convert it into a real directory by re-symlinking its contents individually.
# Idempotent — no-op if .claude/<sub> is already a real directory.
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

# SYMLINK rules / skills / agents — flat under .claude/<sub>/.
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
    [ -L "$target" ] && rm "$target"
    ln -sf "${entry%/}" "$target"
  done
done

# SYMLINK guides — under .claude/guides/<area>/ (area-isolated).
mkdir -p "$DST/guides/$AREA"
guides_count=0
for entry in "$SRC/guides"/*; do
  [ -e "$entry" ] || continue
  name=$(basename "$entry")
  [ "$name" = ".gitkeep" ] && continue
  target="$DST/guides/$AREA/$name"
  if [ -e "$target" ] && [ ! -L "$target" ]; then
    echo "SKIP: $target exists and is not a symlink"
    continue
  fi
  [ -L "$target" ] && rm "$target"
  ln -sf "${entry%/}" "$target"
  guides_count=$((guides_count+1))
done

if [ "$guides_count" -eq 0 ]; then
  echo "NOTE: $DST/guides/$AREA/ is empty — $AREA-protocols has no guides yet (expected for new areas)."
fi
```

Report which symlinks were created. If any `CONVERT:` lines fired, surface that to the user explicitly **and** tell them to ask the admin to inspect `~/Ametyst/protocols/general-protocols/{rules,skills,agents,guides}/` for stray entries that the previous buggy install may have written into the shared repo. Always report `guides/<area>/` if it ended up empty, so the user understands that's expected behavior, not a missing step.

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

**Migration guard — run FIRST.** `communication-routing.md` is per-user state (the routing table grows every time this skill runs). It MUST be a real file, never a symlink. Legacy onboarding installs (before this fix) symlinked it from `~/Ametyst/protocols/general-protocols/rules/communication-routing.md` — editing through that symlink would write into the shared protocols clone, dirtying upstream and triggering `git pull --rebase` conflicts on the next protocol sync. Convert before editing:

```bash
ROUTING="<working-dir>/.claude/rules/communication-routing.md"
if [ -L "$ROUTING" ]; then
  echo "CONVERT: $ROUTING is a symlink (legacy install) — replacing with a real-file copy so this edit stays local."
  real_target=$(readlink "$ROUTING")
  rm "$ROUTING"
  cp "$real_target" "$ROUTING"
fi
```

Then read `<working-dir>/.claude/rules/communication-routing.md`. Add a row to the routing table for the area, mapping the area's Slack channel and Notion databases to `.claude/rules/<area>-teamspace-communication/`.

Read the area's `notion-protocol.md` (just symlinked) to extract the database list, and add a row like:

```
| #<area>, <area> Notion databases | `.claude/rules/<area>-teamspace-communication/` |
```

If the row already exists, do nothing.

If the migration guard fired, surface that to the user: tell them to check `git status` inside `~/Ametyst/protocols/general-protocols/` for any unintended edits to `rules/communication-routing.md` from previous runs of this skill, and `git checkout -- rules/communication-routing.md` there to discard them before the next `git pull`.

---

### Step 7 — Add `git pull` line to Morning startup

Inside `AMETYST:BLOCK` in user's CLAUDE.md, find the "Morning startup" section and append:

```
- `git -C ~/Ametyst/protocols/<area>-protocols pull --rebase --quiet`
```

If the line already exists, do nothing.

---

### Step 8 — Verify Notion + Slack

#### Notion verification — two-stage with REST fallback

Read the just-symlinked `<working-dir>/.claude/rules/<area>-teamspace-communication/notion-protocol.md`, find the Tasks database `data_source_id`. Per-backend scoping replaced the legacy `Team` field — do NOT use a `Team` filter; an unfiltered query (or a trivial `Status` filter) is the correct test.

**Stage A — MCP query.** Call `mcp__notionApi__API-query-data-source` with just the `data_source_id` and `page_size: 1`. Three outcomes:

1. **Success with results from the right workspace** → mark Notion as verified. Done.
2. **Success but results look wrong** (e.g. the returned page titles clearly belong to a different workspace) → the loaded `notionApi` MCP is pointing at another Ametyst workspace. This happens when the user has multiple Notion integrations configured (multi-workspace setup). **Skip to Stage B.**
3. **Error response** (`unauthorized`, `object_not_found`, network failure, etc.) → could be either (a) the integration isn't connected to the area backend page in Notion, or (b) the wrong MCP is loaded. **Skip to Stage B** before concluding (a).

**Stage B — REST fallback.** Don't trust the MCP verdict yet — verify directly against the Notion REST API using the token the user pasted during onboarding.

```bash
WORKDIR="<working-dir>"
DATA_SOURCE_ID="<value from notion-protocol.md>"

# Locate the NOTION_TOKEN. Try project scope first (more specific), then global.
NOTION_TOKEN=""
SOURCE=""
if [ -f "$WORKDIR/.claude/settings.local.json" ] && grep -q '"NOTION_TOKEN"' "$WORKDIR/.claude/settings.local.json" 2>/dev/null; then
  NOTION_TOKEN=$(node -e "console.log((JSON.parse(require('fs').readFileSync('$WORKDIR/.claude/settings.local.json','utf8')).mcpServers?.notionApi?.env?.NOTION_TOKEN)||'')")
  SOURCE="project ($WORKDIR/.claude/settings.local.json)"
fi
if [ -z "$NOTION_TOKEN" ] && [ -f "$HOME/.claude/settings.json" ] && grep -q '"NOTION_TOKEN"' "$HOME/.claude/settings.json" 2>/dev/null; then
  NOTION_TOKEN=$(node -e "console.log((JSON.parse(require('fs').readFileSync(process.env.HOME+'/.claude/settings.json','utf8')).mcpServers?.notionApi?.env?.NOTION_TOKEN)||'')")
  SOURCE="global (~/.claude/settings.json)"
fi

if [ -z "$NOTION_TOKEN" ]; then
  echo "ERROR: NOTION_TOKEN not found in either settings file — cannot run REST fallback."
  exit 1
fi

echo "REST: using token from $SOURCE."
HTTP_STATUS=$(curl -sS -o /tmp/notion_verify.json -w "%{http_code}" \
  -X POST "https://api.notion.com/v1/data_sources/$DATA_SOURCE_ID/query" \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"page_size":1}')
echo "REST HTTP $HTTP_STATUS"
head -c 500 /tmp/notion_verify.json; echo
```

Interpret the REST result:

- **HTTP 200** → the token has access to that data source. Notion is verified. If Stage A also failed, tell the user: *"The loaded Notion MCP in this session is pointing at a different workspace, but your configured token is correct. Restart Claude Code so the right MCP loads (and resume this session)."* — then mark Notion as verified-pending-restart.
- **HTTP 401 / 403** → token is invalid or the Notion integration is **not connected** to the area's backend page. Tell the user to ask the admin to add the integration to the `<area>` Notion backend page.
- **HTTP 404 (`object_not_found`)** → integration connected but doesn't have access to this specific data source. Same admin ask: connect the integration to the area's backend page (which propagates to its data sources).
- **HTTP 429** → rate limited. Retry once after ~5s. If still 429, move on and flag for the user.
- **Other / network error** → surface verbatim. Don't fabricate a diagnosis.

Only conclude *"integration missing on backend page"* after Stage B has returned 401/403/404. A Stage A failure alone is not enough — it's frequently caused by the wrong MCP being loaded, not by a real integration gap.

#### Slack verification

Post a test message to the area channel:

```
:wave: *<Name> joining as IC of <Area>*
Symlinks installed, protocols loaded.
```

Channel IDs (lookup from area's `slack-protocol.md`):
- strategy → `C0AHN1ZB18T`
- governance → `C0AKQDJEFFH`
- product → (read from protocol)
- gtm → (read from protocol)

If post fails (`not_in_channel`), ask the admin to invite the bot. If post fails with auth error in a multi-workspace setup, suggest the user restart Claude Code so the correct Slack MCP loads.

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
