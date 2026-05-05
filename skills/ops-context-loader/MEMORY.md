# Context Loader — Memory

Learned corrections and operational rules accumulated from real usage.

---

## Rule 1 — Always ask for depth, no exceptions

Never skip Step 1 (parameter gathering) even if the target seems obvious from the user's message. Depth must always be explicitly confirmed with the user via `AskUserQuestion`. Deciding autonomously wastes tokens and produces the wrong scope of context.

## Rule 2 — CLAUDE.md is the exploration boundary

This system uses `CLAUDE.md` files as structural signals. The rule is:

- Every explorable level **has a `CLAUDE.md`**. If a submodule listed in a parent CLAUDE.md does not have its own `CLAUDE.md`, **stop there** for that branch of the tree.
- **Never substitute** a missing `CLAUDE.md` with: source files, `package.json`, generic `README.md`, or any other file. No `CLAUDE.md` = nothing to read in that branch.
- For GitHub repos: first fetch the full list of `CLAUDE.md` paths using `git/trees` recursive, then read **only those** according to the requested depth. Do not explore folders not in the list.

## Rule 3 — No free exploration

The context loader is not a generic explorer. It never goes to "see what's in" a folder. It only follows the explicit graph of `CLAUDE.md` files. If there is no CLAUDE.md structure, report "(not found)" and move on.
