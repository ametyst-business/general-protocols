# Communication Routing

## Rule
Before any Slack message or Notion API call, read the relevant protocol files below.

## Routing table

| Context | Protocol files |
|---|---|
| #general, #standup, #news | `.claude/rules/general-teamspace-communication/` |

All paths are relative to the project root.

## Loading rules
- Always load `notion-protocol.md` first (IDs, queries, write rules)
- Load `notion-schema.md` only when creating or updating entries
- Load `slack-protocol.md` before posting any message
- Follow message formats exactly — no paraphrasing

## Note
This file is installed by `/settings-onboarding`. Each `/settings-become-teamspace-ic`
appends a row to the routing table for the area joined.
