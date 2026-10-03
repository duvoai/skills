---
name: clarity-mapping
description: >
  Map how a process works with Duvo Clarity. Use when the user asks to map,
  document or understand how their team does a piece of work, to interview
  people about a process, or to analyze what those interviews found. Sets up a
  Duvo workspace if the user has none, creates the process, invites people to
  short voice interviews within the plan's allowance, tracks who has done
  theirs, and analyzes the process map.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Clarity Mapping

## What is Clarity?

[Duvo](https://duvo.ai) Clarity learns how work gets done by interviewing the people who do it. Each person has a short voice interview in the browser, and Duvo turns what they say into a map of the process: its steps, roles, systems, handoffs and problems. Everyone you invite to a process gets an email invitation to their interview.

## What you're doing

The user wants to understand how a piece of work gets done. You take them from nothing to an analysis of one process:

1. Get their Duvo workspace, creating a Free one if they have none.
2. Create the process.
3. Invite the right people to interviews, after the user confirms the list.
4. Report who has done their interview.
5. Analyze the process map the interviews produce.

Map one process at a time. For a second process, run the steps again from step 2.

## Operating mode

Use this session's configured Duvo access to perform the operations below. Follow its runtime instructions for invocation and parameter lookup. Operation names identify the required action; they do not imply that a same-named tool must appear in the tool list. Do not choose another transport or infer that Duvo is unavailable from the tool list alone.

### Connector or CLI

Both reach the same operations and return the same fields.

- **Through the Duvo connector** (claude.ai, ChatGPT and other MCP clients), each operation below is a tool with that name.
- **With the `duvo` CLI** (for example in Claude Code), run the command in brackets after each operation and add `--json` to read the fields this skill names. Pass `--team <team_id>` to team-scoped commands, or run `duvo team use <team_id>` once. If the user isn't signed in, ask them to run `duvo login` first.

## Rules

- **Never email anyone without the user's explicit yes.** Inviting a person sends them an email. Before you do, show the user the process and each person's name and email, and say that each will get an email from Duvo. Wait for a clear yes.
- **Use only the names and email addresses the user gave you**, from their org chart, a file, a directory connector or the chat. Never guess or construct an email address.
- **Stay within the interview allowance.** Invite no more people in a run than `interviews_remaining` (step 4).
- **Stop on a permission refusal.** If a tool refuses because of the user's role, tell the user they need a team manager or admin to set this up, and stop. Don't retry.
- **Pass links on exactly as you received them.** That includes `billing_url` and any link in a refusal.
- **Don't use organization-level interviews or the Operating Model.** They are not part of the Free and Pro plans. Analyze from the process summaries.

## Steps

### 1. Get the workspace

Call `createWorkspace` (`duvo teams create-workspace`). It never creates a second workspace: if the user already has one, it returns it with `created: false`.

- `created: true` means you just set up a Free workspace. Tell the user it includes Clarity and Automation, and that Free has a small number of interviews.
- Keep `team_id` for the calls below.
- If `has_clarity_access` is `false`, stop. If `interviews_remaining` is also `0`, the workspace has no plan yet: give the user `billing_url` to choose one. Otherwise tell them Clarity isn't on their plan and give them `billing_url`.
- `interviews_remaining` is a number or `"unlimited"`. You need it in step 3.

If the call is refused, pass the message on to the user, including any link, and stop. A refusal with a link usually means their company or email domain is already on Duvo, and they need to join that workspace first.

### 2. Create the process

Ask which process to map, in the user's own words: for example, "how we approve supplier invoices". Then call `createClarityProcess` (`duvo clarity create --name <name>`) for the team with that `name`. Keep the returned `id` as the process id.

### 3. Choose who to interview

Ask who does the work, or take it from their org chart: a spreadsheet, a pasted list, or one of the user's connectors (an HR system or a directory). Pick the people who do or hand off the work, one or two per role, and note each person's name, email and role in the process.

Check the allowance. `interviews_remaining` from step 1 is how many interviews the plan has left, or `"unlimited"`. Tell the user the number, and that interviews already under way elsewhere in the team use it too. By default, invite no more people to this process than that. Duvo enforces the limit when an interview starts: if an invitee is refused, the plan is full, so tell the user and give them `billing_url`.

If more people should be interviewed than the plan allows, say how many and who you would add. Step 4 covers what to tell them when the allowance runs out.

### 4. Confirm, then invite

Check the allowance before you show the list: call `createWorkspace` (`duvo teams create-workspace`) and read `interviews_remaining`. This run invites at most that many people. The number only drops when an interview is recorded, so it doesn't change as you send invitations, and invitations still pending from earlier runs aren't counted in it.

- If it is `0`, invite no one. Tell the user the plan has no interviews left and give them `billing_url`, where they can upgrade to a plan with more interviews.
- If the list is longer than that, ask the user which people to invite now, up to that many. Tell them who would be left out and give them `billing_url`.
- On Free, a workspace can also invite at most 25 email addresses in total.

Show the user the process and the list: name, email and role for each person. Say that each will get an email invitation to a short voice interview.

Then stop and wait for the user to say yes to that list. A request to map the process is not a yes, and neither is approval given before they saw the list. If they change the list, show it again and wait again. Don't create or send any invitation until they have said yes.

Every time the user says yes to a list, the first one or a shorter one, call `createWorkspace` again and read `interviews_remaining` right before sending, since it can drop while they decide. If it is now lower than the list, send nothing yet: tell them the new number, give them `billing_url`, and ask them to pick that many people from the list. Send only when the list they confirmed fits the latest number.

Then invite each person on the list:

1. Call `createTeamInvite` for the team with `email`, `role: "team:clarity-member"` and `processId` set to the process id. Keep the returned invitation `id`.
2. Call `sendTeamInviteEmail` with that `id` to email the invitation. Creating an invitation sends nothing by itself.

With the CLI, one command does both: `duvo invite create --email <email> --role team:clarity-member --process <process-id> --send-email`.

If an invitation is refused with code `INVITE_LIMIT_REACHED`, stop inviting and show the user the message, which includes the upgrade link. If the email fails to send, tell the user; you can retry it once with `sendTeamInviteEmail` (`duvo invite resend <id>`).

Tell the user who was emailed, that the interview is a short voice conversation in the browser, and that the interviewer will say who asked for it.

### 5. Report progress

Read two things for the process:

1. `listClarityProcessMembers` with the process id (`duvo clarity members list <process-id>`). `pending` lists invitations not accepted yet. `members` lists people who accepted, whether or not they have recorded. `contributors` lists only people who recorded without an accepted invitation, so an invited person never appears there.
2. `getClarityProcess` with the process id and `captures: "lite"` (`duvo clarity get <process-id>`). Each entry in `captures` is one interview, with the `userId` of the person who recorded it and its `status`.

For each person you invited:

- A capture with their `userId` and `status: "complete"`: they have done their interview.
- A capture with their `userId` and another status (`recording`, `processing`): their interview is under way.
- No capture with their `userId`, and they're in `members`: they accepted but haven't recorded yet.
- In `pending`: they haven't accepted the invitation yet.

Tell the user who has and hasn't done their interview. Don't chase people yourself; the user decides whether to remind them.

### 6. Analyze

Once at least one interview is done:

1. Make sure the process has a map. If it isn't in `listClarityProcessSummaries` for the team yet (`duvo clarity process-summaries`; page through with `limit` and `offset` until you have seen `total` processes), call `generateClarityProcessSnapshot` with `process_id` set to the process id and `kind: "current_process"` (`duvo clarity generate-current-process <process-id>`). Generation runs in the background, so check again later.
2. Call `listClarityProcessSummaries` for the team with `detail: "skeleton"` and `include_current_steps: "true"`. The process carries its summary and SWOT, and the roles and systems involved. Steps and projected impact appear only once the process also has a transformation proposal.
3. For the step-by-step current process, call `listClarityProcessSnapshots` with `process_id` and `kind: "current_process"` (`duvo clarity list-snapshots <process-id> current_process`), take the snapshot with the highest `version_number`, and read it with `getClarityProcessSnapshot` (`duvo clarity get-current-process <process-id> <snapshot-id>`).
4. Analyze what you read: how the work flows, where it waits, where people or systems hand work over, and what the interviewees said is hard. Base every claim on the process data. Say where the picture is thin because an interview is missing.

Only say that a person said or confirmed something if it is in their interview's summary or transcript. Quote it, or name the interview it came from. Label anything you inferred from the process data as your inference, not as something someone said.
