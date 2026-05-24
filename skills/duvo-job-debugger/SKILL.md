---
name: duvo-job-debugger
description: >
  Investigate why a Duvo Job failed or produced the wrong outcome. Use when
  the user shares a failed Job, asks "why did this Job fail", or wants to fix
  a recurring failure on an Assignment. Reads the Job transcript via the Duvo
  API or user-provided data, names the root cause from a fixed failure-mode
  taxonomy, and proposes one concrete fix.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Duvo Job Debugger

## What is Duvo?

[Duvo](https://duvo.ai) is an AI platform that designs, runs, and monitors
autonomous agents (called **Assignments**) for business process automation.
A **Job** is one execution of an Assignment. When a Job fails or produces the
wrong outcome, the user wants two things:

1. **What went wrong on this specific Job.**
2. **What change would prevent it next time.**

This skill answers both — grounding the diagnosis in the actual transcript and
turning it into a concrete proposal.

## Operating modes

This skill works in two modes:

- **API mode** — the Duvo public API is available as MCP tools or via the
  `duvo` CLI. Use `getRun`, `listRunMessages`, `getRevision` to pull data
  directly.
- **Paste mode** — no API access. Ask the user to share the Job ID, the SOP
  that was in effect, and relevant transcript excerpts.

## The single most important rule

**Distinguish the symptom from the root cause.** A tool-call error or missing
field is the symptom. The root cause is almost always upstream:

- The **SOP** didn't specify what to do on that branch
- A **Connection** wasn't available or was missing scopes
- A **Setup input** or **File** the SOP referenced was missing
- A **terminal action** was missing from a branch
- The Assignment was carrying two Jobs of work in one SOP

## Investigation workflow

1. **Anchor on the Job.** Get the Job's `build_id`, status, and any top-level
   error, plus the SOP that was in effect. Always use the revision the Job ran
   against, not the Assignment's current revision.

2. **Read the transcript.** Walk forward and locate the **decision point**
   where the Job took the path that led to the failure.

3. **Map the failure to a category** (see taxonomy below).

4. **Pull supporting evidence as needed.** Connection state, pattern across
   recent Jobs, case terminal closure.

5. **Propose one fix.** Name the artifact that has to change and the specific
   change.

## Failure-mode taxonomy

1. **Connection failure.** OAuth expired, scope missing, upstream 4xx/5xx.
   Fix: refresh/extend Connection scopes, or add a fallback branch.

2. **SOP ambiguity.** The SOP used "decide" or "use judgment" where a concrete
   threshold was needed. Fix: inline an `if [threshold]: [action]` rule.

3. **Missing terminal closure.** A queue-driven Job ended without
   `complete_case` / `fail_case` / `postpone_case` / `request_handover`.
   Fix: add the terminal action to that branch.

4. **Missing HITL on a high-stakes action.** The Assignment took a costly
   action autonomously. Fix: gate that branch on a Human-in-the-loop
   checkpoint.

5. **Missing data.** The SOP referenced a Setup input, File, or case field
   that wasn't present. Fix: add the missing input or handle the empty case.

6. **Batch / iteration leak.** The SOP said "process all pending records"
   instead of one case. Fix: rewrite as single-case SOP.

7. **Wrong decomposition.** One Assignment doing the work of two. Fix: split
   into two Assignments connected by `request_handover` or a case queue.

## What a fix looks like

A fix is **one concrete change to one artifact**:

- "Add to Step 4: _If amount > $5,000, use **Human in the loop** to confirm._"
- "Extend the **Gmail** Connection scopes to include `gmail.send`."
- "Add a `supplier_tier_a_threshold` Setup input; reference it in Step 2."
- "Split the SOP at Step 6 into a second Assignment via `request_handover`."

Avoid vague suggestions like "review your SOP" or "tighten the logic".

## Output format

Return one structured response with these sections:

- **Failure mode** — one of the taxonomy categories, or a named pattern
- **Where it went wrong** — the specific transcript point and matching SOP step
- **Evidence** — one or two quoted lines from the transcript or SOP
- **Fix** — one concrete change to one artifact
- **Next step** — e.g. "Rewrite Step 4 of the SOP" or "Refresh the Gmail
  Connection"

## Anti-patterns

- Diagnosing without reading the transcript
- Diagnosing against the current SOP when the Job ran against an earlier Build
- Stopping at the surface error (a 401 is not the diagnosis)
- Bundling multiple unrelated fixes
- Inventing Connection names or tool calls not in the transcript

## Duvo terminology

| Term           | Not                          |
| -------------- | ---------------------------- |
| Assignment     | agent, AI teammate           |
| Job            | task, run                    |
| SOP            | instructions                 |
| Connection     | integration                  |
| Files          | knowledge base               |
| Start Work     | run agent                    |
| Setup          | configuration                |

## Resources

- [Duvo documentation](https://docs.duvo.ai)
- [Duvo CLI](https://www.npmjs.com/package/@duvoai/cli)
- [Web app](https://app.duvo.ai)
