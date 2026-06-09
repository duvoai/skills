---
name: aop-writer
description: >
  Draft, rewrite, or critique a Duvo Agent AOP. Use when the user wants
  to improve an existing AOP or write a new one from a brief. Returns the AOP
  as a single complete markdown document in the canonical GOAL / STEPS / NOTES
  shape.
license: MIT
metadata:
  author: duvoai
  version: "1.0.2"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# AOP Writer

## What is Duvo?

[Duvo](https://duvo.ai) is an AI-powered automation platform that handles repetitive business work across the systems a team already uses. Unlike traditional automation that follows rigid, pre-programmed rules, a Duvo **Agent** understands the goal, adapts to each situation, and acts on the user's behalf through their own **Connections** (linked tools like Gmail, Slack, or a CRM) — as if the user were doing the work themselves. An Agent is configured once — its **AOP** (the markdown procedure that becomes its prompt), Connections, and settings form a **Build** — and then runs **Runs**: individual executions, each with an input, a full transcript, and a result.

## What an AOP is in Duvo

An AOP is the instruction document attached to an Agent. At Run-start time it is concatenated into the Agent's prompt, so its quality directly determines how the Agent behaves on every Run — against real Connections, with real escalation consequences. You are not writing documentation for a human reader; you are writing operational instructions that an LLM-driven Agent will execute on live data.

Treat that downstream effect as the constraint on every word you write.

## The single most important rule

**An AOP is designed around processing ONE case/item from start to finish — all decision branches, all error handling, all terminal states, in one document.** The Duvo platform handles iteration (picking up the next case, retrying, scheduling). Never write batch loops, "for each item" wrappers, or "process all pending records". The AOP describes what to do with a single case, end to end.

## Canonical structure

Every AOP uses the same three-section markdown shape (see `references/aop-structure.md` for the full template):

- **`# GOAL`** — one sentence describing what the Agent does on a single case and the expected outcome.
- **`# STEPS`** — a numbered list of imperative actions. Inline everything that belongs to a single step (data formats, decision criteria, early returns, error handling, handover or completion conditions) into that step.
- **`# NOTES`** _(optional)_ — only for genuinely cross-cutting concerns that don't belong to a single step (timezone conventions, naming patterns). Omit it otherwise.

Do **not** create separate `PREREQUISITES`, `ERROR HANDLING`, `DATA HANDLING`, `ESCALATION`, or `TRIGGER` sections. Inline those concerns into the step where they apply.

## Voice

Outcome-focused, transparent, written for a non-engineer ops manager. Use Duvo terminology (Agent, Run, AOP, Connection — not assignment, task, job, integration). Imperative steps. Concrete thresholds, not fuzzy adverbs. See `references/aop-voice.md` for the full rules and worked examples.

## Patterns that produce excellent AOPs

A great AOP weaves these patterns into the `# STEPS` section.

1. **Reference tools by Connection name.** "Use **Gmail** to send the response." "Search **Google Sheets** for the customer record." This tells the runtime Agent which Connection (and therefore which tool) to use.

2. **Inline decision criteria with concrete thresholds.** Replace "use your judgment" with explicit `if … then …` rules. "If the amount exceeds $5,000, escalate." "If the supplier has had >3 late deliveries in the past year, note this as leverage."

3. **Inline early returns and error handling** with `If [condition]: [action]`. Example: "If `initial_postpone_done` is NOT present in the case data: call `update_case` to set it, then call `postpone_case` with `1d`, and exit."

4. **Human-in-the-loop checkpoints for high-stakes actions.** "If the contract value exceeds $100,000: use **Human in the loop** to get approval before sending." Make HITL a first-class step on any branch where a wrong action would be costly to reverse, not an afterthought.

5. **The postpone-then-retry idiom for SLAs and waiting periods.** On first pickup, set a flag on the case and `postpone_case` with the wait duration; on second pickup, do the real work. This is how an AOP expresses "wait N days then act" without a separate Agent per stage. See `references/case-lifecycle.md`.

6. **Terminal closure on every branch.** For queue-consuming Agents, every branch of the AOP must end with `complete_case`, `fail_case`, `postpone_case`, or `request_handover`. For standalone Agents, every branch must end with a clear terminal action (send the email, save the row, post the result). Cases left in an ambiguous state are the #1 failure mode.

## Case lifecycle (queue-driven Agents)

If the Agent is a producer, consumer, or producer+consumer (it has the `case-queue-producer` or `case-queue-consumer` Connection), the AOP must reference the case lifecycle tools by name in the step where they apply:

- `claim_case`, `update_case`, `postpone_case`, `complete_case`, `fail_case`
- `add_cases`, `list_cases` (producers)
- `request_handover` (when the Agent passes the case to a different Agent)

Read `references/case-lifecycle.md` for the per-tool contract, the postpone-then-retry idiom, and the handover-vs-terminality rule before writing a queue-driven AOP.

If the request gives no signal about queues, cases, or handovers, write a standalone-Agent AOP and skip the lifecycle tools.

## Anti-patterns — reject and rewrite

- **Vague outcomes** ("handle the request appropriately"). Replace with a concrete success condition.
- **No decision criteria** ("use your best judgment"). Replace with explicit thresholds.
- **Batch / iteration logic** ("for each pending invoice, do X"). The platform iterates. Rewrite as a single-case AOP.
- **Missing terminal closure**. For queue Agents, every branch must end in `complete_case` / `fail_case` / `postpone_case` / `request_handover`. For standalone Agents, every branch must end in a concrete action.
- **Walls of text**. Use numbered steps with a blank line between them. Two-to-four-sentence paragraphs inside a step.
- **Hardcoded team data** (specific supplier names, dollar thresholds tied to one team, individual people's email addresses) embedded in the AOP body. These belong in Files or as Setup inputs, not in the AOP.
- **Internal jargon** ("API call", "webhook", "endpoint"). Speak in business actions; the Agent chooses the tool call.
- **Steps that say "how" instead of "what"** ("call `create_order` with body=…"). Steps describe the business action; the Agent figures out the tool call.
- **Separate `PREREQUISITES` / `ERROR HANDLING` / `ESCALATION` / `DATA HANDLING` sections.** Inline them into the relevant step.
- **One AOP doing the work of three.** If the steps cross connection domains _and_ span distinct cadences (e.g., real-time inbox scan and a 7-day reminder cycle) _and_ have a wait or hand-off between phases, the work is two Agents, not one. Flag the decomposition signal in your response and offer to split.

## Decomposition signal

An AOP that has collapsed multiple Agents into one is hard to debug, hard to evolve, and tends to fail at the seams. When critiquing or rewriting, watch for these signals:

- Steps that change Connection domain mid-flow (e.g., Snowflake → Gmail → Enterprise Browser).
- Steps that wait for an external event between phases (a reply, an approval, a daily cycle).
- The AOP exceeds ~10 top-level steps and the steps fall into clear phase boundaries.

When these signals appear, mention the decomposition option in a one-line **Note to user** above the rewritten AOP. Do not refuse to return the requested AOP — produce it, but flag the option.

## Output rule

Return a single complete AOP in markdown, in the canonical `# GOAL` / `# STEPS` / `# NOTES` shape. Not a diff. Not a list of suggestions. Not prose like "here are some changes I'd recommend". The caller will diff your output against the original.

The only exceptions:

- The user explicitly asked for a critique, suggestions, or a list of issues without a rewrite — respond in that shape, with a clear heading noting it is not a usable AOP.
- You detected a decomposition signal — return the requested AOP first, then a one-line **Note to user** above it offering the split.

Keep AOPs concise — a great AOP fits on a screen or two. If yours is much longer, you're probably writing two AOPs as one (see the decomposition signal above).

## Reading the request

1. **Find the current AOP** in the conversation. If one is present, treat it as authoritative — preserve anything the user did not ask to change. If absent, the user is drafting from a brief; produce the full document from scratch.
2. **Determine intent from wording.** Full rewrite, targeted fix, draft-from-brief, or critique. Match the output shape to the intent — do not return a critique when asked for a rewrite, and do not return a rewrite when asked "tell me what's wrong with this".
3. **Detect queue context.** Look for keywords (queue, case, postpone, handover, reminder, follow-up, SLA, daily, hourly) or the presence of any `_case` / `request_handover` tool name in the current AOP. If detected, apply the case lifecycle patterns; otherwise treat the Agent as standalone.

You have no access to the user's data, Connections, Files, or Agents. Write from the AOP text and request in the conversation only. Do not invent tool calls, integration names, or product features.

## Final check before returning

Walk through this list once on your draft. Fix anything that fails before returning the AOP to the user.

- [ ] Structure: starts with `# GOAL` (one sentence), then `# STEPS` (numbered), `# NOTES` only if needed.
- [ ] Single-case framing: no batch loops, no "for each", no "all pending".
- [ ] Every step is an imperative business action; tools referenced by Connection name.
- [ ] Every decision point has a concrete threshold or rule — no "use your judgment".
- [ ] Every branch terminates: queue Agents end in a case lifecycle tool; standalone Agents end in a concrete action.
- [ ] Duvo terminology used consistently (Agent, Run, AOP, Connection, Files, Login, Start Work, Setup).
- [ ] No separate Preconditions / Escalation / Error Handling sections — everything inlined.
- [ ] No hardcoded team-specific data that should live in Files or Setup.
- [ ] If a decomposition signal is present, a one-line **Note to user** mentions the split option.

See `references/aop-examples.md` for canonical worked examples to calibrate against before returning.

## See also

- `run-debugger` — when a Run ran with an AOP and failed, start there to diagnose the root cause; it will hand off to this skill if the fix is an AOP rewrite.
- `workflow-debugger` — audits an Agent across many Runs; hands off here when a systemic fix is an AOP change.
- `duvo-cli` — once the AOP is finalized, `duvo revisions create` / `duvo revisions update` ships it from the terminal.

## Resources

- [Duvo](https://duvo.ai) — product website
- [Duvo documentation](https://docs.duvo.ai) — building Agents, AOPs, Connections, queues
- [Web app](https://app.duvo.ai) — author Agents and watch Runs in the browser
- [Duvo CLI (`@duvoai/cli`)](https://www.npmjs.com/package/@duvoai/cli) — script AOP-driven Agents from the terminal; pairs with the `duvo-cli` skill
- [Public skill repository](https://github.com/duvoai/skills) — the MIT-licensed community release of this skill, packaged for installation in third-party Claude Code setups
