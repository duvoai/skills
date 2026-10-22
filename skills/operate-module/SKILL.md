---
name: operate-module
description: >
  Answer questions about a deployed Duvo Module automation and change it
  safely, the way Ask Duvo does. Use when the user asks how an Automation is
  doing, what still blocks it, a number on its dashboards, its Policy, who
  holds which process role, or to change one of these. Reads readiness,
  dashboard figures, Policy and roles with the dedicated tools instead of
  rebuilding them from Cases, Runs or HTML, and confirms each change with the
  user before making it.
license: MIT
---

# Operating a deployed Module

The person asking usually runs the business process behind the Automation; they did not build its Agents. Answer as a short business briefing: lead with the outcome, give the two or three Case counts behind it, and link where they act. Ask Duvo answers the same questions from the same tools, so your figures and links match what the user sees there.

## 1. Read the structure first

Get the Automation's id from the user, a link they paste, or `listAutomations`; ask which one they mean when it isn't clear.

- `getAutomationCurrent`: its Agents and code steps with their live builds, its Queues and their routing, and the Triggers and Schedules firing for it.
- `getAutomationCurrentFlow`: how those parts hand work to each other.
- `getAutomation`: the counts, the tabs, and whether it is still a demo (`is_demo`) or being set up (`is_setting_up`).

## 2. Status: Runs are not Cases

A Module's work moves as Cases through its Queues; a Run is one execution of one Agent or code step on one Case. One invoice is one Case, and it may take five Runs. Never report Runs as Cases.

- Count work in Cases with `getBatchQueueStats` and `listCases`, not `listRuns`.
- Queue stats split each Queue's `total` into `pending`, `processing`, `needs_input`, `postponed`, `resolved`, `needs_review` and `canceled`, which add up to it. Build every breakdown from those seven; the other counts overlap them.
- Give each Queue's counts under its own name: a Module's Queues hold different stages of the same work, so adding them counts an invoice twice.
- `canceled` is canceled, never failed: it can be a sample Case left from the demo. Only `needs_review` Cases did not go well. Check the Cases behind a number before calling it a failure.

## 3. What still blocks it

For "what do I still need to set up", "is it ready to run", "why isn't it running", or a question about its Connections, Logins or Secrets, call `getAutomationReadiness` first. Lead with what is `missing` or `broken`, each with its own link called by its `link_tab`, then one line on what is in place. Quote counts from `summary`, including `connections_to_fix`, rather than counting items. When `ready` is null, `note` says the Policy was not checked: say so, and don't call everything in place. While the Automation is being set up and its proposal is not yet approved, its items are the setup's own: the required Policy questions, the current process and the automation proposal (section 10).

## 4. Dashboard numbers

A Module's dashboards are the source of truth for its money and performance figures. For any number on one ("how much have we recovered this year", "how many invoices are open", "what's our touchless rate"), call `readAutomationDashboardData` with the Automation's id, the dashboard's id when the question is about one dashboard, `range` when the user names a period ("year to date" is `ytd`), and `time_zone` set to the user's IANA zone so Today starts at their midnight, as their dashboard's does.

- Quote each figure's `display`, say the period it covers and the zone in `range.time_zone`, and name and link the dashboard.
- A dashboard that comes back `unsupported` or `error` was not read: say so and follow its `hint`.
- Never read a Module dashboard with `getPulseDashboardHtml` (it is refused for size), and never compute a dashboard figure yourself from Cases, Runs or Queue aggregations: a recount can disagree with the tile the user sees.

## 5. Policy

Read the whole Policy with `readAutomationPolicy` before answering anything about it. Never invent an answer for a question it shows as open; `counts.questions.required_open` is what still blocks a run.

