---
description: Push local files to Notion databases. Supports calls, brainstorm outputs, docs, or any .md file. Use whenever the user says "push to Notion", "push calls", "send to Notion", "upload to Notion", or similar.
argument-hint: "[filename or leave empty to browse]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, mcp__notionApi__API-post-page, mcp__notionApi__API-patch-page, mcp__notionApi__API-patch-block-children, mcp__notionApi__API-query-data-source, mcp__notionApi__API-retrieve-a-page, mcp__notionApi__API-get-block-children, mcp__slack__slack_post_message, mcp__slack__slack_get_channel_history
---

## Push to Notion

Push local `.md` files to any Notion database, with optional Slack announcement. The skill adapts its behavior based on the **content type** being pushed.

The Notion and Slack protocol files for each teamspace are already loaded in the CLAUDE.md rules context — reference them directly for database IDs, write rules, and message formats. Do not re-read them.

---

### Step 1 — What are you pushing?

1. Ask the user **what type of content** he wants to push:
   - **Call** — analyzed call from `meetings/calls-analyzed/`
   - **Generic file** — any `.md` file from anywhere in the project

2. Based on the type, gather the file(s):

   **For calls:**
   - Glob all `.md` files in `meetings/calls-analyzed/`.
   - Read `memory.md` in this skill's folder — exclude already-pushed files.
   - If no unpushed calls remain, suggest running `/analyze-calls` first. Stop.
   - Show a one-line summary per file: `[filename] — [H1 title] — [date]`.

   **For generic files:**
   - If a filename was passed as argument, use that.
   - Otherwise ask the user for the file path or glob pattern.
   - Show the list of matched files.

3. Ask the user to **select which file(s)** to push.

---

### Step 2 — Where does it go?

1. **Discover connected teamspaces** by listing folders matching `.claude/rules/*-teamspace-communication/` in the project root. Each folder name with the `-teamspace-communication` suffix removed is a teamspace (e.g. `strategy`, `governance`, `product`, `gtm`). Exclude `general` from the list — it has no Notion databases.

2. Ask the user which **teamspace** to push to, presenting only the discovered list. Do not hardcode teamspace names.

3. **Propose** the target database based on content type:
   - Calls → Meetings database
   - Generic files → ask the user which database (show the list from the teamspace's notion-protocol)

4. **Ask for confirmation** — the user can override the proposed database.

This determines the `database_id`, `data_source_id`, and Slack channel ID from the teamspace's `notion-protocol.md` and `slack-protocol.md`.

---

### Step 3 — Build and preview the Notion entry

For each selected file, build the Notion page payload.

#### Call mode — property mapping

| Notion property | Source | Notes |
|---|---|---|
| `Meeting name` (title) | H1 heading of the file | e.g. "Constitution", "Adyant Dagur" |
| `Date` (date) | `Date:` field in the file | Parse ISO datetime, use date portion |
| `Category` (multi_select) | Infer from content | Pick exactly one option from the `Category` list in the target teamspace's `notion-schema.md`. If unclear → ask the user. |
| `Tags` (multi_select) | Infer from content | Pick one or more options from the `Tags` list in the target teamspace's `notion-schema.md`. Skip if that teamspace's Meetings DB has no `Tags` field. |
| `Contributors` (relation) | the user | Set the user as Contributor. Skip if that teamspace's Meetings DB has no `Contributors` relation (e.g. Governance). |

**Category & Tags inference:** each teamspace defines its own valid options for `Category` and `Tags` in its `notion-schema.md`. Always consult the target teamspace's schema before inferring. Pick the option(s) that best match the call content. If multiple plausibly fit or none clearly fit, ask the user.

#### Generic mode — property mapping

| Notion property | Source | Notes |
|---|---|---|
| Title field (varies by DB) | H1 heading of the file, or filename | Ask the user to confirm |
| Other properties | Ask the user | Show available properties from the schema for the target DB |

#### Page body content (both modes)

Write the file content as a single `code` block with `language: "markdown"`. If the content exceeds 2000 characters, split it into multiple `rich_text` items within the same code block. This follows the Notion writing protocol.

#### Preview

Show the user a formatted preview of each entry before pushing:

```
File: [filename]
Target: [database name] ([teamspace])
Action: create | update
Title: [title value]
Properties: [key properties set]

Body preview (first 3 lines)...
```

Ask for confirmation before proceeding. The user can adjust any property at this point.

---

### Step 4 — Duplicate check and push

For each confirmed file:

1. **Check for duplicates** — query the target database using `API-query-data-source`, filtering by title. If a match is found, warn the user and ask whether to skip or create anyway.
2. **Create the page** using `API-post-page` with the target `database_id` as parent. Set all properties from the mapping.
3. **Add the body** using `API-patch-block-children` after page creation — append the markdown code block as a child of the new page.
4. **On success**, update `memory.md` in this skill's folder — append a row to the tracking table.
5. **On failure**, report the error, do not mark as pushed, continue with the next file.

---

### Step 5 — Slack announcement

After all files are pushed, ask the user: **"Want to announce on Slack?"**

If yes, post one message per pushed file on the teamspace Slack channel using the appropriate format:

**For Meetings (calls):**
```
:heavy_plus_sign: *Created — Meetings*
[Full name of the person] — [1-line summary of what the call was about]
```

The full name must match the filename convention (e.g. `2026-03-20_simone-trezzi.md` → "Simone Trezzi"). The summary should be a concise sentence extracted from the call's Summary section.

**For all other databases:**
```
:heavy_plus_sign: *Created — [Database name]*
Entry: [title]
Fields: [key fields set]
```

---

### Step 6 — Local file cleanup (calls only)

After pushing call files, ask the user what to do with each local file in `meetings/calls-analyzed/`:

- **Move** — ask for the destination folder (e.g. `meetings/calls-archived/`)
- **Delete** — remove the file from the repo
- **Keep** — leave it where it is

Skip this step entirely for generic files.

---

### Step 7 — Confirm

Tell the user:
- Which files were pushed (created vs updated)
- Which teamspace and database
- What was posted on Slack (if anything)
- What happened to local files (moved / deleted / kept) — calls only

---

### Tracking pushed files

Maintain `memory.md` in this skill's folder to track what has been pushed.

Format:
```markdown
# Pushed Files

| File | Type | Pushed to | Database | Date pushed |
|---|---|---|---|---|
| 2026-03-10_samuele.md | call | Strategy | Meetings | 2026-03-13 |
```

- Before showing available files (Step 1), read this file and exclude already-pushed entries (for calls).
- After each successful push (Step 4), append a row using Edit.
- If the file does not exist, create it with the header using Write.

---

### Edge cases

- **Empty source** → inform the user, suggest relevant action (e.g. `/analyze-calls` for calls).
- **All files already pushed** → inform the user, nothing to do.
- **Notion API error** → report the error, do not mark as pushed, continue with next file.
- **Duplicate found** → always check before pushing. Warn the user and ask whether to skip or create anyway.
