---
name: duvo-sop-writer
description: >
  Draft, rewrite, or critique a Duvo Assignment SOP (Standard Operating
  Procedure). Use when the user wants to improve an existing SOP or write a
  new one from a brief. Returns the SOP as a single complete markdown document
  in the canonical GOAL / STEPS / NOTES shape.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Duvo SOP Writer

## What is Duvo?

[Duvo](https://duvo.ai) is an AI platform that designs, runs, and monitors
autonomous agents (called **Assignments**) for business process automation.
Each Assignment is driven by an **SOP** — a markdown instruction document that
determines how the agent behaves on every Job (execution).

## What an SOP is

An SOP is the instruction document attached to an Assignment. At Job-start time
it is concatenated into the Assignment's prompt, so its quality directly
determines how the Assignment behaves — against real Connections (integrations),
with real escalation consequences. You are writing operational instructions that
an LLM-driven agent will execute on live data.

## The single most important rule

**An SOP is designed around processing ONE case/item from start to finish.**
All decision branches, all error handling, all terminal states, in one document.
The Duvo platform handles iteration (picking up the next case, retrying,
scheduling). Never write batch loops, "for each item" wrappers, or "process all
pending records".

## Canonical structure

Every SOP uses three sections:

- **`# GOAL`** — one sentence describing what the Assignment does on a single
  case and the expected outcome.
- **`# STEPS`** — a numbered list of imperative actions. Inline everything
  (data formats, decision criteria, early returns, error handling, handover or
  completion conditions) into the step where it applies.
- **`# NOTES`** _(optional)_ — only for genuinely cross-cutting concerns
  (timezone conventions, naming patterns). Omit otherwise.

Do **not** create separate `PREREQUISITES`, `ERROR HANDLING`, `DATA HANDLING`,
`ESCALATION`, or `TRIGGER` sections.

## Patterns that produce excellent SOPs

1. **Reference tools by Connection name.** "Use **Gmail** to send the
   response." This tells the runtime which Connection (and tool) to use.

2. **Inline decision criteria with concrete thresholds.** Replace "use your
   judgment" with explicit `if ... then ...` rules. "If the amount exceeds
   $5,000, escalate."

3. **Inline early returns and error handling** with
   `If [condition]: [action]`.

4. **Human-in-the-loop checkpoints for high-stakes actions.** "If the contract
   value exceeds $100,000: use **Human in the loop** to get approval before
   sending."

5. **The postpone-then-retry idiom for SLAs.** On first pickup, set a flag and
   `postpone_case` with the wait duration; on second pickup, do the real work.

6. **Terminal closure on every branch.** For queue Assignments, every branch
   must end with `complete_case`, `fail_case`, `postpone_case`, or
   `request_handover`. For standalone Assignments, every branch must end with
   a clear terminal action.

## Case lifecycle (queue-driven Assignments)

If the Assignment consumes or produces cases, the SOP must reference these
tools by name in the step where they apply:

- `claim_case`, `update_case`, `postpone_case`, `complete_case`, `fail_case`
- `add_cases`, `list_cases` (producers)
- `request_handover` (pass case to a different Assignment)

## Anti-patterns to avoid

- **Vague outcomes** ("handle the request appropriately") — use concrete
  success conditions
- **No decision criteria** ("use your best judgment") — use explicit thresholds
- **Batch / iteration logic** ("for each pending invoice") — write single-case
  SOPs
- **Missing terminal closure** — every branch must terminate
- **Hardcoded team data** — put specific values in Files or Setup inputs
- **Internal jargon** ("API call", "webhook") — describe business actions
- **Steps that say "how" instead of "what"** ("call `create_order` with
  body=...") — describe the business action; the agent picks the tool

## Duvo terminology

Use these terms consistently:

| Term           | Not                          |
| -------------- | ---------------------------- |
| Assignment     | agent, AI teammate           |
| Job            | task, run                    |
| SOP            | instructions                 |
| Connection     | integration                  |
| Files          | knowledge base               |
| Start Work     | run agent                    |
| Setup          | configuration                |
| Login          | credential                   |

## Output rule

Return a single complete SOP in markdown, in the canonical
`# GOAL` / `# STEPS` / `# NOTES` shape. Not a diff. Not a list of
suggestions.

## Final check before returning

- [ ] Starts with `# GOAL` (one sentence), then `# STEPS` (numbered),
  `# NOTES` only if needed
- [ ] Single-case framing: no batch loops, no "for each"
- [ ] Every step is an imperative business action; tools referenced by
  Connection name
- [ ] Every decision point has a concrete threshold — no "use your judgment"
- [ ] Every branch terminates with a case lifecycle tool or concrete action
- [ ] Duvo terminology used consistently
- [ ] No separate Preconditions / Escalation / Error Handling sections

## Resources

- [Duvo documentation](https://docs.duvo.ai)
- [Building Assignments](https://docs.duvo.ai/building-assignments)
- [Web app](https://app.duvo.ai)
