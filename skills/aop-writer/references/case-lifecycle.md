# Case Lifecycle in AOPs

Read this reference when the Agent is queue-driven — it has a `case-queue-producer` Connection, a `case-queue-consumer` Connection, or both. Standalone Agents (no queue role) ignore this file.

The case lifecycle tools are platform primitives whose **names matter to the runtime**. Reference them by name in the step where they apply — they are the one exception to the "describe what, not how" rule.

## Producer tools

Used by Agents with the `case-queue-producer` Connection.

- **`add_cases`** — enqueue one or more cases (1–100 per call). Each case has a `title` (required), and optional `data` and `labels`. Always batch when producing multiple cases in one Run; the platform expects up to 100 per call.
- **`list_cases`** — query an existing queue with filters (`title_search`, `status`, etc.). Use as a dedup helper before `add_cases` when the producer runs frequently enough to risk duplicates.

A producer AOP's terminal step is the `add_cases` call (or a no-op if there is nothing to enqueue this Run). Producers do not call `claim_case` / `complete_case`.

## Consumer tools

Used by Agents with the `case-queue-consumer` Connection. A consumer AOP processes one claimed case from start to finish.

- **`claim_case`** — fetch the case for this Run (and atomically claim it, if not already bound at Run start). **MUST be step 1** of a consumer AOP on the initial run, because case data is not injected into the prompt automatically. On a follow-up message in the same Run, skip this step — the case is already bound. A Run can only claim one case in its lifetime.
- **`update_case`** — write `title`, `data`, or `labels` back to the in-flight case. Add an `update_case` step only where the process needs to record progress, intermediate findings, or state that must survive a `postpone_case`.
- **`postpone_case`** — release the case and schedule a re-claim. The `postpone_to` parameter accepts a relative duration (`"2h"`, `"1d"`, `"1w"`) or an ISO timestamp. Use only for cases that _can_ complete later (waiting on a response, an SLA, a rate limit). Do NOT postpone for permanent or environment issues — use `fail_case` instead.
- **`complete_case`** — terminal success. Mark the case done after the AOP's work is finished.
- **`fail_case`** — terminal failure for permanent issues (missing data, broken credentials, unavailable tools).
- **`request_handover`** — pass the claimed case to a different Agent. This tool exists at run time only when the AOP configures a handover target by @-mentioning that Agent — see "Configuring the handover target" below.

## The postpone-then-retry idiom

This is the canonical pattern for any AOP step that says "wait N days then act" — SLAs, follow-up reminders, scheduled second touches.

Add a flag on the case the first time the AOP runs. On the next pickup, the flag is present and the AOP does the real work.

```markdown
2. If `initial_postpone_done` is NOT present in the case data (this is the first pickup):
   - Call `update_case` to set `initial_postpone_done: true` in the case data.
   - Call `postpone_case` with `postpone_to: "1d"` (or whatever the wait is).
   - Exit — do not continue to the steps below.

3. (Second pickup — `initial_postpone_done` is true.) Search the Slack thread for a PO reference. …
```

Use a different flag name (`reminder_sent`, `escalation_sent`) for each waiting phase if the AOP has more than one wait.

## Terminal closure rule

Every branch of a consumer AOP must end in **exactly one** of:

- `complete_case` — success.
- `fail_case` — permanent failure.
- `postpone_case` — temporary wait; the case returns to the queue.
- `request_handover` — pass the claimed case to a different Agent.

If a branch does not end in one of these calls, Duvo settles the case for the Agent — and the AOP no longer controls the result:

- The Run finished its work cleanly → the case is auto-completed (the activity feed marks it as such, with no explanation of the work) and its real outcome is left to Duvo's case evaluation. A branch that should have failed or postponed is now indistinguishable from a success until evaluation catches it.
- The Run ended with an error, was cut off mid-work, or requested a handover the platform rejected → the case is failed with no reason at all. (A Run a human stops is neither: its case is canceled.)

Neither is an acceptable ending to design for: both throw away the reason, and a missing terminal call is still the most common defect in queue-driven AOPs. Every branch must reach a terminal call.

## Handover vs. terminality

Handover and case terminality are **mutually exclusive**.

- If the AOP plans to call `request_handover` in a branch, do **not** also call `complete_case`, `fail_case`, or `postpone_case` in the same branch. The platform passes the claimed case to the target Agent automatically.
- Conversely, every consumer AOP that does **not** request a handover must end every branch with `complete_case`, `fail_case`, or `postpone_case`.

The `handover` Connection itself is added automatically at run time — do not list it in the Setup. But naming `request_handover` in prose does **not**, on its own, configure a handover: the tool only exists at run time once the AOP @-mentions the target Agent (see below).

## Configuring the handover target

A handoff to another Agent is configured by **@-mentioning that Agent at the handoff step** — not by describing `request_handover` in prose. The @-mention is what registers the target Agent as an allowed handover target and makes the `request_handover` tool available at run time. Without a matching mention, `request_handover` is never injected and the branch cannot hand over.

So an AOP branch that hands off must:

1. **@-mention the target Agent** in the step where the handoff happens — e.g. "If the request is a billing question, hand over to **@Billing Specialist**." This skill has no access to the user's Agents, so write the mention as a readable placeholder — `@Target Agent` — for the caller to resolve to the real Agent.
2. **Call `request_handover`** to that Agent as the branch's terminal action, and nothing else (see "Handover vs. terminality" above).

The set of `@Agent` mentions in the AOP is the source of truth for the handover targets: a `request_handover` branch with no mention is a defect.

## When to use which terminal

| Situation                                                                                           | Terminal call      |
| --------------------------------------------------------------------------------------------------- | ------------------ |
| The AOP's work is finished and the case is resolved.                                                | `complete_case`    |
| The case can't be resolved now but can be retried later (SLA, rate limit, awaiting external reply). | `postpone_case`    |
| The case can't be resolved at all (missing data, broken Connection, unsupported case type).         | `fail_case`        |
| A different Agent is better placed to continue (different domain, different specialist).            | `request_handover` |

## Case-lifecycle anti-patterns

- **Forgetting `claim_case` as step 1.** Without it, the AOP has no case data to read.
- **Using `postpone_case` for permanent issues.** Permanent issues bury the queue under cases that will never resolve — use `fail_case`.
- **Calling `complete_case` and `request_handover` in the same branch.** Pick one.
- **Updating the case only at the end.** Update as you go — `update_case` writes survive a postpone or handover, so they are how the next Agent gets context.
- **Looping inside the AOP** ("for each pending invoice in the queue"). One Run, one case. The platform iterates.

## See also

- `references/aop-structure.md` — where these tool calls appear in the GOAL / STEPS / NOTES shape.
- `references/aop-examples.md` — worked examples that use the postpone-then-retry idiom and the handover pattern.
