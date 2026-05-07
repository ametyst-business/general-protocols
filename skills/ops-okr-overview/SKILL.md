---
description: Fetch OKRs (Objectives + Key Results) from all connected areas plus Projects + Tasks where the user is a Contributor. Outputs a schematic markdown overview directly in chat — no file written. By default shows only `In progress` and `Not started` items. Pass `all` to also include `Failed` and `Done`. Use whenever the user says "okr overview", "show my okrs", "what's on my plate", or runs /okr-overview [all].
allowed-tools: Read, Glob, Bash(ls *), mcp__notionApi__API-query-data-source
---

## ops-okr-overview

Single-shot read-only skill. No questions at start — when invoked, fetch
everything based on the areas the user is connected to and print a structured
overview in chat.

Behavior:
1. Detect connected areas from `.claude/rules/` (skip `general` and `governance`).
2. Resolve the user's `Team Members` page ID in each area (via email; fallback to name).
3. Fetch Objectives + Key Results for each area (default: only `In progress` + `Not started`).
4. Fetch Projects + Tasks where `Contributors contains <user-id>` (default: only `In progress` + `Not started`).
5. Print one markdown block in chat. No file write.

### Argument

- (none) — default. Only `In progress` and `Not started` items.
- `all` — also include `Failed` and `Done` (apply no Status filter). Use only when explicitly requested.

Hierarchy is visual only — Strategy is rendered first because it's
conceptually top-level, but Notion has no parent-child relation between
strategy KRs and product/gtm KRs.

---

### Step 1 — Detect connected areas

```bash
ls -1 .claude/rules/ | grep -E '-teamspace-communication$' | sed 's/-teamspace-communication$//'
```

Filter out `general` and `governance`. Result is a list of area names — typically a subset of `[strategy, product, gtm]` depending on which teamspaces the user has joined.

If the resulting list is empty, tell the user "no connected areas with OKRs" and stop.

Order for the output: `strategy` first (top-level), then the rest alphabetically.

---

### Step 2 — Load each area's protocol IDs

For each area, read `.claude/rules/<area>-teamspace-communication/notion-protocol.md` to get the `data_source_id` for:
- Objectives
- Key Results
- Projects
- Tasks
- Team Members

Hold these in memory for the rest of the skill — do not re-read.

---

### Step 3 — Resolve user's Team Members ID per area

The user's email is in the auto-loaded `userEmail` memory line.

For each area, query Team Members in parallel:

```json
{
  "data_source_id": "<area Team Members data_source_id>",
  "filter": { "property": "Email", "email": { "equals": "<user email>" } },
  "page_size": 1
}
```

If a query returns 0 results for an area (email mismatch between git config and Notion record), fall back to a name-based lookup using the title field. The user's name is available in the conversation context (typically derived from the email local-part or known from prior sessions). Query without filter, page_size 20, and match the `Name` (title) property locally.

Store `{ area: tm_page_id }`. If both lookups fail, leave that area without a `tm_page_id` and skip the personal filtering for it in Step 5 — still show area-wide OKRs in Step 4.

---

### Step 4 — Fetch Objectives + Key Results per area

For each area, run two queries in parallel (Objectives + Key Results), and run all areas in parallel as well.

**Default filter** (`Status` is `In progress` OR `Not started`):

```json
{
  "data_source_id": "<Objectives data_source_id>",
  "filter": {
    "or": [
      { "property": "Status", "status": { "equals": "In progress" } },
      { "property": "Status", "status": { "equals": "Not started" } }
    ]
  },
  "page_size": 100
}
```

**`all` mode** (argument `all` was passed): omit the `filter` field entirely so every row is returned regardless of Status.

Same shape for Key Results.

For each row, extract:
- `Goal name` (title)
- `Status` (status)
- `Priority` (select)
- `Quarter` (multi_select — comma-join)
- `Due date` (date.start, may be null)
- For Objectives: `Key Results` relation (list of KR page IDs)
- For Key Results: `Objectives` relation (list of Objective page IDs)

Build in memory:
- `objectives_by_id[obj_id] = { name, status, priority, quarter, due, kr_ids, area }`
- `krs_by_id[kr_id] = { name, status, priority, quarter, due, area }`

Sort objectives within an area by Due date ascending (null last), then Priority (High → Medium → Low).

---

### Step 5 — Fetch Projects + Tasks where user is Contributor

For each area where `tm_page_id` was resolved, run two queries in parallel.

**Default filter** (Contributors AND Status is `In progress` OR `Not started`):

```json
{
  "data_source_id": "<Projects data_source_id>",
  "filter": {
    "and": [
      { "property": "Contributors", "relation": { "contains": "<tm_page_id>" } },
      {
        "or": [
          { "property": "Status", "status": { "equals": "In progress" } },
          { "property": "Status", "status": { "equals": "Not started" } }
        ]
      }
    ]
  },
  "page_size": 100
}
```

**`all` mode**: drop the inner `or` block — keep only the Contributors filter:

```json
{
  "filter": { "property": "Contributors", "relation": { "contains": "<tm_page_id>" } }
}
```

Same shape for Tasks.

For Projects extract: `Project name`, `Status`, `Priority`, `Due date`, `area`.
For Tasks extract: `Task name`, `Status`, `Priority`, `Due date`, `area`, parent `Projects` relation (for context).

Sort each list by Due date ascending (null last).

Skip Workflows entirely. They are not part of this overview.

---

### Step 6 — Format output in chat

Write a single markdown block directly in chat. **Do not write to any file.**

Header line:
```
# OKR Overview — <YYYY-MM-DD>
```

Then, for each area in order (Strategy first, then alphabetical), one section:

```
## <Area title-cased>

### Objectives
- **<Goal name>** — <Status> · <Priority> · <Quarter> · Due <YYYY-MM-DD or "—">
  - KR: **<Goal name>** — <Status> · <Quarter> · Due <YYYY-MM-DD or "—">
  - KR: ...
```

Render Objectives' KRs as nested bullets, matched by relation IDs. If an Objective has no KRs, skip the nested block. If a KR has no parent Objective in the area, list it under a final "_Unlinked KRs_" sub-bullet inside that area's `### Objectives`.

If an area has zero non-Done Objectives, render `_No active objectives._` under that area's heading.

Then a separator and the personal section:

```
---

## Your active work

### Projects
| Project | Area | Status | Priority | Due |
|---|---|---|---|---|
| <name> | <area> | <status> | <priority> | <YYYY-MM-DD or —> |

### Tasks
| Task | Area | Project | Status | Priority | Due |
|---|---|---|---|---|---|
| <name> | <area> | <project name or —> | <status> | <priority> | <YYYY-MM-DD or —> |
```

Rules:
- If Projects list is empty across all areas, replace the table with `_No active projects assigned to you._`
- Same for Tasks.
- For Tasks, `Project` column shows the parent project's name when the relation exists; `—` otherwise. Do not query each project page separately — resolve the name from the Projects already fetched in Step 5; if not found there, just print the relation count or `—`.
- No emojis.
- Plain markdown. No code fences around the whole output.
- Dates always `YYYY-MM-DD`.

---

### Notes

- This skill is read-only. It must not write to Notion or to disk.
- Hierarchy across areas is visual only.
- Governance is excluded by design (admin-only, not part of cross-area OKR planning).
- Workflows are not included.
- All output goes to chat.
