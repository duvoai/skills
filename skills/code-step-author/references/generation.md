# Review generated changes

Generation writes a draft for review. Start it only for a confirmed change.
A request for advice alone does not authorize a save. Use this sequence for
both Agent and Code Steps.

1. Call `generateRevision` for the source build with the confirmed prompt.
   On a versioned team, pass `selected_automation_revision_id` when the
   selected revision is known. Omission targets the active revision; do not
   invent an ID. Retain `generation.builder_run_id` and `generation.build_id`
   from the 202 response. The returned build may differ from the source build.
   Use the returned IDs for all later reads and decisions. A 202 means pending,
   not that the change succeeded.
2. Poll `getBuilderRun` using that run ID with bounded waits between requests.
   `not_started` and `running` mean pending. On `waiting_for_input`, present
   the actual `pending_user_questions` through the host's question interface.
   Preserve the wording and choices. Use `resumeBuilderRun` only with the
   user's answers keyed by the exact original `question` text, then continue
   polling. Do not invent answers or start a second generation. If the host
   cannot collect an answer, leave the work pending.
3. On `waiting_for_approve`, read the returned build with `getRevision` and
   compare its saved config with the run's `checkpoint_config`. For an Agent,
   compare the AOP; for a Code Step, compare the program, not AOP fields.
   Inspect warnings and the actual resource, Connection and Policy bindings.
   Present the changed passages and explain any difference from the confirmed
   proposal. Wait for the user's decision to keep or discard these changes.
4. To keep changes, call `acceptRevisionGeneration`; to discard them, call
   `declineRevisionGeneration`. Supply the reviewed build ID, builder run ID
   and `expected_started_at` copied exactly from `builder_run.started_at`,
   including `null` when returned. Each new prompt has a new run ID even when
   it reuses a sandbox. A stale decision is a conflict: refresh through the
   available host interface and review again. If the current run cannot be
   identified, stop and explain the blocker. Do not retry with guessed IDs or
   timestamps.
5. Read the build again after either decision. Acceptance keeps the saved
   draft. Decline restores the checkpoint config; inspect the resulting
   bindings rather than assuming all resource changes were undone. Both
   decisions can report `completed`, so that status alone does not prove
   changes were accepted or preserved. Neither decision promotes or activates
   anything.

For `failed` or `cancelled`, explain the actual state and what stopped.
`cancelBuilderRun` only cancels `not_started`, `running` or `waiting_for_input`;
discard a draft awaiting approval through `declineRevisionGeneration`.
Cancellation does not promise rollback of previously saved content.

If a request fails or times out, inspect the known run and build before
retrying. An access refusal or 404 is not proof that generation never started.
If polling cannot finish this turn, report the work as pending and retain the
IDs for continuation. Never report pending, failed, declined or unverified
changes as applied. A saved draft still requires separate activation.
