---
description: "Interactive brainstorming skill. Use whenever Patrick says 'brainstorm', 'let's think about', 'ragiona con me su', or wants to explore a topic interactively with tools and reasoning modes."
argument-hint: "[topic]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, AskUserQuestion, EnterPlanMode, ExitPlanMode, WebSearch, WebFetch, mcp__notionApi__*, mcp__slack__*
---

## Brainstorm

Generic, interactive brainstorming runtime. Becomes specific based on the **prompt** (reasoning mode, persistent for the entire session) selected at startup. Capabilities (tool descriptions) live globally in `.claude/capabilities/` and are available to any skill. Output path is chosen by the user at startup — never hardcoded.

The skill is self-improving: during execution it detects missing prompts and offers to create them on the spot.

---

### Step 1 — Setup

1. Glob `prompts/*.md` inside this skill's folder to discover available reasoning modes.
2. Glob `.claude/capabilities/*.md` to discover available tools/services.

3. Present Patrick with:
   - **Prompts available:** list each `.md` file name + its one-line description. Ask which one to use, or a custom reasoning mode, or none.
   - **Capabilities available:** list each `.md` file from `.claude/capabilities/` — these are the tools/services available during the session.
   - **Topic:** ask what the brainstorming is about (skip if already provided as argument or in the conversation).
   - **Output path:** ask Patrick where to save the output (e.g. `areas/governance/brainstorms/`, `areas/product/brainstorms/`, etc.). Create the folder if it does not exist.
   - **Context sources:** ask where to fetch context from before starting. Options:
     - **Structural** — crawl CLAUDE.md files of one or more domain-expansion folders (e.g. `areas/governance/`, `areas/product/`, `context/company-context/`). Patrick specifies the folders.
     - **Wiki** — query the `lighting-void/` knowledge base on a topic. Patrick specifies the topic.
     - **Both** — structural + wiki combined.
     - **None** — skip context loading, start from the raw topic.

4. Read and internalize the selected prompt file. The prompt becomes the persistent reasoning lens for the entire session.

---

### Step 1.5 — Context loading (sub-agent, skip if Patrick chose "None")

If Patrick selected Structural, Wiki, or Both in Step 1:

1. Invoke the `ops-context-loader` skill via a sub-agent with the chosen mode and the specified folders/topic. Do NOT navigate the folders or the wiki manually.
2. Keep the returned context snapshot in your working context. Do NOT dump the raw snapshot to Patrick — summarize it in 3-5 bullets when presenting the plan in Step 2.
3. If the sub-agent returns empty or inconclusive context, report it and ask Patrick whether to proceed anyway or pick different sources.

Proceed to Step 2.

---

### Step 2 — Brainstorming plan (plan mode)

1. Enter plan mode.
2. If context was loaded in Step 1.5, open with a brief recap (3-5 bullets) of what's relevant from the snapshot. Then, based on the topic, active prompt, available capabilities, and loaded context, propose a step-by-step brainstorming plan. Each step should describe:
   - What you will do (research, ask a question, use a capability, synthesize)
   - What you expect to learn or decide
3. **Suggest connections.** Beyond the plan steps, proactively suggest:
   - **Data sources** that could enrich the brainstorming (e.g. "we could pull investor feedback from analyzed calls", "Notion financials could give us budget constraints")
   - **Automation opportunities** that emerge from the topic — things Patrick could build after the brainstorming to operationalize the result (e.g. "if we define hiring criteria here, you could create a skill that loops weekly to check new candidates against them", "this fundraising checklist could become a capability that pre-fills pitch prep for each investor")
   - **Cross-domain links** to other areas of domain-expansion that are relevant (e.g. "areas/gtm/ has customer data that could inform this", "the call transcripts in meetings/ might have VC feedback on this exact question")

   These are suggestions, not requirements — Patrick picks what to pursue.
4. Patrick reviews and adjusts the plan.
5. Exit plan mode once aligned.

The plan is a guide, not a rigid script — execution can deviate based on what emerges.

---

### Step 3 — Interactive execution

1. Create the output file at the path confirmed by Patrick in Step 1.
2. Follow the plan fluidly:
   - Use tools from `.claude/capabilities/` when the brainstorming calls for external data.
   - Ask Patrick targeted questions to gather his perspective or make decisions.
   - Use sub-agents for heavy research tasks (web search, Notion queries, context loading).
   - If something unexpected emerges, adapt the direction — the plan is not a constraint.
3. Throughout execution, keep the conversation interactive: present findings, propose options, ask for Patrick's take, iterate.
4. **Keep suggesting connections** as they emerge during execution — not just at plan time. If a finding opens a door to another data source, automation idea, or cross-domain link, surface it immediately.

#### Self-improvement: capability/prompt branch

During execution, if you identify a gap — a prompt that doesn't exist yet but would make this (or future) brainstorming sessions better — surface it to Patrick:

> *"I think we'd benefit from [X]. Want me to:*
> 1. *Note it and continue (create it later)*
> 2. *Branch — I create it now and we resume after"*

**If Patrick chooses option 1 (note it):**
Add it to a `## Gaps identified` section in the output file. These become a backlog for future improvement.

**If Patrick chooses option 2 (branch):**
1. **Checkpoint** — write the current state to the output file:
   ```markdown
   ## Checkpoint — paused to create [prompt]
   **Where we were:** [current step in the plan]
   **Conclusions so far:** [bullet points]
   **Resume from:** [what to do next when we come back]
   ```
2. **Create** the prompt file in `prompts/`.
3. **Load** the new file into the active session.
4. **Resume** the brainstorming from the checkpoint, now with the new tool available.

---

### Step 4 — Conclusion

1. When the brainstorming reaches a natural conclusion, write the final output to `brainstorms/<topic-slug>.md` with this structure:

```markdown
# Brainstorm — <Topic>

**Date:** <YYYY-MM-DD>
**Prompt:** <prompt used or "none">
**Capabilities:** <capabilities used or "none">

## Starting question
<the original question or topic>

## Key findings
<bullet points of what was discovered or decided during the brainstorming>

## Conclusion
<the final answer, decision, or synthesis>

## Next steps
<actionable items that come out of the brainstorming>

## Connections surfaced
<data sources, automation ideas, cross-domain links, and skill/capability suggestions that emerged — even if not pursued during this session>

## Gaps identified
<capabilities or prompts that were missing and should be created for future sessions>
```

2. Update `memory.md` in this skill's folder — add a row to the registry table.
3. Show Patrick a brief recap of the brainstorming result.
