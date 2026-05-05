---
description: Audit an area-wiki for health (orphan pages, broken wikilinks, missing frontmatter, duplicate entities, stale ingested raw), apply approved fixes, push to GitHub. Admin-only. Trigger phrases - "lint wiki", "audit wiki", "wiki health", "clean up wiki".
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(gh *), Bash(git *), Bash(mkdir *), Bash(mv *), AskUserQuestion
---

## admin-lint-and-share-wiki

Run a health audit on an area-wiki. Surface findings to the admin,
let them approve fixes interactively, apply them, commit + push.

---

### Step 1 — Pick area-wiki, verify access, clone

Same as `admin-ingest-and-share-wiki` Step 1-2.

---

### Step 2 — Run audit

For the cloned wiki at `$CLONE`, check:

1. **Orphan pages** — any page in `$CLONE/wiki/**/*.md` that is not referenced (`[[Page]]`, `[[Page|alias]]`, or path link) by any other page or by `index.md`. Use Grep across `$CLONE/wiki/**/*.md` and `$CLONE/index.md`.

2. **Broken wikilinks** — for every `[[X]]` reference in any page, check that a page resolves (filename slug match or alias). Collect unresolved targets.

3. **Missing frontmatter** — pages without a `name:` or `type:` field. Read each page header.

4. **Duplicate entities** — pages with similar `name:` or overlapping `aliases:`. Build a map of normalized slugs.

5. **Stale `_ingested/`** — files in `$CLONE/raw/_ingested/` older than 6 months (use file mtime). Optional cleanup.

Build a structured report.

---

### Step 3 — Present the report

Show the admin:

```
LINT REPORT — <area>-wiki

Orphan pages (<N>):
  - wiki/entities/foo.md
  - wiki/concepts/bar.md

Broken wikilinks (<N>):
  - wiki/entities/baz.md      links [[NonExistent]]
  - wiki/concepts/qux.md      links [[Old Name]]

Missing frontmatter (<N>):
  - wiki/concepts/quz.md      (no `type:`)
  - wiki/entities/x.md        (no `name:`)

Duplicate entities (<N>):
  - wiki/entities/openai.md  ↔  wiki/entities/open-ai.md

Stale _ingested/ files (<N>):
  - raw/_ingested/2025-01-12_call.md  (older than 6 months)
```

If the wiki is healthy (zero findings), tell the admin and stop.

---

### Step 4 — Admin approves fixes per category

For each non-empty category, ask the admin what to do:

- **Orphan pages** — options:
  - `archive` — move each to `wiki/_archive/`
  - `keep` — flag in frontmatter (`status: orphan-flagged`) for later review
  - `skip` — do nothing

- **Broken wikilinks** — options:
  - `remove` — convert each `[[X]]` to plain text `X`
  - `flag` — replace with `[[X|TODO: lint flagged broken link]]`
  - `skip`

- **Missing frontmatter** — options:
  - `stub` — add minimal frontmatter (`name`, `type`) inferred from filename and folder
  - `skip` — flag with `# TODO: lint missing frontmatter` at top of file

- **Duplicate entities** — surface side-by-side; the admin chooses which to keep (no auto-merge — too risky).

- **Stale `_ingested/`** — options:
  - `delete` — remove the files
  - `skip`

Use `AskUserQuestion` per category, multi-select where appropriate.

---

### Step 5 — Apply fixes

For each approved fix, perform the corresponding `Edit`, `Write`, or `mv`. Track every change for the recap.

---

### Step 6 — Commit and push

```bash
cd "$CLONE"
git add -A
git diff --cached --stat
git commit -m "lint: <summary — orphans archived, links flagged, frontmatter stubbed, etc.>"
git push
```

If `git push` fails, `git pull --rebase` and retry. Never force-push.

---

### Step 7 — Recap

Tell the admin:

```
Lint complete on <area>-wiki.

Fixes applied:
  - <N> orphan pages archived
  - <N> broken wikilinks removed/flagged
  - <N> frontmatter stubs added
  - <N> stale ingested files deleted

Items needing manual review:
  - <list of duplicate-entity decisions deferred>

Commit: <hash>
```
