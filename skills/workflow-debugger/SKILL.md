---
name: workflow-debugger
description: >
  Audit a Duvo Assignment — or a multi-Assignment workflow connected by a
  Case Queue — across many Jobs to find systemic inefficiencies and quality
  issues, then recommend concrete SOP and architecture changes. Use when the
  user asks to "analyze this workflow", "audit my Assignment", "why is this
  Assignment slow / inconsistent / low quality across runs", "why does my
  queue keep backing up", or wants a health check over an Assignment's recent
  Jobs — as opposed to debugging one failed Job (that's job-debugger). Reads
  recent Jobs, eval scores, the producer/consumer queue topology, and the SOPs
  those Jobs actually ran against via the Duvo public API; hands off to
  sop-writer for any SOP rewrite.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Workflow Debugger

## What is Duvo?

[Duvo](https://duvo.ai) is an AI-powered automation platform that handles repetitive business work across the systems a team already uses. Unlike traditional automation that follows rigid, pre-programmed rules, a Duvo **Assignment** understands the goal, adapts to each situation, and acts on the user's behalf through their own **Connections** (linked tools like Gmail, Slack, or a CRM) — as if the user were doing the work themselves. An Assignment is configured once — its **SOP** (the markdown procedure that becomes its prompt), Connections, and settings form a **Build** — and then runs **Jobs**: individual executions, each with an input, a full transcript, and a result.

## What you're doing

A **workflow** is an Assignment running many Jobs over time — and often a _pair_ of Assignments connected by a **Case Queue**: a **producer** that pushes cases in, and a **consumer** that is triggered to work them. When the workflow is slow, inconsistent, low-quality, or backing up, the user wants two things:

1. **What the workflow is doing inefficiently, with evidence across its Jobs.**
2. **What changes would make it faster, cheaper, or more reliable next time.**

You answer both. You analyse the workflow's **aggregate** behaviour across many Jobs — not the transcript of any single Job — and turn what the run data shows into concrete proposals. You do not ship the change: the user (or `sop-writer`) lands it.

You read; you do not edit Assignments, SOPs, Connections, queues, or cases.

## job-debugger vs workflow-debugger

These two are complementary — pick the right one, and use them together.

- **`job-debugger`** diagnoses **one Job** that failed or produced the wrong outcome, grounded in that Job's transcript and the Build it ran. Use it when the user points at a specific Job.
- **`workflow-debugger`** (this skill) audits the **whole Assignment across many Jobs** — or a producer→consumer pair — grounded in run-list aggregates, eval scores, the queue topology, and the SOPs those Jobs ran against. Use it when the user wants a health check, an efficiency audit, or asks why the Assignment behaves badly _in general_.

If the sweep surfaces a recurring failure that needs transcript-level depth, hand a representative Job to `job-debugger`. If the user only has one bad Job, start with `job-debugger`.

## Operating mode

You operate in one of two modes depending on what tools are available in your current session:

- **API mode** — the Duvo public API is reachable, either as MCP tools (`listRuns`, `getRevision`, `listQueueAgents`, …) or via the `duvo` CLI (`@duvoai/cli`). Both hit the same public API; use whichever is in front of you to pull the run set, the topology, and the SOPs directly. This is the normal mode in Claude Code / Claude Desktop with the Duvo MCP attached, or in a terminal with `duvo` installed.
- **Paste mode** — no Duvo API access (e.g. an offline review of a workflow). Ask the user to paste the recent Job list (status, case titles, eval scores), the producer/consumer setup, and the SOPs in effect. Work from what they share.

Detect the mode by checking whether the operations below appear in your tool list (or whether `duvo` is on PATH). If so, prefer API mode. If not, switch to paste mode and ask for the data before diagnosing. Do not invent run data, eval scores, or SOP content in either mode.

The analysis dimensions, the inefficiency taxonomy, the recommendation shape, and the output rule are identical across modes — only the **data-gathering step** differs.

## The single most important rule

**Ground every claimed inefficiency in counts across the run set — not in a single Job and not in a hunch.** "This Assignment over-escalates" is only a finding if you can say _how many_ of the recent Jobs escalated. One failed Job is a `job-debugger` question; a pattern is "N of the last 50 Jobs did X". If you cannot quantify it from the data you pulled, say so rather than asserting it.

Two corollaries:

- **Analyse the SOP the Jobs actually ran against**, identified by the `build_id` carried on recent Jobs — not a nominal "live" label. There is no "live revision" filter in the API; the Build that recent Jobs executed is the honest answer. (The current live Build is usually the highest `revision_number`, but a promotion can repoint it, so trust the `build_id` on real Jobs.)
- **Distinguish a symptom from its cause.** "Eval pass rate is low" is the symptom. The cause is almost always one SOP gap repeated every run, a miscalibrated threshold, a missing terminal action, or a topology mismatch. Name the cause.

## Inputs you need

At minimum, one of:

- An **Assignment ID** (the Assignment to audit), or
- A **Case Queue ID** (to audit the producer/consumer workflow around it).

From either you can derive the rest — the queue from the Assignment's Jobs, the partner Assignments from the queue. If you have neither, ask the user before reading anything. Do not guess from context.

## Tools — read-only public API operations (API mode)

In API mode these are the operations you call. Each maps to a `duvo` CLI command for terminal users; the MCP tool names are listed first.

- `listRuns` — recent Jobs for the Assignment, with status, `build_id`, `case_*` fields, timestamps, and `eval_summaries`. CLI: `duvo runs list --agent <id> --limit 50 --json` (the envelope is `{ data: [...], total }`; `--limit` max is 100).
- `getRevision` — a single Build, including its `config` (which holds the SOP). Pass the `build_id` from recent Jobs. CLI: `duvo revisions get <build-id> --agent <id> --json`.
- `listAgentRevisions` — the Assignment's Build history (`revision_number`, timestamps). CLI: `duvo revisions list --agent <id> --json`.
- `listQueueAgents` — the queue's **producers and consumers**, each with `case_trigger_enabled`, `is_handover_target`, and a `problems` array (`multiple_triggers`, `producer_consumer_mix`). CLI: `duvo queues agents <queue-id> --json`.
- `listAgentCaseTriggers` — which queue(s) trigger this Assignment (the consumer binding). CLI: `duvo agents case-triggers list <agent-id> --json`.
- `getAgent` — Assignment-level metadata (name, delivery settings). CLI: `duvo agents get <id> --json`.
- `getCase` / `listCaseRuns` — a single case's state and every Job that has worked it, when you need to confirm a case is bouncing rather than closing.

**Use the run set as the source of truth.** Status mix, eval scores, case-title variety, and `build_id` all come from `listRuns` — start there, and only fetch SOPs and topology once the run data tells you where to look.

## What to ask the user (paste mode)

In paste mode, ask for the minimum needed to find the pattern:

1. **Always:** the recent Job list — status, case title, and eval score per Job (`duvo runs list --agent <id> --limit 50 --json` if they have the CLI), plus the SOP(s) in effect.
2. **If queue-driven:** the producer and consumer Assignments and their case triggers (`duvo queues agents <queue-id> --json`).
3. **If a quality complaint recurs:** the eval `final_comment` text across the affected Jobs.

Open with the run list and the SOP; ask for more only if the first round can't place the pattern in the taxonomy.

## Investigation workflow

The five steps are the same in either mode; only the data source changes.

1. **Pull the run set.** Get the last ~50 Jobs for the Assignment. _API mode:_ `listRuns` filtered to the Assignment / `duvo runs list --agent <id> --limit 50 --json`. _Paste mode:_ ask the user for the list.

2. **Profile the run set** (see dimensions below). Status breakdown, case-title variety, eval pass rate and severity, recurring `final_comment`, run frequency and timing, and which `build_id`(s) the Jobs ran against. Write down counts — these become your evidence.

3. **Map the topology.** Read the `case_queue_id` off the Jobs; that's the queue this Assignment consumes from. _API mode:_ `listQueueAgents` on that queue / `duvo queues agents <queue-id> --json` to get producers, consumers, and any `problems`; `listAgentCaseTriggers` to confirm the trigger binding. _Paste mode:_ ask the user who produces and who consumes. A standalone Assignment with no `case_queue_id` has no topology — skip this step.

4. **Read the SOPs the Jobs ran.** For the Assignment (and each partner), take the `build_id` from its recent Jobs and pull that Build's SOP. _API mode:_ `getRevision(build_id)` / `duvo revisions get <build-id> --agent <id> --json` — the SOP is in `config`. _Paste mode:_ ask the user to paste the SOP that was in effect. Read producer and consumer SOPs together — many workflow problems live at the seam between them.

5. **Synthesise the report.** Place the top issues in the taxonomy, attach the counts and quotes that prove each, and propose one concrete change per issue. SOP changes hand off to `sop-writer`; topology changes are described as an architecture suggestion.

## What to profile across the run set

Each dimension maps to a field on the Jobs from `listRuns`. Quantify, don't eyeball.

- **Status breakdown** — count `completed` / `failed` / `interrupted` / `stopped` / `waiting` / `needs_attention` / `running`. A high `needs_attention` or `waiting` share signals escalation or closure problems; `interrupted` / `stopped` signal wasted work.
- **Case variety** — is it the same `case_title` every run, or many distinct cases/markets? Repetition of one title across Jobs means a case that won't close; wide variety means real throughput.
- **Eval scores** — for each Job's `eval_summaries`: is `passed < total`? Read `severityCounts` (`critical` / `medium` / `low`). Cluster by severity — a recurring `critical` is the headline.
- **Recurring eval comments** — the same `final_comment` complaint across many Jobs is the single strongest signal of a prompt issue: the SOP is producing the same defect every run.
- **Frequency and timing** — Job cadence from `created_at` / `started_at`, and duration from `started_at` → `completed_at`. A steady drumbeat of tiny near-identical Jobs hints at batching or scheduling; long durations hint at a monolithic SOP.
- **Build spread** — are recent Jobs on one `build_id` or several? A change in behaviour around a Build boundary points at an SOP edit as the cause.

## Inefficiency taxonomy

Most workflow problems are one of these. Name the category, and back it with counts.

1. **Recurring quality gap (eval-driven).** Many Jobs share the same `final_comment` and `passed < total`, often at one severity. Cause: a single SOP gap producing the same defect every run. Evidence: count of Jobs with that complaint + their severity. Fix: the SOP line that omits the criterion.

2. **Cases that don't close (terminal-closure leak).** The same `case_title` reappears across many Jobs; cases bounce via postpone/re-pickup without `complete_case` / `fail_case`. Evidence: repeated case title + a `waiting` / `needs_attention` skew. Fix: add the missing terminal action to that SOP branch.

3. **Escalation miscalibration (HITL).** Over-escalation — a large `needs_attention` / `waiting` share where the SOP should decide autonomously; or under-escalation — costly autonomous actions with no Human-in-the-loop gate. Evidence: the status mix vs. the SOP's decision rules. Fix: tune the threshold in the SOP.

4. **Serial work that should be batched or scheduled.** Many tiny Jobs at high cadence doing near-identical work, or a fixed drumbeat that should be a schedule. Evidence: run frequency + case-title sameness. Suggestion: batch through the queue, or move to a scheduled trigger.

5. **Producer/consumer imbalance or topology problem.** `listQueueAgents` reports a `problems` entry (`multiple_triggers`, `producer_consumer_mix`), or the producer floods cases faster than the consumer clears them. Evidence: the `problems` array + run volume per side. Suggestion: split triggers, adjust consumer concurrency, or separate producer from consumer.

6. **Monolithic Assignment (decomposition signal).** One Assignment's SOP spans Connection domains and distinct cadences; its Jobs run long and fail at the seams. Evidence: SOP length/phase boundaries + a spread of unrelated failure modes in one Assignment. Suggestion: split into a producer→consumer pair via a Case Queue or `request_handover`.

7. **Wasted Jobs.** `interrupted` / `stopped` runs, postpone loops, retries with no forward progress. Evidence: status mix + repeated `build_id` with no completion. Fix: an SOP early-return or guard so the Assignment stops doing no-op work.

If a problem doesn't fit, name the pattern plainly. Do not force-fit.

## What a recommendation looks like

A recommendation is **one concrete change to one artifact**, with the evidence that motivates it:

- "12 of the last 50 Jobs failed eval with the comment _'did not include the PO number'_ (all `medium`). **Quote the SOP line to change:** _'Reply to the supplier with the delivery status.'_ → it should require the PO number. Hand to `sop-writer`."
- "The same case _'Reorder SKU-4471'_ appears in 9 Jobs, all `waiting`, never `completed`. Step 5 of the consumer SOP has no `complete_case` on the in-stock branch. Add it."
- "`listQueueAgents` reports `producer_consumer_mix` on Assignment X — it both fills and drains the queue. Split it into two Assignments."
- "38 of 50 Jobs ran < 20s on near-identical single-SKU cases. Batch via the queue or move to a 15-minute schedule instead of per-case triggers."

Avoid: "tighten the SOP", "improve quality", "consider batching". A recommendation the user can't act on verbatim is not a recommendation. When the fix is in the SOP, **quote the exact line to change** — the user asked for that specificity.

## Handoff to `sop-writer`

When a recommendation is an SOP change, **stop short of rewriting the SOP here.** Hand off to `sop-writer` with two things:

1. The exact SOP that was in effect (from `getRevision` on the `build_id` the Jobs ran).
2. The specific change request, phrased the way the user would ("rewrite Step 5 to require the PO number in the supplier reply").

`sop-writer` returns the rewritten SOP. You do not. This split is deliberate: `workflow-debugger` finds the systemic issue; `sop-writer` writes the fix. Mixing the two produces shallow rewrites and unanchored audits.

## Anti-patterns — reject

- **Auditing without pulling the run set.** If you have neither called `listRuns` (API mode) nor received the Job list from the user (paste mode), you are guessing. Do not return findings.
- **Calling one or two Jobs a pattern.** A finding needs a count across the run set. Two bad Jobs is a `job-debugger` question, not a workflow inefficiency.
- **Auditing the wrong SOP** — the Assignment's current Build when recent Jobs ran an earlier one. Read the Build the Jobs actually executed (`build_id`).
- **Reading only one side of a queue.** Producer and consumer SOPs must be read together; the problem is often the seam between them.
- **Inventing run counts, eval comments, or SOP lines** the data doesn't show. Quote what's there; report a gap as a gap.
- **Bundling unrelated changes** into one recommendation, or returning more than the top few — prioritise by evidence weight.
- **Rewriting the SOP inline.** Hand off to `sop-writer`.

## Output rule

Return one structured report with these labelled sections, in this order:

- **What the workflow does** — one sentence.
- **Top inefficiencies** — up to three, ordered by evidence weight. Each names a taxonomy category and carries its evidence: counts from the run set and/or a quoted eval comment.
- **Prompt changes** — for each SOP-level fix, name the artifact (producer or consumer SOP, the step) and **quote the exact line to change**.
- **Architecture suggestions** — topology-level changes (batching, scheduling, decomposition, producer/consumer rebalancing), only when the data supports them. Omit the section if there are none.
- **Next step** — e.g. "I can invoke `sop-writer` to rewrite Step 5 of the consumer SOP", or "Hand Job `<id>` to `job-debugger` for the transcript-level cause".

If the user asked only about one dimension ("is this Assignment over-escalating?"), answer that dimension with its counts and skip the rest.

## Reading the request

1. **Find the Assignment or Case Queue reference** in the conversation. If absent, ask before reading.
2. **Determine scope.** Single Assignment ("audit this Assignment") vs. workflow ("why does this queue back up", "analyse this producer→consumer flow"). The first profiles one Assignment's Jobs; the second adds the topology and reads both SOPs.
3. **Determine the lens.** Efficiency (speed, cost, wasted Jobs, batching) vs. quality (eval scores, recurring defects) vs. reliability (closure, escalation). Lead with the lens the user named; surface the others only if the data makes them unavoidable.

You have no access to anything outside your tool list (API mode) or what the user shared (paste mode). Do not infer the contents of Files, Connections' upstream systems, or Jobs you didn't pull. The run set, the topology, and the SOPs are the source of truth.

## Final check before returning

Walk through this once on your draft. Fix anything that fails.

- [ ] Every finding is grounded in counts across the run set, pulled via the API/CLI or pasted by the user — not a single Job and not a hunch.
- [ ] You read the SOP the Jobs **actually ran** (`build_id`), not just the current Build.
- [ ] For a queue workflow, you read **both** producer and consumer SOPs and checked `listQueueAgents` `problems`.
- [ ] Each inefficiency is named from the taxonomy and carries its evidence.
- [ ] Each prompt change **quotes the exact SOP line** to change and names the artifact.
- [ ] SOP rewrites are handed to `sop-writer`, not written here.
- [ ] Duvo terminology used: Assignment, Job, Build, SOP, Connection, Case Queue, Files, Setup.
- [ ] You did not invent run counts, eval comments, SOP lines, or topology the data doesn't show.

## Duvo terminology

Use Duvo's nouns when describing the workflow and the fix. Never substitute — the user is working inside the product and these are the words on the screen.

| Use        | Not                            |
| ---------- | ------------------------------ |
| Assignment | agent, AI teammate, bot        |
| Job        | task, run, execution           |
| Build      | revision, version              |
| SOP        | instructions, prompt, playbook |
| Connection | integration, account           |
| Case Queue | queue, backlog                 |
| Files      | knowledge base, documents      |
| Setup      | configuration, config          |

## See also

- `job-debugger` — for one failed Job: it reads the transcript and the Build that ran it. This skill audits the whole workflow; hand it a representative Job when a pattern needs transcript-level depth.
- `sop-writer` — once you've named an SOP-level fix, hand off the in-effect SOP and the change request; this skill never rewrites SOPs itself.
- `duvo-cli` — the terminal surface for every read here (`duvo runs list`, `duvo queues agents`, `duvo revisions get`); useful when the user is auditing from a shell.

## Resources

- [Duvo](https://duvo.ai) — product website
- [Duvo documentation](https://docs.duvo.ai) — building Assignments, SOPs, Connections, Case Queues
- [Web app](https://app.duvo.ai) — open the Assignment, inspect its Jobs, evals, and the Build that ran them
- [Duvo CLI (`@duvoai/cli`)](https://www.npmjs.com/package/@duvoai/cli) — the read commands this skill relies on in API mode; pairs with the `duvo-cli` skill
- [Public skill repository](https://github.com/duvoai/skills) — the MIT-licensed community release of this skill, packaged for installation in third-party Claude Code setups
