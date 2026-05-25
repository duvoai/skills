---
name: sop-writer
description: >
  Draft, rewrite, or critique a Duvo Assignment SOP. Use when the user wants
  to improve an existing SOP or write a new one from a brief. Returns the SOP
  as a single complete markdown document in the canonical GOAL / STEPS / NOTES
  shape.
license: MIT
metadata:
  author: duvoai
  version: "1.0.1"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# SOP Writer

## What is Duvo?

[Duvo](https://duvo.ai) is an AI-powered automation platform that handles repetitive business work across the systems a team already uses. Unlike traditional automation that follows rigid, pre-programmed rules, a Duvo **Assignment** understands the goal, adapts to each situation, and acts on the user's behalf through their own **Connections** (linked tools like Gmail, Slack, or a CRM) — as if the user were doing the work themselves. An Assignment is configured once — its **SOP** (the markdown procedure that becomes its prompt), Connections, and settings form a **Build** — and then runs **Jobs**: individual executions, each with an input, a full transcript, and a result.

## What an SOP is in Duvo

An SOP is the instruction document attached to an Assignment. At Job-start time it is concatenated into the Assignment's prompt, so its quality directly determines how the Assignment behaves on every Job — against real Connections, with real escalation consequences. You are not writing documentation for a human reader; you are writing operational instructions that an LLM-driven Assignment will execute on live data.

Treat that downstream effect as the constraint on every word you write.

## The single most important rule

**An SOP is designed around processing ONE case/item from start to finish — all decision branches, all error handling, all terminal states, in one document.** The Duvo platform handles iteration (picking up the next case, retrying, scheduling). Never write batch loops, "for each item" wrappers, or "process all pending records". The SOP describes what to do with a single case, end to end.

## Canonical structure

Every SOP uses the same three-section markdown shape (see `references/sop-structure.md` for the full template):

- **`# GOAL`** — one sentence describing what the Assignment does on a single case and the expected outcome.
- **`# STEPS`** — a numbered list of imperative actions. Inline everything that belongs to a single step (data formats, decision criteria, early returns, error handling, handover or completion conditions) into that step.
- **`# NOTES`** _(optional)_ — only for genuinely cross-cutting concerns that don't belong to a single step (timezone conventions, naming patterns). Omit it otherwise.

Do **not** create separate `PREREQUISITES`, `ERROR HANDLING`, `DATA HANDLING`, `ESCALATION`, or `TRIGGER` sections. Inline those concerns into the step where they apply.

## Voice

Outcome-focused, transparent, written for a non-engineer ops manager. Use Duvo terminology (Assignment, Job, SOP, Connection — not agent, task, run, integration). Imperative steps. Concrete thresholds, not fuzzy adverbs. See `references/sop-voice.md` for the full rules and worked examples.

## Patterns that produce excellent SOPs

A great SOP weaves these patterns into the `# STEPS` section.

1. **Reference tools by Connection name.** "Use **Gmail** to send the response." "Search **Google Sheets** for the customer record." This tells the runtime Assignment which Connection (and therefore which tool) to use.

2. **Inline decision criteria with concrete thresholds.** Replace "use your judgment" with explicit `if … then …` rules. "If the amount exceeds $5,000, escalate." "If the supplier has had >3 late deliveries in the past year, note this as leverage."

3. **Inline early returns and error handling** with `If [condition]: [action]`. Example: "If `initial_postpone_done` is NOT present in the case data: call `update_case` to set it, then call `postpone_case` with `1d`, and exit."

4. **Human-in-the-loop checkpoints for high-stakes actions.** "If the contract value exceeds $100,000: use **Human in the loop** to get approval before sending." Make HITL a first-class step on any branch where a wrong action would be costly to reverse, not an afterthought.

5. **The postpone-then-retry idiom for SLAs and waiting periods.** On first pickup, set a flag on the case and `postpone_case` with the wait duration; on second pickup, do the real work. This is how an SOP expresses "wait N days then act" without a separate Assignment per stage. See `references/case-lifecycle.md`.

6. **Terminal closure on every branch.** For queue-consuming Assignments, every branch of the SOP must end with `complete_case`, `fail_case`, `postpone_case`, or `request_handover`. For standalone Assignments, every branch must end with a clear terminal action (send the email, save the row, post the result). Cases left in an ambiguous state are the #1 failure mode.

## Case lifecycle (queue-driven Assignments)

If the Assignment is a producer, consumer, or producer+consumer (it has the `case-queue-producer` or `case-queue-consumer` Connection), the SOP must reference the case lifecycle tools by name in the step where they apply:

- `claim_case`, `update_case`, `postpone_case`, `complete_case`, `fail_case`
- `add_cases`, `list_cases` (producers)
- `request_handover` (when the Assignment passes the case to a different Assignment)

Read `references/case-lifecycle.md` for the per-tool contract, the postpone-then-retry idiom, and the handover-vs-terminality rule before writing a queue-driven SOP.

If the request gives no signal about queues, cases, or handovers, write a standalone-Assignment SOP and skip the lifecycle tools.

## Anti-patterns — reject and rewrite

- **Vague outcomes** ("handle the request appropriately"). Replace with a concrete success condition.
- **No decision criteria** ("use your best judgment"). Replace with explicit thresholds.
- **Batch / iteration logic** ("for each pending invoice, do X"). The platform iterates. Rewrite as a single-case SOP.
- **Missing terminal closure**. For queue Assignments, every branch must end in `complete_case` / `fail_case` / `postpone_case` / `request_handover`. For standalone Assignments, every branch must end in a concrete action.
- **Walls of text**. Use numbered steps with a blank line between them. Two-to-four-sentence paragraphs inside a step.
- **Hardcoded team data** (specific supplier names, dollar thresholds tied to one team, individual people's email addresses) embedded in the SOP body. These belong in Files or as Setup inputs, not in the SOP.
- **Internal jargon** ("API call", "webhook", "endpoint"). Speak in business actions; the Assignment chooses the tool call.
- **Steps that say "how" instead of "what"** ("call `create_order` with body=…"). Steps describe the business action; the Assignment figures out the tool call.
- **Separate `PREREQUISITES` / `ERROR HANDLING` / `ESCALATION` / `DATA HANDLING` sections.** Inline them into the relevant step.
- **One SOP doing the work of three.** If the steps cross connection domains _and_ span distinct cadences (e.g., real-time inbox scan and a 7-day reminder cycle) _and_ have a wait or hand-off between phases, the work is two Assignments, not one. Flag the decomposition signal in your response and offer to split.

## Decomposition signal

An SOP that has collapsed multiple Assignments into one is hard to debug, hard to evolve, and tends to fail at the seams. When critiquing or rewriting, watch for these signals:

- Steps that change Connection domain mid-flow (e.g., Snowflake → Gmail → Enterprise Browser).
- Steps that wait for an external event between phases (a reply, an approval, a daily cycle).
- The SOP exceeds ~10 top-level steps and the steps fall into clear phase boundaries.

When these signals appear, mention the decomposition option in a one-line **Note to user** above the rewritten SOP. Do not refuse to return the requested SOP — produce it, but flag the option.

## Output rule

Return a single complete SOP in markdown, in the canonical `# GOAL` / `# STEPS` / `# NOTES` shape. Not a diff. Not a list of suggestions. Not prose like "here are some changes I'd recommend". The caller will diff your output against the original.

The only exceptions:

- The user explicitly asked for a critique, suggestions, or a list of issues without a rewrite — respond in that shape, with a clear heading noting it is not a usable SOP.
- You detected a decomposition signal — return the requested SOP first, then a one-line **Note to user** above it offering the split.

Keep SOPs concise — a great SOP fits on a screen or two. If yours is much longer, you're probably writing two SOPs as one (see the decomposition signal above).

## Reading the request

1. **Find the current SOP** in the conversation. If one is present, treat it as authoritative — preserve anything the user did not ask to change. If absent, the user is drafting from a brief; produce the full document from scratch.
2. **Determine intent from wording.** Full rewrite, targeted fix, draft-from-brief, or critique. Match the output shape to the intent — do not return a critique when asked for a rewrite, and do not return a rewrite when asked "tell me what's wrong with this".
3. **Detect queue context.** Look for keywords (queue, case, postpone, handover, reminder, follow-up, SLA, daily, hourly) or the presence of any `_case` / `request_handover` tool name in the current SOP. If detected, apply the case lifecycle patterns; otherwise treat the Assignment as standalone.

You have no access to the user's data, Connections, Files, or Assignments. Write from the SOP text and request in the conversation only. Do not invent tool calls, integration names, or product features.

## Final check before returning

Walk through this list once on your draft. Fix anything that fails before returning the SOP to the user.

- [ ] Structure: starts with `# GOAL` (one sentence), then `# STEPS` (numbered), `# NOTES` only if needed.
- [ ] Single-case framing: no batch loops, no "for each", no "all pending".
- [ ] Every step is an imperative business action; tools referenced by Connection name.
- [ ] Every decision point has a concrete threshold or rule — no "use your judgment".
- [ ] Every branch terminates: queue Assignments end in a case lifecycle tool; standalone Assignments end in a concrete action.
- [ ] Duvo terminology used consistently (Assignment, Job, SOP, Connection, Files, Login, Start Work, Setup).
- [ ] No separate Preconditions / Escalation / Error Handling sections — everything inlined.
- [ ] No hardcoded team-specific data that should live in Files or Setup.
- [ ] If a decomposition signal is present, a one-line **Note to user** mentions the split option.

See `references/sop-examples.md` for canonical worked examples to calibrate against before returning.

## See also

- `job-debugger` — when a Job ran with an SOP and failed, start there to diagnose the root cause; it will hand off to this skill if the fix is an SOP rewrite.
- `duvo-cli` — once the SOP is finalized, `duvo revisions create` / `duvo revisions update` ships it from the terminal.

## Resources

- [Duvo](https://duvo.ai) — product website
- [Duvo documentation](https://docs.duvo.ai) — building Assignments, SOPs, Connections, queues
- [Web app](https://app.duvo.ai) — author Assignments and watch Jobs in the browser
- [Duvo CLI (`@duvoai/cli`)](https://www.npmjs.com/package/@duvoai/cli) — script SOP-driven Assignments from the terminal; pairs with the `duvo-cli` skill
- [Public skill repository](https://github.com/duvoai/skills) — the MIT-licensed community release of this skill, packaged for installation in third-party Claude Code setups
