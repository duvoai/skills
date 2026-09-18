---
name: code-step-author
description: >
  Answer questions about writing the Python program of a Duvo code step and
  the `duvo` SDK it calls. Use when the user asks how a code step reads its
  case, claims or settles work on a Queue, reaches a Connection, reads Policy
  values or attached files, what `duvo.execution` holds, which errors the SDK
  raises, or wants a snippet for a code step. Grounds every answer in the SDK
  surface that actually ships into the sandbox, and hands the writing of the
  program itself to the step's own AI generation rather than pasting code the
  user must transcribe.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Code Step Author

## What is Duvo?

[Duvo](https://duvo.ai) is an AI-powered automation platform that handles repetitive business work across the systems a team already uses. An automation is built from two kinds of node, told apart by the guarantee each makes rather than the work it does:

- An **Agent** is judgement — an **AOP** (the markdown procedure that becomes its prompt), evals, memory, skills, and all human interaction.
- A **Code Step** is deterministic compute — a Python program the platform invokes. It reads and writes one case's data, runs exactly once per dispatch, and fails the case if it fails. No model runs when it executes.

A Code Step's versions are **Builds** and its executions are **Runs**, exactly as an Agent's are. Where an Agent's Build holds an AOP, a Code Step's Build holds source and an entrypoint.

## What you're doing

The user is writing, reading, or debugging the program inside a Code Step, and needs to know what the `duvo` client can actually do. That surface has **no autocomplete and no generated type declarations** — it is only knowable from prose, and there is nothing in the public product documentation that covers it. That is what this skill is for.

Read `references/sdk.md` before answering anything specific. It is the same text the step's own AI generation is given, generated from one source, so an answer grounded in it and a program Duvo writes cannot disagree.

**The one rule that matters: never invent SDK surface.** A plausible-looking method that does not exist produces a step that fails on its first dispatch, and the author has no way to tell that from a real bug. If the user needs something `references/sdk.md` does not name, say it is not in the SDK rather than guessing at a neighbouring method.

## Answering

Answer from the reference, in the user's terms:

- **Quote the real call**, with its keyword arguments. `duvo.claim_case(queue_id=...)`, not "claim the case".
- **Keep snippets to the few lines that answer the question.** The user has their program on screen; a whole rewritten file buries the one line they asked about.
- **Name the boundary when it explains the answer.** No Connection credential is in the sandbox; only the Connections attached to the step resolve; only the Python standard library and `duvo` are installed. Most "why can't I just…" questions are one of these three.
- **Settle every case the program claims.** A claimed case left unsettled blocks the queue, and it is the single most common correctness bug in an authored step.

## Which Connections and tools this revision actually has

`references/sdk.md` says what the `duvo` client offers. It cannot say which Connections are attached to the revision in front of you, or what tools each one reports — that is per-build, and two builds of the same step can differ.

**`listCodeStepTools`** answers it, for one Build: every reachable Connection, the attribute the author writes after `duvo.connections.`, and each Connection's tool names with one-line descriptions. Call it before saying anything specific about what a step can call.

- **Copy the `attribute` exactly.** It is not derivable from the Connection's type: a build with one Gmail answers to `duvo.connections.gmail`, a build with two answers only to `gmail_<instance>` and drops the short name entirely, so a name you construct yourself is an `AttributeError`.
- **At `confidence: "current"`, a Connection absent from the list is not attached.** Say so and offer to attach it; do not write a call the step cannot make. At `approximate` the list cannot settle it — see below.
- **`status` is not `ok`** — the Connection is reachable at run time, but its catalog could not be listed. Its tool names are _unverified_, not absent: answer, and say the name could not be checked.
- **`confidence: "approximate"`** means a refresh failed and this is the last catalog that succeeded. Both tools report it. Still answer from it — but any name you cannot find is _unverified_, not missing, Connections included: this catalog predates whatever was attached since it was resolved, so saying a tool does not exist or a Connection is not attached is the one answer it has no standing to give.

**`describeCodeStepTools`** gives up to eight of one Connection's tools in full: arguments, which are required, and the exact call. Use it for the tools you are about to quote.

**Quote `call_form` verbatim; never assemble the call yourself.** Whether `duvo.connections.<name>.<tool>(arg=...)` reaches the right thing depends on the Connection's name, the tool's name and every argument name at once, and most of the ways it does not are silent:

- an argument named `from` or `class` is a `SyntaxError` — Python cannot take it as a keyword;
- a name with punctuation _parses_: `duvo.connections.gmail.send-message(to=1)` is `duvo.connections.gmail.send - message(to=1)`, a subtraction;
- a tool named `call` or `server` reaches the SDK's own method on the Connection instead;
- a Connection whose name is a Python keyword needs `duvo.connections["class"]`.

`call_form` accounts for all four, and it is the same computation the step's own generation uses — so quoting it keeps your answer and any program Duvo writes in agreement.

`arguments_known: false` means the server described its arguments in a form Duvo cannot read (`$ref`, `allOf`, `oneOf`). That is _unknown_, never _none_ — do not tell the user the tool takes no arguments.

Both tools are read-only and describe one Build. They are not available where code steps are off, in which case answer from `references/sdk.md` alone and say tool names are unverified.

## Writing the program is a different action

Answering a question is free. **Changing the step's program is a save**, and it goes through the Code Step's own AI generation — not through you pasting source the user has to transcribe.

When the user wants the program written or changed:

1. Confirm what the change should do. A request for advice is not authorization to save.
2. Run the generation lifecycle in `.claude/skills/meta-agent/references/automation-editing.md` — `generateRevision` on the step's Build with the confirmed prompt, poll `getBuilderRun`, present the diff, and accept or decline on the user's word. It writes a draft for review; it does not promote or activate anything.
3. The generation is given this same SDK reference **and this revision's tool catalog**, so restate neither in the prompt. Pass what the author wants the step to _do_.

Two things it cannot edit, and the user should be told before they type a prompt rather than after: a step whose source is its own **files** rather than a program on the Build, and a Build whose config cannot be read.

## Boundaries

| The user wants                                      | Skill                                                        |
| --------------------------------------------------- | ------------------------------------------------------------ |
| What the `duvo` SDK can do, or a snippet against it | This skill                                                   |
| Which Connections and tools one revision has        | This skill — `listCodeStepTools`, `describeCodeStepTools`    |
| An Agent's AOP written, rewritten, or critiqued     | `aop-writer` — none of the SDK applies to an AOP             |
| Why one Code Step Run failed                        | `run-debugger`, then come back here if the fix is in the SDK |
| Where a Code Step belongs in a new workflow         | `workflow-architect`                                         |

A Code Step never conducts human interaction — it flags, and an Agent node conducts the escalation. If the user is asking how to ask a person something from inside a Code Step, that is the answer: it belongs in the Agent after it.
