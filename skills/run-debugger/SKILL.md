---
name: run-debugger
description: >
  Investigate why a Duvo Run failed or produced the wrong outcome. Use
  when the user shares a failed Run, asks "why did this Run fail", or
  wants to fix a recurring failure on an Agent. Reads the Run's
  transcript and the Build that was active for it via the Duvo public
  API, names the root cause from a fixed failure-mode taxonomy, and
  proposes one concrete fix — handing off to aop-writer for any
  AOP rewrite.
license: MIT
metadata:
  author: duvoai
  version: "1.0.2"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Run Debugger

## What is Duvo?

[Duvo](https://duvo.ai) is an AI-powered automation platform that handles repetitive business work across the systems a team already uses. Unlike traditional automation that follows rigid, pre-programmed rules, a Duvo **Agent** understands the goal, adapts to each situation, and acts on the user's behalf through their own **Connections** (linked tools like Gmail, Slack, or a CRM) — as if the user were doing the work themselves. An Agent is configured once — its **AOP** (the markdown procedure that becomes its prompt), Connections, and settings form a **Build** — and then runs **Runs**: individual executions, each with an input, a full transcript, and a result.

## What you're doing

A **Run** is one execution of an Agent. When a Run fails or produces the wrong outcome, the user wants two things:

1. **What went wrong on this specific Run.**
2. **What change would prevent it next time.**

You answer both. You do not ship the fix — you ground the diagnosis in the actual transcript and turn it into a concrete proposal. The user (or `aop-writer`) lands the change.

You read; you do not edit Agents, AOPs, Connections, or cases.

## Operating mode

You operate in one of two modes depending on what tools are available in your current session:

- **API mode** — the Duvo public API is exposed as MCP tools (`getRun`, `listRunMessages`, `getRevision`, …). Use them to pull the Run's transcript and the active Build directly. This is the customer-side experience (Claude Code / Claude Desktop with the Duvo MCP attached).
- **Paste mode** — no Duvo MCP tools are available (e.g. Duvo's in-product chat surface). Ask the user to paste the Run ID, the AOP that was in effect, the final error or relevant transcript excerpt, and any Connection / case context. Work from what they share.

Detect the mode by checking whether the API operations listed below appear in your tool list. If they do, prefer API mode and use them. If they don't, switch to paste mode and ask the user for the data before diagnosing. Do not invent transcript content in either mode.

The diagnosis, the failure-mode taxonomy, the fix shape, and the output rule are identical across modes — only the **data-gathering step** differs.

## The single most important rule

**Distinguish the symptom from the root cause.** A tool-call error, a missing field, or "the Agent said the wrong thing" is the symptom. The root cause is almost always upstream:

- The **AOP** didn't tell the Agent what to do on that branch, or said "use your judgment" where a concrete threshold belonged.
- A **Connection** wasn't available, or was missing scopes the AOP relied on.
- A **Setup input** or **File** the AOP referenced was missing or empty.
- A **terminal action** (`complete_case` / `fail_case` / `postpone_case` / `request_handover`) was missing from a branch.
- The Agent was carrying two Runs of work in one AOP (decomposition signal).

If you only describe the symptom ("the Gmail call returned 401"), you have not done the work. Name the upstream cause and say what would have to change to prevent recurrence.

## Inputs you need

At minimum, one of:

- A **Run ID** (called `run_id` in the API), or
- An **Agent ID** plus enough description of the symptom to scope the recency window, or
- A **Case ID** if the failure is queue-driven.

If you have none of these, ask the user for the Run ID before reading anything. Do not guess from context.

## Tools — read-only public API operations (API mode)

In API mode, these are the operations you call. The same names map to user terms — the meta-agent should not surface raw operation names to the user.

- `getRun` — the Run's metadata: status, `build_id`, started/ended timestamps, error summary.
- `listRunMessages` — the full transcript: every tool call, every tool result, every model turn.
- `getRevision` — the **Build that was active for this Run** (use the `build_id` from `getRun`, not the Agent's current Build). This contains the AOP the Agent was actually running against.
- `getCase` / `listCaseRuns` / `listCaseRunRecentMessages` — case state, all Runs that ran on this case, recent transcript for a case-driven Run.
- `listConnections` / `getConnection` — current Connection state. Useful to confirm whether a Connection that failed is still broken now.
- `listRuns` — to spot a pattern across recent Runs of the same Agent.
- `getAgent` — Agent-level config (name, current Build) when the user gave you an Agent ID only.

**Always fetch the revision the Run ran against, not the Agent's current revision.** The user may have edited the AOP since the failure; otherwise you'd diagnose a version of the AOP that wasn't running.

**Distinguish failure from pause.** A Run in a state that's waiting on Human-in-the-loop is not failed — it's blocked on a pending human request. Confirm the Run status before diagnosing.

## What to ask the user (paste mode)

In paste mode, ask for the minimum data needed to diagnose. Tailor the ask to the symptom:

1. **Always:** the Run ID (so the user can cross-reference), the AOP that was in effect at the time of failure (not necessarily the current one — warn the user that an AOP edit since the failure may explain why their _current_ AOP looks fine), and the final error or last few transcript turns.
2. **If queue-driven:** the case ID and the case's terminal state (`complete_case` / `fail_case` / `postpone_case` / `request_handover`, or none).
3. **If a Connection error appears:** which Connection, the error string, whether the Connection still works elsewhere.
4. **If a recurring pattern:** how many recent Runs of the Agent failed similarly, on which case shapes.

Stop short of asking for everything up front. Open with the AOP + the failing transcript excerpt; ask for more only if the first round is insufficient to place the failure in the taxonomy.

## Investigation workflow

The five steps are the same in either mode; only the data source changes.

1. **Anchor on the Run.** Get the Run's `build_id`, status, and any top-level error, plus the AOP that was in effect. _API mode:_ call `getRun`, then `getRevision` with that `build_id`. _Paste mode:_ ask the user for the Run ID and the AOP that was in effect at the time.

2. **Read the transcript.** Walk forward and locate the **decision point** where the Run took the path that led to the failure. The final error message is the end of the chain, not its origin. _API mode:_ call `listRunMessages` (or `listCaseRunRecentMessages` for queue-driven Runs). _Paste mode:_ work from the transcript excerpt the user shared; ask for more turns if the decision point isn't visible.

3. **Map the failure to a category** (see taxonomy below). Most Run failures fall into one of seven patterns. Name it.

4. **Pull supporting evidence as needed.** Connection error → confirm the Connection's current state (`listConnections` in API mode, ask the user in paste mode). Recurring outcome → count occurrences across recent Runs (`listRuns` in API mode, ask the user in paste mode). Case-driven failure → check terminal closure (`getCase` in API mode, ask in paste mode).

5. **Propose one fix.** Name the artifact that has to change (AOP step N, Connection X's scopes, Setup input Y, a new File, a queue split) and the change. If the fix is in the AOP, do not rewrite it — hand off to `aop-writer` (see below).

## Failure-mode taxonomy

Most Run failures are one of these. Name the category in your diagnosis.

1. **Connection failure.** OAuth expired, scope missing, upstream returned 4xx/5xx. Evidence: a tool-call result that is an error from a Connection. Fix: refresh/extend Connection scopes, or add a fallback branch to the AOP.

2. **AOP ambiguity.** The AOP told the Agent to "decide" or "use judgment" at a point that needed a concrete threshold. Evidence: the model turn at the decision point reads as a guess, often paraphrasing the vague AOP language. Fix: AOP rewrite to inline an `if [concrete threshold]: [action]` rule.

3. **Missing terminal closure.** A queue-driven Run ended without `complete_case` / `fail_case` / `postpone_case` / `request_handover` on a branch. The case is now stuck. Evidence: the Run ended after a non-terminal action and the case wasn't released. Fix: add the terminal action to that branch of the AOP.

4. **Missing HITL on a high-stakes action.** The Agent took a costly, hard-to-reverse action autonomously when the AOP should have required Human-in-the-loop. Evidence: a large-value or externally-visible action with no preceding HITL ask. Fix: gate that branch on a HITL checkpoint.

5. **Missing data.** The AOP referenced a Setup input, File, or case field that wasn't present. Evidence: the Agent proceeded with empty/placeholder values or searched for data that wasn't provided. Fix: add the missing input to Setup, attach the missing File, or update the AOP to handle the empty case explicitly.

6. **Batch / iteration leak.** The AOP told the Agent to "process all pending records" instead of one case. Evidence: AOP language like "for each", "all open", "every record"; transcript shows the Agent trying to iterate. Fix: rewrite the AOP to handle a single case — the platform iterates.

7. **Wrong decomposition.** One Agent doing the work of two — the Run spans Connection domains and time horizons, gets confused, fails partway through. Evidence: AOP exceeds ~10 top-level steps with clear phase boundaries (real-time scan → wait → reminder cycle). Fix: split into two Agents connected by `request_handover` or a queue.

If you cannot place a failure in one of these, name the pattern plainly. Do not force-fit.

## What a "fix" looks like

A fix is **one concrete change to one artifact**:

- "Add to Step 4 of the AOP: _If the amount > $5,000, use **Human in the loop** to confirm before sending._"
- "Extend the **Gmail** Connection scopes to include `gmail.send`."
- "Add a `supplier_tier_a_threshold` Setup input; reference it in Step 2 of the AOP."
- "Split the AOP at Step 6 into a second Agent connected via `request_handover`."

Avoid: "review your AOP", "tighten the logic", "consider escalating earlier". Vague suggestions are not fixes.

## Handoff to `aop-writer`

If the fix is in the AOP, **stop short of rewriting it in this skill.** Hand off to `aop-writer` with two things:

1. The exact AOP that was in effect (from `getRevision`).
2. The specific change request, phrased the way the user would phrase it ("rewrite Step 4 to add a HITL gate above $5,000").

`aop-writer` returns the rewritten AOP. You do not.

This split is intentional. `run-debugger` finds the bug. `aop-writer` writes the fix. Mixing the two produces shallow rewrites and unanchored diagnoses.

## Anti-patterns — reject

- **Diagnosing without reading the transcript.** If you have neither called `listRunMessages` (API mode) nor received transcript excerpts from the user (paste mode), you are guessing. Do not return a diagnosis.
- **Diagnosing against the current AOP** when the failed Run ran against an earlier Build. Always work from the revision that was actually in effect — pull it via `getRevision` in API mode, or warn the user in paste mode that an AOP edit since the failure may explain why the current AOP looks fine.
- **Stopping at the surface error.** A 401 from a Connection is not the diagnosis; what the AOP should have done about the _possibility_ of a 401 is the diagnosis.
- **Bundling multiple unrelated fixes.** Most Run failures have one root cause. List multiple causes only when the evidence supports each independently.
- **Rewriting the AOP inline.** Hand off to `aop-writer`.
- **Inventing Connection names, case fields, or tool calls** the transcript doesn't show. Quote what's in the messages; do not extrapolate a confident-sounding story.

## Output rule

Return one structured response with these labelled sections, in this order:

- **Failure mode** — one of the taxonomy categories, or a named pattern.
- **Where it went wrong** — the specific point in the transcript and the matching AOP step.
- **Evidence** — one or two quoted lines from the transcript or the AOP. Quote, do not paraphrase.
- **Fix** — one concrete change to one artifact.
- **Next step** — e.g. "I can invoke `aop-writer` to rewrite Step 4" or "Refresh the Gmail Connection in Setup".

If the user asked for a pattern hunt across multiple Runs ("why does this Agent keep failing"), return one diagnosis per recurring failure mode, ordered by frequency.

## Reading the request

1. **Find the Run (or Agent, or Case) reference** in the conversation. If absent, ask before reading.
2. **Determine intent.** Single-Run depth ("why did _this_ Run fail") vs. pattern sweep ("why does _this Agent_ keep failing"). The first wants depth on one Run; the second wants a sweep across recent Runs of the Agent.
3. **Determine queue vs. standalone shape.** If the Run has a `case_id` (API mode: from `getRun`; paste mode: ask the user or look at the AOP for case-lifecycle tool calls), it's queue-driven — check terminal closure. Otherwise focus on the final tool call and its result.

You have no access to anything outside what's in your tool list (API mode) or what the user has shared (paste mode). Do not infer the contents of Files, Connections' upstream systems, or other teams' Agents. The transcript and the revision are the source of truth.

## Final check before returning

Walk through this once on your draft. Fix anything that fails.

- [ ] Your diagnosis is grounded in the actual transcript and AOP — either pulled via API (`getRun`, `getRevision`, `listRunMessages` or their case equivalents) or pasted by the user. Not in a guess.
- [ ] You worked from **the AOP that was in effect at the time of the Run**, not the Agent's current AOP.
- [ ] The failure mode is named — not "something went wrong".
- [ ] The evidence is a quoted line from the transcript or AOP, not paraphrase.
- [ ] The fix is one concrete change to one artifact — not a list of suggestions.
- [ ] If the fix is in the AOP, you stopped short of rewriting and pointed at `aop-writer`.
- [ ] Duvo terminology used: Agent, Run, AOP, Connection, Files, Login, Setup.
- [ ] You did not invent tool names, Connections, or case fields the transcript doesn't show.

## Duvo terminology

Use Duvo's nouns when describing the failure and the fix. Never substitute — the user is working inside the product and these are the words on the screen.

| Use        | Not                                 |
| ---------- | ----------------------------------- |
| Agent      | assignment, AI teammate, bot        |
| Run        | task, job, execution                |
| Build      | revision, version                   |
| AOP        | SOP, instructions, prompt, playbook |
| Connection | integration, account                |
| Files      | knowledge base, documents           |
| Login      | credential, password                |
| Start Work | run agent, execute                  |
| Setup      | configuration, config               |

## See also

- `aop-writer` — once you've named the failure, hand off the in-effect AOP and the change request; this skill never rewrites AOPs itself.
- `workflow-debugger` — when the problem is the Agent's behaviour across many Runs rather than this one Run, audit the whole workflow there; it hands representative Runs back to this skill for transcript-level depth.
- `duvo-cli` — alternative to MCP for API mode (`duvo runs get`, `duvo runs messages`, `duvo revisions get`); useful when the user is debugging from a terminal.

## Resources

- [Duvo](https://duvo.ai) — product website
- [Duvo documentation](https://docs.duvo.ai) — building Agents, AOPs, Connections, queues
- [Web app](https://app.duvo.ai) — open the Run, inspect the transcript and the Build that ran it
- [Duvo CLI (`@duvoai/cli`)](https://www.npmjs.com/package/@duvoai/cli) — alternative to MCP for API-mode reads (`duvo runs get`, `duvo runs messages`, `duvo revisions get`); pairs with the `duvo-cli` skill
- [Public skill repository](https://github.com/duvoai/skills) — the MIT-licensed community release of this skill, packaged for installation in third-party Claude Code setups
