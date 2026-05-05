---
description: Ingest raw sources (calls, articles, recaps) in an area-wiki into entity/concept/domain pages, then push to GitHub. Admin-only - verifies push access. Trigger phrases - "ingest wiki", "ingest and share", "distill wiki", "wiki ingest".
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(gh *), Bash(git *), Bash(mkdir *), Bash(mv *), AskUserQuestion
---

## admin-ingest-and-share-wiki

Read selected raw sources in `{area}-wiki/raw/`, distill them into
entity/concept/domain pages with wikilinks, then commit and push.
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

### Step 3 — Pick raw source(s) to ingest

Glob `$CLONE/raw/**/*.md` excluding `_ingested/`. Show the list grouped by subfolder (calls, articles, recaps).

Use `AskUserQuestion` (multi-select): which raw source(s) to ingest now?

If zero candidates, tell the admin: "No raw sources to ingest. Add files under `$CLONE/raw/` first." Stop.

---

### Step 4 — Read existing wiki state

Read `$CLONE/index.md` and Glob `$CLONE/wiki/**/*.md`. Build an in-memory map: `slug -> path` for every existing page.

---

### Step 5 — Distill each raw source

For each selected raw source:

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
- After all pages for a raw source are written, `mv` the raw source from its current location into `$CLONE/raw/_ingested/`.

---

### Step 7 — Update index.md and append to log.md

If new top-level entities/domains were added, update `$CLONE/index.md` to list them.

Append to `$CLONE/log.md`:

```
## <YYYY-MM-DD> — ingest

Source: <raw source filename(s)>
Pages created: <list>
Pages updated: <list>
```

---

### Step 8 — Commit and push

```bash
cd "$CLONE"
git add -A
git commit -m "ingest: <comma-separated raw source basenames>"
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
