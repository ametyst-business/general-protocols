---
description: Load structured context by crawling CLAUDE.md files (structural mode) or querying one or more connected wikis for a topic (wiki mode), or both. Use whenever you need context — repo structure, domain boundaries, or topical knowledge. Triggered by "give me context on", "read the repo", "load context", "what's in this codebase", or "dammi contesto su".
argument-hint: "[target/path or topic]"
allowed-tools: Bash(gh *), Read, Grep, Glob, AskUserQuestion, mcp__notionApi__API-query-data-source
---

## Context Loader

> **Before executing:** read `MEMORY.md` in this folder — it contains learned rules that override default behavior.

Your job is to collect context for a task. Three modes:

- **Structural** — crawl CLAUDE.md files of one or more repos/folders. Returns a structured repo snapshot.
- **Wiki** — query one or more connected wikis (listed in the project root `.claude/CLAUDE.md` under "Connected wikis") for a topic. Returns the top N relevant wiki pages.
- **Both** — run both and merge the output.

---

### Step 1 — Choose mode and gather parameters

Ask the user via AskUserQuestion:

1. **Mode** — `Structural` / `Wiki` / `Both`?

Then, based on mode:

**If Structural or Both:**
- **Targets** — repos/folders to crawl (absolute local path or `org/repo`, comma-separated for multiple)
- **Depth** — `1` = root only, `2` = root + direct submodules, `3` = root + two levels
- **Focus** — optional, specific domain to prioritize (default: `all`)

**If Wiki or Both:**
- **Read connected wikis** — open the project root `.claude/CLAUDE.md` and find the "Connected wikis" table. Each row has `Name | Mode | Location`. The list is the source of truth for which wikis can be queried.
- **Wikis** — ask the user which wiki(s) to query. Use AskUserQuestion with the list of names from the table; allow multi-select. If the user just says "all", select every row. Always ask — never default silently.
- **Topic** — what to look up (e.g. "a16z", "fundraising strategy", "European banking regulation")

**Optional (any mode) — Notion context:**
If the task needs Notion data, ask which teamspace (Strategy / Governance) and which database. Query via `API-query-data-source` with appropriate Team filter.

---

### Step 2 — Run structural crawl (if mode is Structural or Both)

Launch a sub-agent of type `repo-crawler` with this prompt:

```
Targets: {TARGETS}
Depth: {DEPTH}
Focus: {FOCUS}
```

The `repo-crawler` agent handles CLAUDE.md boundary rules. See `.claude/agents/repo-crawler.md`. Wait for its response.

---

### Step 3 — Run wiki lookup (if mode is Wiki or Both)

For each wiki the user selected in Step 1, resolve a local path and run the same lookup logic. Do this inline — no sub-agent needed.

**Resolve wiki path (per selected wiki):**
- If `Mode = local` → use the `Location` value as-is (absolute path).
- If `Mode = github` → cache-clone the repo:
  ```bash
  CACHE="$HOME/.cache/ametyst-wikis/<wiki-name>"
  if [ -d "$CACHE/.git" ]; then
    git -C "$CACHE" pull --quiet --rebase
  else
    gh repo clone <Location> "$CACHE" -- --quiet
  fi
  ```
  Then use `$CACHE` as the resolved path.

**Run lookup against each resolved path:**

1. **Read the catalog** at `<path>/index.md` (skip if missing).

2. **Match the topic** against wiki pages using this priority order:
   - **Filename match** (highest priority) — topic matches the slug of a page file. Glob: `<path>/wiki/**/*{topic-slug}*.md`
   - **Alias match** — Grep `<path>/wiki/**/*.md` for `aliases:.*{topic}`
   - **Tag match** — same but `tags:.*{topic}`
   - **Content mention** (lowest priority) — full-text Grep for the topic across `<path>/wiki/**/*.md`

3. **Rank and dedupe across all selected wikis.** If a page appears in multiple tiers, keep the highest rank. Max 5 pages total across the union of wikis. Tag each result with the wiki it came from.

4. **Read the top N matching pages** (default 3, up to 5 if the topic is broad). Include their full content — frontmatter included — in the snapshot.

5. **If zero matches across all selected wikis:** report to the user "No wiki pages on this topic yet across {wikis queried}. The topic may be latent in raw sources but not yet distilled." Suggest running `/wiki-ingest` on a relevant raw source.

---

### Step 4 — Present the result

Build the final snapshot. Structure:

```markdown
# Context Snapshot

**Generated:** {today's date}
**Mode:** {Structural | Wiki | Both}

---

## Repo structure
{output from repo-crawler, only if Structural/Both}

---

## Wiki pages — topic: {topic}
{only if Wiki/Both}

### {page 1 filename} — matched on: {filename | alias | tag | content}
{full page content including frontmatter}

### {page 2 filename} — matched on: ...
{...}

---

## Notion data (if queried)
{...}
```

Present to the user. Say: "Context loaded. Here's what I found — ready to work from this."

Keep the snapshot in the current conversation context for follow-up work.

---

### Notes

- Wiki pages are markdown with wikilinks `[[Page]]`. When a page references another via wikilink, the agent resolves it by filename search if deeper context is needed.
- Each wiki is maintained by its own `ops-wiki-ingest` / `ops-wiki-lint` (run from inside that wiki repo). If a wiki seems stale or missing entries, those skills are the lever, not this one.
- Structural crawl does NOT descend into any connected wiki — that's what wiki mode is for. Don't double-count.
- Cached GitHub-mode wiki clones live at `~/.cache/ametyst-wikis/<wiki-name>`. Safe to delete; the skill re-clones on demand.
