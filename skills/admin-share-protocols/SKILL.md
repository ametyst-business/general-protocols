---
description: Take protocol changes from the admin's personal OS (master copies) and push them to the appropriate ametyst-business/{area}-protocols repo. Admin-only — verifies push access via gh. Trigger phrases - "share protocols", "push protocols", "publish my protocols".
allowed-tools: Read, Write, Edit, Glob, Bash(gh *), Bash(git *), Bash(diff *), Bash(cp *), Bash(mkdir *), AskUserQuestion, mcp__slack__slack_post_message
---

## admin-share-protocols

Push selected protocol files from the admin's personal OS to the
appropriate `ametyst-business/{area}-protocols` repo. Workflow: diff local
master copies vs cloned area repo → admin selects what to push → copy into
clone → git commit + push → optionally announce on Slack.

---

### Step 1 — Pick area and verify access

Use `AskUserQuestion`: "Which area's protocols are you sharing? (general, strategy, governance, product, gtm)"

Verify the admin has push access:

```bash
gh repo view ametyst-business/<area>-protocols --json viewerPermission --jq .viewerPermission
```

If the result is not `ADMIN`, `MAINTAIN`, or `WRITE`, tell the admin they cannot push to that repo and stop.

---

### Step 2 — Locate the local clone

```bash
CLONE="$HOME/Ametyst/protocols/<area>-protocols"
if [ ! -d "$CLONE/.git" ]; then
  mkdir -p "$HOME/Ametyst/protocols"
  gh repo clone ametyst-business/<area>-protocols "$CLONE" -- --quiet
else
  git -C "$CLONE" pull --rebase --quiet
fi
```

---

### Step 3 — Ask the admin which files to share

The admin's personal OS may contain files for multiple areas (rules, skills,
agents, guides for areas other than the one selected in Step 1). Do NOT
auto-discover everything and assume it belongs to this area. Instead, let
the admin pick explicitly.

Ask: "Where are your master copies? (default: current working directory's `.claude/`)"

Default is `<cwd>/.claude/`. Confirm.

Then, for each subfolder in order (`rules`, `skills`, `agents`, `guides`):

1. Glob `<personal-os>/<sub>/**/*` (excluding `.gitkeep` and hidden files).
2. Show the list and use `AskUserQuestion` (multi-select):
   "Which files/folders from `<sub>/` do you want to share to `<area>-protocols`? (skip if none)"
3. Skip a subfolder entirely if the admin selects nothing.

Collect the union of selected paths. If nothing selected across all subfolders, tell the admin "nothing to share" and stop.

---

### Step 4 — Compute diff on selected files only

For each selected path, compare its content against the corresponding path in `$CLONE`:

- **NEW** — selected path is missing from `$CLONE`
- **CHANGED** — exists in both, content differs
- **IDENTICAL** — already up to date (drop from the push set)

Show a summary:

```
Selected for share to <area>-protocols:

NEW files (will be added):
  - skills/eng-sprint-planner/SKILL.md
  - rules/some-rule.md

CHANGED files (will be replaced in clone):
  - skills/ops-brainstorm/SKILL.md
  - rules/general-teamspace-communication/slack-protocol.md

IDENTICAL (already up to date — skipping):
  - rules/notion-content-rules.md
```

For each CHANGED entry, optionally show a brief `diff -u` summary (lines added/removed).

If the resulting NEW + CHANGED set is empty, tell the admin "nothing to push" and stop. Otherwise, ask for final confirmation: "Push these N files to `<area>-protocols`?"

---

### Step 5 — Apply changes to clone

For each selected file, copy from the personal OS path to the corresponding `$CLONE` path. Create parent directories if needed.

```bash
mkdir -p "$(dirname "$CLONE/$rel")" && cp "$personal/$rel" "$CLONE/$rel"
```

---

### Step 6 — Commit and push

Ask for a commit message. Default suggestion: `update: <comma-separated file basenames>`.

```bash
cd "$CLONE"
git add -A
git diff --cached --stat   # show what's about to be committed
git commit -m "<message>"
git push
```

If `git push` fails (e.g., behind remote), `git pull --rebase` and retry. If still fails, abort and report.

---

### Step 7 — Optional Slack announcement

Ask: "Announce on `#<area>`? (yes / no)"

If yes, post to the area channel:

```
:books: *Protocols updated — <area>*
<list of files changed>
```

---

### Step 8 — Recap

Tell the admin:

```
Pushed to ametyst-business/<area>-protocols.
Commit: <hash>
Files: <count> (<count> new, <count> changed)
Slack: <announced | skipped>
```
