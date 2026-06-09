# Case Lifecycle in SOPs

Read this reference when the Assignment is queue-driven — it has a `case-queue-producer` Connection, a `case-queue-consumer` Connection, or both. Standalone Assignments (no queue role) ignore this file.

The case lifecycle tools are platform primitives whose **names matter to the runtime**. Reference them by name in the step where they apply — they are the one exception to the "describe what, not how" rule.

## Producer tools

Used by Assignments with the `case-queue-producer` Connection.

- **`add_cases`** — enqueue one or more cases (1–100 per call). Each case has a `title` (required), and optional `data` and `labels`. Always batch when producing multiple cases in one Run; the platform expects up to 100 per call.
- **`list_cases`** — query an existing queue with filters (`title_search`, `status`, etc.). Use as a dedup helper before `add_cases` when the producer runs frequently enough to risk duplicates.

A producer SOP's terminal step is the `add_cases` call (or a no-op if there is nothing to enqueue this Run). Producers do not call `claim_case` / `complete_case`.

## Consumer tools

Used by Assignments with the `case-queue-consumer` Connection. A consumer SOP processes one claimed case from start to finish.

- **`claim_case`** — fetch the case for this Run (and atomically claim it, if not already bound at Run start). **MUST be step 1** of a consumer SOP on the initial run, because case data is not injected into the prompt automatically. On a follow-up message in the same Run, skip this step — the case is already bound. A Run can only claim one case in its lifetime.
- **`update_case`** — write `title`, `data`, or `labels` back to the in-flight case. Add an `update_case` step only where the process needs to record progress, intermediate findings, or state that must survive a `postpone_case`.
- **`postpone_case`** — release the case and schedule a re-claim. The `postpone_to` parameter accepts a relative duration (`"2h"`, `"1d"`, `"1w"`) or an ISO timestamp. Use only for cases that _can_ complete later (waiting on a response, an SLA, a rate limit). Do NOT postpone for permanent or environment issues — use `fail_case` instead.
- **`complete_case`** — terminal success. Mark the case done after the SOP's work is finished.
- **`fail_case`** — terminal failure for permanent issues (missing data, broken credentials, unavailable tools).

## The postpone-then-retry idiom

This is the canonical pattern for any SOP step that says "wait N days then act" — SLAs, follow-up reminders, scheduled second touches.

Add a flag on the case the first time the SOP runs. On the next pickup, the flag is present and the SOP does the real work.

```markdown
2. If `initial_postpone_done` is NOT present in the case data (this is the first pickup):
   - Call `update_case` to set `initial_postpone_done: true` in the case data.
   - Call `postpone_case` with `postpone_to: "1d"` (or whatever the wait is).
   - Exit — do not continue to the steps below.

3. (Second pickup — `initial_postpone_done` is true.) Search the Slack thread for a PO reference. …
```

Use a different flag name (`reminder_sent`, `escalation_sent`) for each waiting phase if the SOP has more than one wait.

## Terminal closure rule

Every branch of a consumer SOP must end in **exactly one** of:

- `complete_case` — success.
- `fail_case` — permanent failure.
- `postpone_case` — temporary wait; the case returns to the queue.
- `request_handover` — pass the claimed case to a different Assignment.

If a branch does not end in one of these calls, Duvo auto-marks the case `Failed` with no context. This is the most common failure mode for queue-driven SOPs — every branch must reach a terminal call.

## Handover vs. terminality

Handover and case terminality are **mutually exclusive**.

- If the SOP plans to call `request_handover` in a branch, do **not** also call `complete_case`, `fail_case`, or `postpone_case` in the same branch. The platform passes the claimed case to the target Assignment automatically.
- Conversely, every consumer SOP that does **not** request a handover must end every branch with `complete_case`, `fail_case`, or `postpone_case`.

The `handover` Connection itself is added automatically at runtime — do not list it in the Setup, and do not need to mention it in the SOP except via the `request_handover` tool name.

## When to use which terminal

| Situation                                                                                           | Terminal call      |
| --------------------------------------------------------------------------------------------------- | ------------------ |
| The SOP's work is finished and the case is resolved.                                                | `complete_case`    |
| The case can't be resolved now but can be retried later (SLA, rate limit, awaiting external reply). | `postpone_case`    |
| The case can't be resolved at all (missing data, broken Connection, unsupported case type).         | `fail_case`        |
| A different Assignment is better placed to continue (different domain, different specialist).       | `request_handover` |

## Case-lifecycle anti-patterns

- **Forgetting `claim_case` as step 1.** Without it, the SOP has no case data to read.
- **Using `postpone_case` for permanent issues.** Permanent issues bury the queue under cases that will never resolve — use `fail_case`.
- **Calling `complete_case` and `request_handover` in the same branch.** Pick one.
- **Updating the case only at the end.** Update as you go — `update_case` writes survive a postpone or handover, so they are how the next Assignment gets context.
- **Looping inside the SOP** ("for each pending invoice in the queue"). One Run, one case. The platform iterates.

## See also

- `references/sop-structure.md` — where these tool calls appear in the GOAL / STEPS / NOTES shape.
- `references/sop-examples.md` — worked examples that use the postpone-then-retry idiom and the handover pattern.
