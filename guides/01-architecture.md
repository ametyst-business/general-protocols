# Architecture — Connected Protocols

How the Ametyst connected-protocols system works: who owns what, how an
agent stays in sync, and how files flow between local OSes and GitHub.

---

## Repos

Two repos per area:
- `ametyst-business/{area}-protocols` — rules, skills, agents, guides for the area
- `ametyst-business/{area}-wiki` — LLM-maintained knowledge base for the area

Five areas: `general`, `strategy`, `governance`, `product`, `gtm`.

---

## Roles & access

| Role | Read access | Write access |
|---|---|---|
| Founder | All repos | All repos |
| Head of area | All repos except governance | Their area's protocols + wiki |
| IC of area | Their area + general | None (PR only) |

---

## Source of truth — Head's local OS

The head of an area maintains the master copy of their area's protocols
**locally** in their personal OS (e.g., a `domain-expansion/` repo). When
the head wants to share an update with the team, they push selected files
to `ametyst-business/{area}-protocols` via the `admin-share-protocols`
skill.

The GitHub repo is therefore a downstream copy curated by the head — not a
shared editing space. Other members consume it read-only.

---

## Symlink architecture (IC / member)

A non-head member onboards via `/settings-onboarding`. The skill:

1. Clones `ametyst-business/general-protocols` into `~/Ametyst/protocols/general-protocols`
2. Creates symlinks from inside that clone into the user's `.claude/{rules,skills,agents}/`
3. Merges the `general-protocols/CLAUDE.md` snippet (between `<!-- AMETYST:BLOCK:START -->` and `<!-- AMETYST:BLOCK:END -->`) into the user's `.claude/CLAUDE.md`
4. (Optionally) builds self-context

For each area the user has access to, they run `/settings-become-teamspace-ic`,
which repeats the same pattern with `{area}-protocols`.

Every clone is auto-refreshed at session start by the morning-startup
section of CLAUDE.md (`git -C ~/Ametyst/protocols/<area>-protocols pull
--rebase --quiet`). Symlinks make the user's `.claude/` reflect the latest
content immediately.

---

## Wiki access

Wikis are queried on demand. The list of connected wikis lives inside the
user's CLAUDE.md (inside the `AMETYST:BLOCK`). `ops-context-loader` reads
that table and asks which wiki(s) to query.

- `mode: github` — cache-clones to `~/.cache/ametyst-wikis/<name>` and queries locally
- `mode: local` — reads the path in place (e.g., a sibling repo on disk)

---

## Daily flow

```
Session start
  └─ Morning-startup (in CLAUDE.md)
     └─ git pull on every connected protocols clone

User runs a skill
  └─ skill is loaded via symlink from the clone (always fresh)

Head edits a protocol
  └─ edits in their personal OS
     └─ runs admin-share-protocols
        └─ pushes to ametyst-business/{area}-protocols
           └─ next session, every IC pulls and gets the update
```

---

## Onboarding cheat sheet

```
/settings-onboarding                     once, on first contact with Ametyst
/settings-become-teamspace-ic <area>     once per area you have access to
```

---

## Anatomy of a user's `.claude/`

After onboarding + becoming IC of strategy:

```
your-repo/.claude/
├── CLAUDE.md
│   ├── <your existing content>
│   ├── <!-- AMETYST:BLOCK:START -->
│   │   ## Ametyst — operating context
│   │   ### Communication protocols
│   │   ### Connected wikis (table)
│   │   ### Morning startup (git pull lines)
│   │   ### Language
│   ├── <!-- AMETYST:BLOCK:END -->
│   ├── <!-- AMETYST:AREA:strategy:START -->
│   │   (strategy-specific block)
│   └── <!-- AMETYST:AREA:strategy:END -->
├── rules/
│   ├── communication-routing.md          → symlink → general-protocols
│   ├── notion-content-rules.md           → symlink → general-protocols
│   ├── general-teamspace-communication/  → symlink → general-protocols
│   └── strategy-teamspace-communication/ → symlink → strategy-protocols
├── skills/
│   ├── ops-brainstorm/                   → symlink → general-protocols
│   ├── ops-context-loader/               → symlink → general-protocols
│   └── ...                               → symlinks
└── agents/
    └── ...                               → symlinks
```
