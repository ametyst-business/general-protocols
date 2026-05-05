---
description: Capture a new raw source (URL, pasted transcript/text, or existing file) into an area-wiki, distill it into entity/concept/domain pages with wikilinks, then commit and push. Admin-only - verifies push access. Trigger phrases - "ingest wiki", "ingest and share", "distill wiki", "wiki ingest", "add to wiki".
allowed-tools: Read, Write, Edit, Glob, Grep, WebFetch, Bash(gh *), Bash(git *), Bash(mkdir *), Bash(mv *), Bash(curl *), AskUserQuestion
---

## admin-ingest-and-share-wiki

End-to-end ingest flow on an area-wiki:

1. Capture a raw source (link, paste, or existing file) → write it to `raw/`
2. Distill the raw into entity/concept/domain pages with wikilinks
3. Move the processed raw to `_ingested/`
4. Commit and push

HIL: every page change is approved by the admin before write.

---

### Step 1 — Pick area-wiki and verify access

Use `AskUserQuestion`: "Which area-wiki? (general, strategy, governance, product, gtm)"

```bash
gh repo view ametyst-business/<area>-wiki --json viewerPermission --jq .viewerPermission
```

If not `ADMIN`/`MAINTAIN`/`WRITE`, stop.

---

### Step 2 — Locate or clone the wiki

```bash
CLONE="$HOME/Ametyst/wikis/<area>-wiki"
if [ ! -d "$CLONE/.git" ]; then
  mkdir -p "$HOME/Ametyst/wikis"
  gh repo clone ametyst-business/<area>-wiki "$CLONE" -- --quiet
else
  git -C "$CLONE" pull --rebase --quiet
fi
```

---

### Step 3 — Capture the raw source

The raw source is rarely already in `raw/` — it usually starts as a URL or a
pasted transcript/text. This step lets the admin provide the source in any
of three ways and writes it into the right `raw/` subfolder.

Use `AskUserQuestion`: "How are you providing the raw source?"

Options:
- **URL** — paste a link (article, blog post, recap doc, etc.). The skill
  fetches the content via `WebFetch` (or `curl` for plain pages).
- **Paste** — paste a transcript, notes, an article body, or any text
  directly. The skill writes it verbatim.
- **Existing file** — pick an existing file already in `$CLONE/raw/**/*.md`
  (excluding `_ingested/`). Use this if the file was dropped in manually.

#### URL mode
1. Ask the admin for the URL.
2. Fetch the content. Prefer `WebFetch` for HTML pages, `curl -fsL` for
   plain text or markdown URLs.
3. If the fetch fails, fall back to asking the admin to paste the content.

#### Paste mode
Ask the admin to paste the text. Accept multi-line input. Store as-is.

#### Existing file mode
Glob `$CLONE/raw/**/*.md` excluding `_ingested/`. Show the list grouped by
subfolder. Use `AskUserQuestion` to pick one. Skip Step 4 (no need to
write a new file — it already exists). Use the existing path as
`raw_path`.

#### After capturing (URL or Paste mode)

Ask: "Which subfolder should this go in? (calls, articles, recaps, notes)"
- `calls/` — analyzed call transcripts
- `articles/` — articles you've read for the area
- `recaps/` — meeting recaps and notes
- `notes/` — any other text source

Ask: "Filename? (lowercase, hyphens, no extension — `.md` will be added)"
Suggest a default based on content (e.g., for a URL: a slug from the page
title; for paste: today's date + first words).

Build `raw_path = $CLONE/raw/<subfolder>/<filename>.md`. Create the
subfolder if missing. If `raw_path` already exists, ask whether to
overwrite or pick a different filename.

#### Optional metadata header

Prepend a small frontmatter block to the captured content (URL mode and
Paste mode only):

```yaml
---
source: <URL or "pasted">
captured_at: <YYYY-MM-DD>
captured_by: <admin name from self-context, if known>
---
```

Then the body. Write to `raw_path`.

---

### Step 4 — Read existing wiki state

Read `$CLONE/index.md` and Glob `$CLONE/wiki/**/*.md`. Build an in-memory map: `slug -> path` for every existing page.

---

### Step 5 — Distill the raw source

Operate on `raw_path` (the file just written in Step 3 or the existing one
the admin picked):

1. Read the raw content.
2. Identify the **entities** (people, companies, products, partnerships), **concepts** (frameworks, ideas, mental models), and **domains** (macro-areas) mentioned.
3. For each item:
   - **If a page exists** (slug or alias match) → propose an UPDATE: new bullet point, link, or section, plus inline `[[wikilinks]]` to other pages.
   - **If new** → propose a NEW page with frontmatter:
     ```yaml
     ---
     name: <Display name>
     aliases: [<alt name 1>, <alt name 2>]
     tags: [<tag>, ...]
     type: entity | concept | domain
     ---
     ```
     Plus a body that summarizes the relevant insight from the raw source, with `[[wikilinks]]` to related pages.

4. Present the proposals to the admin (file path + the diff/new content). Use `AskUserQuestion` for each one: approve / edit / skip.

---

### Step 6 — Apply approved changes

- New pages → `Write` to `$CLONE/wiki/<type>s/<slug>.md`
- Updates → `Edit` to existing page
- After all pages have been written, `mv` `raw_path` into `$CLONE/raw/_ingested/`. The raw source itself stays in the repo as the audit trail; only its location moves.

---

### Step 7 — Update index.md and append to log.md

If new top-level entities/domains were added, update `$CLONE/index.md` to list them.

Append to `$CLONE/log.md`:

```
## <YYYY-MM-DD> — ingest

Source: <raw source filename> (<URL or "pasted" or "existing file">)
Pages created: <list>
Pages updated: <list>
```

---

### Step 8 — Commit and push

```bash
cd "$CLONE"
git add -A
git commit -m "ingest: <raw source basename>"
git push
```

If push fails, `git pull --rebase` and retry. Never force-push.

---

### Step 9 — Recap

Tell the admin:

```
Ingested <N> raw source(s) into <area>-wiki.

Pages created: <list>
Pages updated: <list>
Raw moved to _ingested/: <list>

Commit: <hash>
```
