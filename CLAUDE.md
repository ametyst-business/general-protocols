# CLAUDE.md — general-protocols

This file is a snippet. The block between `<!-- AMETYST:BLOCK:START -->` and
`<!-- AMETYST:BLOCK:END -->` is merged into the user's `.claude/CLAUDE.md`
during `/settings-onboarding`. Do not treat this file as a standalone
entry-point for an agent.

<!-- AMETYST:BLOCK:START -->
## Ametyst — operating context

This agent is connected to the Ametyst ecosystem. The following sections
describe how to work within it. Sections inside this block are managed by
`/settings-onboarding` and `/settings-become-teamspace-ic`. Do not edit by
hand — re-run the relevant skill instead.

### Communication protocols
Before any Slack or Notion interaction, read `.claude/rules/communication-routing.md` to find the correct protocol files for the target channel or database. Skip this step only if the active skill already specifies which protocol files to read.

### Connected wikis
Wikis the agent can query in wiki mode. `ops-context-loader` reads this table.

| Name | Mode | Location |
|---|---|---|
| general | github | ametyst-business/general-wiki |

### Morning startup
On the first message of every session, pull the latest version of every
connected protocols repo so symlinks reflect the head's last push:

- `git -C ~/Ametyst/protocols/general-protocols pull --rebase --quiet`

### Language
All files created or modified in this repo must be written in English — no
exceptions. The user may speak in any language; replies in chat can follow
the user's language, but any content written to files must be in English.
<!-- AMETYST:BLOCK:END -->
