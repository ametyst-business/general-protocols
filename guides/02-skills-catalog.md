# Skills catalog — general-protocols

Every skill installed via `/settings-onboarding` or
`/settings-become-teamspace-ic` from `general-protocols`. For each: when to
use, trigger phrases, and one example.

---

## settings-onboarding

**When:** first time you're being onboarded to Ametyst — your agent has no Ametyst protocols yet.
**What it does:** clones `general-protocols`, symlinks rules/skills/agents into your `.claude/`, configures Slack MCP, merges the AMETYST:BLOCK snippet into your CLAUDE.md, optionally builds self-context.
**Trigger phrases:** "onboard me", "set up Ametyst", "settings onboarding", "connect to Ametyst".
**Example:** `/settings-onboarding`

Run **once** per agent. Re-running is safe (idempotent) but unnecessary unless you need to re-symlink.

---

## settings-become-teamspace-ic

**When:** you've been granted access to an area (`strategy` / `governance` / `product` / `gtm`) and want your agent to start using its protocols.
**What it does:** clones `{area}-protocols`, symlinks them, appends the area block to your CLAUDE.md, registers the area's wiki under "Connected wikis", updates `communication-routing.md`, verifies Notion + Slack access.
**Trigger phrases:** "connect strategy", "join governance", "become ic of product", "settings become teamspace ic gtm".
**Example:** `/settings-become-teamspace-ic` → asks which area.

Run **once per area**. Re-running re-symlinks and re-merges (idempotent).

---

## ops-context-loader

**When:** before a non-trivial task that needs understanding of repo structure, domain boundaries, or topical knowledge from a wiki.
**Modes:**
- **Structural** — crawl `CLAUDE.md` files of one or more folders (returns a repo snapshot)
- **Wiki** — search one or more connected wikis for a topic (returns top matching pages)
- **Both** — combined

**Trigger phrases:** "give me context on", "load context", "dammi contesto su X", "read the repo".
**Example:** `/ops-context-loader a16z` → asks mode (Wiki) → asks which wikis (e.g., `general`, `strategy`) → returns ranked pages.

---

## ops-brainstorm

**When:** you want to explore a topic interactively, with a chosen reasoning lens (experienced co-founder, product builder, technical researcher, brand designer).
**What it does:** picks a prompt persona, optionally loads context, then drives an interactive session that ends in a structured output file (e.g., a PRD).
**Trigger phrases:** "brainstorm", "let's think about", "ragiona con me su".
**Example:** `/ops-brainstorm hiring plan Q3` → asks prompt (`experienced-cofounder`) → asks output path → goes interactive.

---

## ops-push-to-notion

**When:** you have a local `.md` file (call analysis, brainstorm output, doc) and want it as a Notion entry in the right teamspace database.
**What it does:** asks teamspace, infers the target database from the file's nature, shows a preview, pushes via the Notion MCP, optionally announces on Slack.
**Trigger phrases:** "push to Notion", "send to Notion", "upload X to Notion".
**Example:** `/ops-push-to-notion` → asks file type (call/brainstorm/doc) → asks teamspace → shows preview → pushes.

---

## When to combine skills

Common chains:

```
/settings-onboarding                       (once, day 1)
/settings-become-teamspace-ic strategy     (once, when given access)
/ops-context-loader <topic>                (before a task that needs context)
/ops-brainstorm <topic>                    (to produce a structured doc)
/ops-push-to-notion                        (to share the result in Notion)
```
