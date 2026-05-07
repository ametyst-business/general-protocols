---
description: Fetch OKRs (Objectives + Key Results) from all connected areas plus Projects + Tasks where the selected contributors (you, other people, or both) are listed. Outputs a schematic markdown overview directly in chat — no file written. By default shows only `In progress` and `Not started` items. Pass `all` to also include `Failed` and `Done`. Use whenever the user says "okr overview", "show my okrs", "what's on my plate", or runs /okr-overview [all].
allowed-tools: Read, Glob, Bash(ls *), AskUserQuestion, mcp__notionApi__API-query-data-source
---

## ops-okr-overview

Read-only skill. Starts with a short interactive selection (whose OKRs to show),
then fetches everything based on the connected areas and prints a structured
overview in chat.

Behavior:
1. Ask the user whose OKRs to show: mine, other people, or mine + other people.
2. If other people are involved, list available Team Members per area and ask which one(s) to include.
3. Detect connected areas from `.claude/rules/` (skip `general` and `governance`).
4. Resolve the selected users' `Team Members` page IDs in each area (via email; fallback to name).
5. Fetch Objectives + Key Results for each area (always area-wide — OKRs are not person-specific).
6. Fetch Projects + Tasks where `Contributors contains <tm_page_id>` for each selected user.
7. Print one markdown block in chat. No file write.

Default Status filter: only `In progress` + `Not started`. With `all` argument: no Status filter.

### Argument

- (none) — default. Only `In progress` and `Not started` items.
- `all` — also include `Failed` and `Done` (apply no Status filter). Use only when explicitly requested.

Hierarchy is visual only — Strategy is rendered first because it's
conceptually top-level, but Notion has no parent-child relation between
strategy KRs and product/gtm KRs.

---

### Step 0 — Mode selection

Ask the user (use `AskUserQuestion` so the choice is structured):

- Question: "Whose OKRs do you want to see?"
- Options: `Mine`, `Other people`, `Mine + other people`

Map the answer to a `mode` variable: `mine` | `others` | `both`.

If `mode == mine`: skip Step 0b — the only selected user is the current user themselves.

---

### Step 0a — Detect connected areas

```bash
ls -1 .claude/rules/ | grep -E '-teamspace-communication$' | sed 's/-teamspace-communication$//'
```

Filter out `general` and `governance`. Result is a list of area names — typically a subset of `[strategy, product, gtm]` depending on which teamspaces the user has joined.

If the resulting list is empty, tell the user "no connected areas with OKRs" and stop.

Order for the output: `strategy` first (top-level), then the rest alphabetically.

For each area, read `.claude/rules/<area>-teamspace-communication/notion-protocol.md` once and hold the `data_source_id` for Objectives, Key Results, Projects, Tasks, Team Members in memory.

---

### Step 0b — People selection (only if `mode != mine`)

For each connected area, query Team Members in parallel, filtered to people whose Status is `Active`, `Stealth`, `Part-time`, or `Contract` (i.e. anyone currently contributing — exclude only `On Leave` and `Inactive`):

```json
{
  "data_source_id": "<area Team Members data_source_id>",
  "filter": {
    "or": [
      { "property": "Status", "status": { "equals": "Active" } },
      { "property": "Status", "status": { "equals": "Stealth" } },
      { "property": "Status", "status": { "equals": "Part-time" } },
      { "property": "Status", "status": { "equals": "Contract" } }
    ]
  },
  "page_size": 100
}
```

Build `people_by_area = { area: [ { name, tm_page_id, email } ] }`.

A single human can appear in multiple area tables (different `Team Members` DBs per area). Deduplicate by name across areas to build a single list of distinct people, while keeping the per-area mapping `name -> { area: tm_page_id }` for later querying.

If `mode == both`, exclude the current user from the selectable list (they're already included). Identify the current user by the `userEmail` memory line; fallback to name match.

Show the available people grouped by area:

```
Available people:

Strategy:
- Alice Rossi
- Bob Bianchi

Product:
- Charlie Verdi
- Dana Neri

GTM:
- Eve Marrone
```

Then ask (use `AskUserQuestion` with `multiSelect: true` if supported, otherwise free-text):

- Question: "Which person/people do you want to include? (one or more)"
- Options: the deduplicated list of names.

Resolve selected names to `{ name, per_area_tm_ids: { area: tm_page_id } }`. Hold this list as `selected_users`.

If `mode == both`, prepend the current user to `selected_users` (resolved in Step 1 below).

If `mode == others` and the user selects nothing, stop with a short message.

---

### Step 1 — Resolve current user's Team Members ID per area (only if `mode in { mine, both }`)

The user's email is in the auto-loaded `userEmail` memory line.

For each area, query Team Members in parallel:

```json
{
  "data_source_id": "<area Team Members data_source_id>",
  "filter": { "property": "Email", "email": { "equals": "<user email>" } },
  "page_size": 1
}
```

If a query returns 0 results for an area (email mismatch between git config and Notion record), fall back to a name-based lookup using the title field. Query without filter, page_size 100, and match the `Name` (title) property locally.

Store the current user as one of `selected_users` with their `per_area_tm_ids`. If a lookup fails for an area, leave that area without a `tm_page_id` for them and skip their personal Projects/Tasks filtering for that area — area-wide OKRs are still shown.

---

### Step 2 — Fetch Objectives + Key Results per area

Area-wide — independent of `selected_users`. For each area, run two queries in parallel (Objectives + Key Results), and run all areas in parallel as well.

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

### Step 3 — Fetch Projects + Tasks per selected user

For each user in `selected_users`, and for each area where their `tm_page_id` was resolved, run two queries in parallel.

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

Group results by user: `projects_by_user[name] = [...]`, `tasks_by_user[name] = [...]`.
Sort each list by Due date ascending (null last).

Skip Workflows entirely. They are not part of this overview.

---

### Step 4 — Format output in chat

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

Then a separator and one personal section **per selected user**, in this order: the current user first (if included), then the others in selection order:

```
---

## Active work — <User name>

### Projects
| Project | Area | Status | Priority | Due |
|---|---|---|---|---|
| <name> | <area> | <status> | <priority> | <YYYY-MM-DD or —> |

### Tasks
| Task | Area | Project | Status | Priority | Due |
|---|---|---|---|---|---|
| <name> | <area> | <project name or —> | <status> | <priority> | <YYYY-MM-DD or —> |
```

If only the current user is selected (`mode == mine`), render the heading as `## Your active work` to preserve the existing wording.

Rules:
- If a user's Projects list is empty across all areas, replace their Projects table with `_No active projects._`
- Same for Tasks → `_No active tasks._`
- For Tasks, `Project` column shows the parent project's name when the relation exists; `—` otherwise. Do not query each project page separately — resolve the name from the Projects already fetched in Step 3 for any user; if not found there, just print `—`.
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
- The interactive selection at Step 0 is mandatory — do not assume `mine` without asking, even if the user just says "okr overview".