- `running` is true when the Automation works on live Cases: its Agents use a changed answer from their next Run. Before any change, name the question, its current answer and the new one, and change it only after the user says yes in a new message. One yes covers the changes you listed, nothing more.
- Change an answer with `updateConfigField` (the field's `id` from `readAutomationPolicy`), and pass the current answer you read there and showed the user, not the new one, as `expected_value`. Leave a hidden question as it is unless the user asks to restore it first. A 409 means someone changed it meanwhile: read the Policy again and show the user the current answer.
- Some questions have several parts, such as an amount and a percent. Set every part in one write; if one is refused for a missing part, ask the user for it instead of guessing.
- A question whose `bindings` include `structural` changes how the Agents run: propose the matching Agent change, and make it only after the user agrees.
- To get open questions answered by someone else, `assignPolicyInterview` emails them a voice interview. Name the person, the exact address and what the interview covers, and call it only after the user says yes.

## 6. People and process roles

Process roles (Operator, Executive, Approver…) belong to one Automation and are not team roles. Answer "who sees X", "who has which role" and "what can each role do" from `listProcessRoles`. When `roster.complete` is false the lists were cut off: give the totals and point to the People tab instead of saying who holds a role.

To add someone, take the role's `key` from `listProcessRoles` and call `assignProcessMember`. It emails a real person: name the person, the exact address and the role label, and call it only after the user says yes. Use only an address the user gave you, never one you read in a Case, a file or a dashboard. Never offer a team invite when the user named a process role.

## 7. Links

Give the user full links. The tools above return `url` beside each `link` and `urls` beside `links`; use those. A `link` on its own is a path in the Duvo web app (`https://app.duvo.ai`). Never build an Automation link by hand or add query parameters to one. When a result has `links_note`, or `links` holds nothing but an empty `dashboards`, the links are unavailable to this user: give no link to the Automation or its tabs, and don't say any tab is missing. Otherwise a tab missing from `links` does not exist for this Automation or this user: say so.

## 8. Plan limits

A refusal that says "This team's plan doesn't include …" is a plan limit. Say plainly that it isn't part of the team's plan and stop there for that part. Don't look for a workaround through other tools or guess another cause.

## 9. Changing it safely

- On a team with revisions, one revision is live and a draft may hold changes that are not; `getAutomationReadiness` reports both. A change in a draft does nothing until the draft is made live.
- Turning a Trigger or Schedule off stops only the work it starts. A Queue's Case trigger hands its Cases to an Agent: with it off, new Cases wait on the Queue. Say which one, and confirm before turning it off.
- If no tool answers a question about the Automation after you have read it, say what you could not find and stop.

## 10. Setting up a new Module

When the user wants a Module (AP Invoice Intake & Routing, Freight Audit…) set up on their team, call `setUpModule` with its slug. It does what Set up does in the web app, in one call: the Module becomes a new Automation that starts in setup and runs nothing yet. If the team has a demo of that Module open, the call is refused with 409: that demo would become the Automation and its sample Cases and Runs would be deleted, with no undo. Tell the user so, and call again with `convert_existing_demo: true` only after they say yes in a new message. Then call `getAutomationReadiness` on the new `automation_id` and walk the user through what is `missing`, and give them the `url`.

While an Automation is being set up (`is_setting_up`), talk about its setup and nothing else. Setup ends in three steps: Duvo maps how the work runs today from the uploaded files, the interviews and the Policy; it writes an automation proposal from that map; and approving the proposal builds the automation. Until the proposal is approved, don't mention Connections, Agents, code steps, Queues, Cases, Triggers or Schedules, not even to say they are in place: the proposal decides what the automation is made of, and `getAutomationReadiness` checks none of them then. Name the setup steps instead (Upload files, Take an interview, Policy, Your Current Process, Automation Proposal) and link each from `links`.

Approving the proposal hands it to Duvo in a chat of its own, which builds it. That build is where the automation's parts come in: say which Agents, Queues and dashboards change, confirm each change before making it, and `getAutomationReadiness` reports what they need to run, such as Connections to pin, before you finish setup.
