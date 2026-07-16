# Command Reference

Full command tree for `duvo` (`@duvoai/cli`) as of the most recent
published version. Cross-reference against `duvo <command> --help` from
the installed binary if a flag here doesn't match — the binary is
authoritative.

Global flags available everywhere:

- `--profile <name>` — override the default profile for this invocation.
- `--team <id>` — override the active team for this invocation (also settable via `DUVO_TEAM_ID`; API-key profiles reject this because the key is already scoped to a team).
- `--version` — print the installed CLI version.
- `--help` — print help for any command.

Every command that talks to the API additionally supports `--json` to
emit the raw API response (the stable scripting contract).

Confirmation flag for destructive commands: `-y` / `--yes` skips the
TTY prompt. In a non-TTY context the prompt is replaced by a refusal —
pass `-y` explicitly.

## Authentication & profiles

| Command                                                              | Purpose                                                                                                                                  |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo login [--name <profile>] [--api-key <key>] [--base-url <url>]` | OAuth sign-in (opens browser). Pass `--api-key` to store a long-lived key without OAuth. `--base-url` defaults to `https://api.duvo.ai`. |
| `duvo logout [--name <profile>] [-y]`                                | Remove a profile (default: the current default profile).                                                                                 |
| `duvo whoami`                                                        | Show the active profile, signed-in user, and auth type (`OAuth` / `API key`).                                                            |
| `duvo profiles list [--json]`                                        | Table of stored profiles; `▶` marks the default.                                                                                         |
| `duvo profiles use [name]`                                           | Switch the default profile. With no argument: interactive picker (TTY only).                                                             |
| `duvo profiles rename <old> <new>`                                   | Rename a profile, preserving its credentials.                                                                                            |
| `duvo profiles remove <name> [-y]`                                   | Remove a profile (alias of `duvo logout --name`).                                                                                        |

## Agents

| Command                                                                                                                                                                                                                                         | Purpose                                                                                                                                  |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo agents list [--limit <n>] [--offset <n>] [--json]`                                                                                                                                                                                        | List agents on your team. `--limit` is 1-100 (default 20).                                                                               |
| `duvo agents get <id> [--json]`                                                                                                                                                                                                                 | Show details for one agent.                                                                                                              |
| `duvo agents create [--name <n>] [--input <text>] [--config <path>] [--model <model>] [--no-build] [--json]`                                                                                                                                    | Create a new agent. Prompts for missing fields in a TTY. `--model` picks the Claude model; `--no-build` skips creating an initial build. |
| `duvo agents update <id> [--name <n>] [--enable-slack\|--disable-slack] [--enable-microsoft-teams\|--disable-microsoft-teams] [--enable-agentic-memory\|--disable-agentic-memory] [--thread-id <id>\|--clear-thread] [--pin\|--unpin] [--json]` | Update display name, delivery settings, thread association, or your per-user pin (`--pin`/`--unpin`).                                    |
| `duvo agents models [--json]`                                                                                                                                                                                                                   | List the LLM models available for agents. The MODEL value is what `--model` accepts.                                                     |
| `duvo agents set-model <id> [--model <model>] [--json]`                                                                                                                                                                                         | Switch the Claude model used by an agent's latest revision. Interactive picker when `--model` is omitted in a TTY.                       |
| `duvo agents delete <id> [-y] [--json]`                                                                                                                                                                                                         | Delete an agent. Active runs are interrupted. Destructive — prompts for confirmation.                                                    |
| `duvo agents move <id> --to-team <id> [--all-together] [--skip-learnings] [--dry-run] [-y] [--json]`                                                                                                                                            | Move an agent (and its connected workspace) to another team. `--all-together` carries connected queues/agents; `--dry-run` previews.     |

### Schedules (per agent)

| Command                                                                                                                                                                                                                                                                                     | Purpose                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo agents schedules list <agent-id> [--json]`                                                                                                                                                                                                                                            | List schedules you own for an agent (`duvo agents schedules <agent-id>` also works as an alias).                                                                                   |
| `duvo agents schedules create <agent-id> --frequency <f> --timezone <tz> [--time <HH:MM>] [--day <day>] [--day-of-month <n>] [--cron <expr>] [--disabled] [--no-recurring] [--json]`                                                                                                        | Create a schedule. Frequencies: `hourly`, `daily`, `workday`, `weekly`, `monthly`, `custom`. `--cron` is required for `custom`; `--time` for `daily`/`workday`/`weekly`/`monthly`. |
| `duvo agents schedules update <agent-id> <schedule-id> [--frequency <f>] [--timezone <tz>] [--time <HH:MM>] [--clear-time] [--day <day>] [--clear-day] [--day-of-month <n>] [--clear-day-of-month] [--cron <expr>] [--clear-cron] [--enable\|--disable] [--recurring\|--one-shot] [--json]` | Update schedule fields. Only supplied fields change.                                                                                                                               |
| `duvo agents schedules delete <agent-id> <schedule-id> [-y] [--json]`                                                                                                                                                                                                                       | Delete a schedule. Destructive — prompts for confirmation.                                                                                                                         |

### Case triggers (per agent)

| Command                                                                                                                                            | Purpose                                                                                                  |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `duvo agents case-triggers list <agent-id> [--json]`                                                                                               | List case triggers configured for an agent.                                                              |
| `duvo agents case-triggers get <agent-id> <trigger-id> [--json]`                                                                                   | Get a case trigger by ID.                                                                                |
| `duvo agents case-triggers create <agent-id> --queue <id> [--enabled] [--concurrency <n>] [--json]`                                                | Create a case trigger bound to a queue (created disabled unless `--enabled`).                            |
| `duvo agents case-triggers update <agent-id> <trigger-id> [--queue <id>] [--enable\|--disable] [--concurrency <n>] [--clear-concurrency] [--json]` | Toggle, rebind, or rescale a trigger.                                                                    |
| `duvo agents case-triggers delete <agent-id> <trigger-id> [-y] [--json]`                                                                           | Delete a case trigger.                                                                                   |
| `duvo agents case-triggers preview <agent-id> --queue <id> [--json]`                                                                               | Preview whether adding a case trigger for this agent would conflict with existing triggers on the queue. |

### Event triggers (per agent)

`duvo agents triggers …` manages event-driven triggers (email received, file uploaded, etc.) for an
agent's connected integrations. The integration must already be connected.

| Command                                                                                                                         | Purpose                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo agents triggers list <agent-id> [--json]`                                                                                 | List the event triggers you own for an agent.                                                                                                         |
| `duvo agents triggers set <agent-id> --integration <slug> --trigger-type <type> [--filter-config <json>] [--disabled] [--json]` | Create or update an event trigger. `--filter-config` is a JSON object with integration-specific filter settings. Created enabled unless `--disabled`. |
| `duvo agents triggers types <agent-id> [--json]`                                                                                | List the trigger types available for an agent by integration — use this to discover valid `--integration` and `--trigger-type` values for `set`.      |

### Agent memory

| Command                                                  | Purpose                                                                                                              |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `duvo agents memory files <agent-id> [--json]`           | List the memory files stored for an agent (`duvo agents memory <agent-id>` also works as an alias).                  |
| `duvo agents memory file-get <agent-id> <path> [--json]` | Print the contents of one memory file. Streams to stdout. Path is relative (e.g. `notes.md`, `context/customer.md`). |

### Evaluation scores (per agent)

| Command                                                                                                    | Purpose                                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo agents eval-scores <agent-id> [--since <iso>] [--scope <all\|custom>] [--revision-id <id>] [--json]` | Aggregate evaluation scores for an agent's runs. `--since` defaults to the last 30 days; `--revision-id` only applies with `--scope=custom`. |

### Evaluation rubrics (per agent, max 5 per revision)

`duvo agents eval-rubrics …` manages an agent's custom Pass/Fail evaluation rubrics.

| Command                                                                                                     | Purpose                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo agents eval-rubrics list <agent-id> [--build <build-id>] [--json]`                                    | List an agent's active custom rubrics (defaults to the live build).                                                                                              |
| `duvo agents eval-rubrics add <agent-id> --title <text> --description <text> [--build <build-id>] [--json]` | Add one rubric. Fails once the revision already has 5.                                                                                                           |
| `duvo agents eval-rubrics replace <agent-id> --input <path\|-> [--build <build-id>] [-y] [--json]`          | Replace the whole rubric set from a JSON array of `{ "title", "description" }` objects (max 5; empty array clears them). Destructive — prompts for confirmation. |
| `duvo agents eval-rubrics update <agent-id> <rubric-id> [--title <text>] [--description <text>] [--json]`   | Edit a rubric's title/description. Creates a new rubric so prior runs keep the original evaluation; returns the new rubric.                                      |
| `duvo agents eval-rubrics remove <agent-id> <rubric-id> [-y] [--json]`                                      | Remove a custom rubric. Destructive — prompts for confirmation.                                                                                                  |

## Agent folders

| Command                                                                                            | Purpose                                                        |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `duvo agent-folders list [--json]`                                                                 | List all agent folders.                                        |
| `duvo agent-folders create --name <n> [--parent <id>] [--json]`                                    | Create a folder, optionally nested.                            |
| `duvo agent-folders update <id> [--name <n>] [--parent <id>\|--move-to-root] [--json]`             | Rename or move a folder.                                       |
| `duvo agent-folders delete <id> [--force] [-y] [--json]`                                           | Delete a folder. `--force` allows deleting a non-empty folder. |
| `duvo agent-folders move-agents --agent <id> [--agent <id> …] (--folder <id>\|--to-root) [--json]` | Move one or more agents into a folder, or to the root.         |

## Revisions (versioned agent configs)

| Command                                                                             | Purpose                                                                 |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `duvo revisions list --agent <id> [--limit <n>] [--offset <n>] [--json]`            | List revisions for an agent.                                            |
| `duvo revisions get <revision-id> --agent <id> [--json]`                            | Get a revision by ID.                                                   |
| `duvo revisions create --agent <id> --name <text> --config-file <path\|-> [--json]` | Create a revision from a config file. Pass `-` to read from stdin.      |
| `duvo revisions update <revision-id> --agent <id> [--json]`                         | Update a revision's config (handover targets are derived from the AOP). |
| `duvo revisions promote <revision-id> --agent <id> [--description <text>] [--json]` | Promote a revision to the agent's live build.                           |

### Revision integrations

`duvo revision-integrations …` manages the integration slots on a
revision and the connections pinned to each slot.

| Command                                                                                                                      | Purpose                                                             |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `duvo revision-integrations list --agent <id> --revision <id> [--json]`                                                      | List integrations attached to a revision.                           |
| `duvo revision-integrations attach --agent <id> --revision <id> --integration <slug> […] [--json]`                           | Attach one or more integrations to a revision.                      |
| `duvo revision-integrations remove <integration-id> --agent <id> --revision <id> [-y] [--json]`                              | Remove an integration slot from a revision.                         |
| `duvo revision-integrations connections list --agent <id> --revision <id> --integration <id> [--json]`                       | List your pinned connections for a slot.                            |
| `duvo revision-integrations connections pin <connection-id> --agent <id> --revision <id> --integration <id> [--json]`        | Pin one of your connections to a revision's integration slot.       |
| `duvo revision-integrations connections unpin <connection-id> --agent <id> --revision <id> --integration <id> [-y] [--json]` | Unpin a connection from a slot.                                     |
| `duvo revision-integrations queues list --agent <id> --revision <id> --integration <id> [--json]`                            | List the queues linked to a queue integration slot.                 |
| `duvo revision-integrations queues set --agent <id> --revision <id> --integration <id> [--queue <id> …] [--json]`            | Replace the queues linked to a slot (omit `--queue` to unlink all). |
| `duvo revision-integrations case-queue-setup --agent <id> --revision <id> [--json]`                                          | Check that every queue slot on a revision points at a queue.        |

### Revision-integration gotchas

- **`attach` creates the slot only.** OAuth and user-provided slots
  (HubSpot, Gong, Slack, custom MCP, …) additionally need one of the
  user's connections pinned (`connections pin`) or runs fail at
  runtime with "not connected". Case-queue slots need a queue mapped
  (`queues set`) instead of a pin.
- **Verify, don't assume.** `case-queue-setup` checks queue slots
  (`linked_queue_count` must be > 0); `connections list` checks a
  pinned connection exists for an OAuth/user slot (expect ≥ 1 entry).
  Neither `attach` nor `pin` failing to happen produces an error
  anywhere until a run dies. See workflows 4 and 7 in `workflows.md`.

## Runs

| Command                                                                                                                                                                                                               | Purpose                                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `duvo runs list [--limit <n>] [--offset <n>] [--sort-order <asc\|desc>] [--status <s>] [--agent <id>] [--user <id>] [--case-queue <id>] [--source <source>] [--search <query>] [--has-issues <true\|false>] [--json]` | List runs for your team with filters.                                                                                                                        |
| `duvo runs get <id> [--json]`                                                                                                                                                                                         | Get an agent run by ID.                                                                                                                                      |
| `duvo runs start --agent <id> [--message <text>] [--sandbox-id <id>] [--webhook-url <url>] [--follow] [--json]`                                                                                                       | Start a new run. `--follow` polls for and streams messages until the run ends. `--sandbox-id` pre-stages files; `--webhook-url` posts events asynchronously. |
| `duvo runs messages <id> [--limit <n>] [--offset <n>] [--follow] [--json]`                                                                                                                                            | List messages for a run. `--follow` polls for new messages until the run reaches a terminal state (streams NDJSON when combined with `--json`).              |
| `duvo runs evaluation <run-id> [--agent <agent-id>] [--json]`                                                                                                                                                         | Get the latest evaluation for a run. The agent ID is resolved automatically when `--agent` is omitted. Also accessible as `duvo runs eval`.                  |
| `duvo runs send-message <id> --message <text> [--json]`                                                                                                                                                               | Post a follow-up message to an in-progress run.                                                                                                              |
| `duvo runs respond <run-id> [request-id] [--approve\|--deny\|--answer <question=answer>] [--json]`                                                                                                                    | Respond to a human-in-the-loop request. `request-id` defaults to the run's pending request. `--answer` is repeatable for multi-question forms.               |
| `duvo runs stop <id> [-y] [--json]`                                                                                                                                                                                   | Stop a running agent run. Destructive — prompts for confirmation.                                                                                            |

## Queues, queue labels, cases

### Queues

| Command                                                                                                                                        | Purpose                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo queues list [--limit <n>] [--offset <n>] [--json]`                                                                                       | List queues.                                                                                                                                                  |
| `duvo queues get <id> [--json]`                                                                                                                | Get a queue by ID.                                                                                                                                            |
| `duvo queues create --name <name> [--description <text>] [--folder-id <id>] [--json]`                                                          | Create a queue.                                                                                                                                               |
| `duvo queues update <id> [--name <n>] [--description <text>\|--clear-description] [--folder-id <id>\|--clear-folder] [--json]`                 | Update a queue.                                                                                                                                               |
| `duvo queues delete <id> [-y] [--json]`                                                                                                        | Delete a queue. Interrupts active runs and removes all cases.                                                                                                 |
| `duvo queues agents <queue-id> [--json]`                                                                                                       | List producer/consumer agents bound to a queue.                                                                                                               |
| `duvo queues link-agent <queue-id> --agent <id> --role <producer\|consumer> [--enable-trigger] [--concurrency <n>] [--revision <id>] [--json]` | Link an agent to a queue in a role. Attaches the case-queue integration on the live revision and maps the queue; for consumers also creates the case trigger. |
| `duvo queues unlink-agent <queue-id> --agent <id> --role <producer\|consumer> [--revision <id>] [--json]`                                      | Reverse of `link-agent`: unmaps the queue from the role's slot; for consumers also removes the case trigger for that queue.                                   |

### Queue labels (label vocabulary on a queue)

| Command                                                                                                           | Purpose                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `duvo queue-labels list --queue <id> [--limit <n>] [--offset <n>] [--json]`                                       | List labels with case counts. `--limit` is 1-1000 (default 1000).                                                  |
| `duvo queue-labels create --queue <id> --value <text> [--key <text>] [--color-hue <0-360>] [--json]`              | Create a label without assigning it. Omit `--key` for a tag-only label.                                            |
| `duvo queue-labels update <label-id> --queue <id> [--key <text>] [--value <text>] [--color-hue <0-360>] [--json]` | Rename a label, change its key, or change its color. A `--color-hue`-only edit re-sends the current value for you. |
| `duvo queue-labels delete <label-id> --queue <id> [-y] [--json]`                                                  | Delete a label and unlink it from all cases.                                                                       |

### Cases

| Command                                                                                                                                                                                                     | Purpose                                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `duvo cases list --queue <id> [--limit <n>] [--offset <n>] [--status <list>] [--search <text>] [--created-at-from <iso>] [--updated-at-from <iso>] [--sort-by <field>] [--sort-order <asc\|desc>] [--json]` | List cases in a queue with filters.                                                                                           |
| `duvo cases get <id> [--json]`                                                                                                                                                                              | Get a case by ID with its event history.                                                                                      |
| `duvo cases create --queue <id> [--title <text>] [--data <text>] [--label <key=value>] [--from-file <path\|->] [--json]`                                                                                    | Create one case (with `--title`) or up to 100 (with `--from-file`).                                                           |
| `duvo cases delete <id> [-y] [--json]`                                                                                                                                                                      | Delete a case. Interrupts associated active runs.                                                                             |
| `duvo cases clear --queue <id> [-y] [--json]`                                                                                                                                                               | Delete every case in a queue.                                                                                                 |
| `duvo cases bulk-delete --queue <id> --ids <list> [-y] [--json]`                                                                                                                                            | Delete multiple cases (1-100, comma-separated IDs).                                                                           |
| `duvo cases bulk-reprocess --queue <id> --agent <id> --ids <list> [-y] [--json]`                                                                                                                            | Re-process cases on a queue consumer agent (interrupts active runs).                                                          |
| `duvo cases bulk-update-status --queue <id> --ids <list> --status <completed\|failed> [-y] [--json]`                                                                                                        | Bulk-update case statuses.                                                                                                    |
| `duvo cases runs <case-id> [--json]`                                                                                                                                                                        | List runs that have worked on a case.                                                                                         |
| `duvo cases runs recent-messages <case-id> <run-id> [--json]`                                                                                                                                               | Recent messages for one run on a case.                                                                                        |
| `duvo cases labels list <case-id> --queue <id> [--json]`                                                                                                                                                    | List labels assigned to a case.                                                                                               |
| `duvo cases labels assign <case-id> --queue <id> --label <"key=value"\|"value"> [--label …] [--json]`                                                                                                       | Assign one or more labels to a case (repeatable).                                                                             |
| `duvo cases labels unlink <case-id> --queue <id> [--label-id <id> …] [--label <"key=value"\|"value"> …] [--json]`                                                                                           | Remove labels from a case by ID or by key=value (resolved against the case's labels). At least one of `--label-id`/`--label`. |

`cases create --from-file` accepts a single case object or an array of
up to 100. Shape per case:

```json
{
  "title": "Case title",
  "data": "Optional free-form data passed to the agent on claim",
  "labels": [{ "key": "priority", "value": "high" }, { "value": "urgent" }]
}
```

Labels are `{"key": "…", "value": "…"}` — omit `key` (or set it to
`""`) for a tag-only label. Pass `-` as the path to read from stdin.

### Queue gotchas

- **Trigger-enable flag differs by command.** `duvo agents case-triggers
create` takes `--enabled`; `duvo agents case-triggers update` takes
  `--enable` / `--disable`; `duvo queues link-agent` takes
  `--enable-trigger`. Three commands, three spellings — check `--help`.
- **Attach can lag a fresh build.** `revision-integrations attach`
  called right after `agents create` can return `count: 0` while the
  build is still settling. `queues link-agent` re-attaches and verifies
  the slot for you; raw `attach` callers should re-check before mapping
  queues.
- **Array bodies in the escape hatch.** `duvo api` builds JSON arrays
  from repeated `-F "queue_ids[]=…"` flags, or from a `--input -` body
  like `{"queue_ids":["…"]}`. A single non-`[]` field does not become an
  array. See `api-escape-hatch.md`.

## Files & sandboxes

### Team files (persistent)

| Command                                                                            | Purpose                                                                                                                                  |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo files list [--json]`                                                         | List team files.                                                                                                                         |
| `duvo files upload-url --file-name <name> --content-type <mime> [--json]`          | Get a presigned upload URL for a new team file. Response carries `filePath` and `signedUrl`.                                             |
| `duvo files download-url <path> [--json]`                                          | Get a presigned download URL.                                                                                                            |
| `duvo files content <path> [--json]`                                               | Print the content of a text file (binary files rejected).                                                                                |
| `duvo files content-set <path> [--content <text>\|--content-file <path>] [--json]` | Update the content of a text file.                                                                                                       |
| `duvo files rename --path <path> --new-name <name> [--json]`                       | Rename a team file's filename only (extension is preserved automatically). `--new-name` must not contain path separators or be `.`/`..`. |
| `duvo files delete <path> [-y] [--json]`                                           | Delete a team file.                                                                                                                      |

### Sandboxes (per-run staging)

| Command                                                              | Purpose                                          |
| -------------------------------------------------------------------- | ------------------------------------------------ |
| `duvo sandboxes create [--json]`                                     | Create a sandbox for staging files before a run. |
| `duvo sandboxes files <sandbox_id> [--path <dir>] [--json]`          | List files in a sandbox directory.               |
| `duvo sandboxes upload <sandbox_id> <file> [--path <dest>] [--json]` | Upload a local file (max 10 MB).                 |
| `duvo sandboxes prepare-upload-url <sandbox_id> --path <p> [--json]` | Get a presigned upload URL for larger files.     |

Pass the sandbox ID to `duvo runs start --sandbox-id <id>` to make the
files available to the run.

## Integrations & connections

### Team integration catalog

| Command                                                                                                                                                   | Purpose                                                                                                                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo integrations list [--json]`                                                                                                                         | List the team's integration catalog.                                                                                                                    |
| `duvo integrations custom create --name <n> --auth-method <method> --server-url <url> [--oauth-client-id <id>] [--oauth-client-secret <secret>] [--json]` | Create a custom integration type in the team catalog. `--oauth-client-id`/`--oauth-client-secret` are paired and only valid with `--auth-method oauth`. |
| `duvo integrations custom delete <custom-integration-id> [-y] [--json]`                                                                                   | Delete a custom integration type. Cascade-removes related connections.                                                                                  |

### Your connections

| Command                                                                                                                                                                                        | Purpose                                                                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo connections list [--type <type>] [--json]`                                                                                                                                               | List your connections (filter by integration type).                                                                                                                   |
| `duvo connections get <id> [--json]`                                                                                                                                                           | Get one of your connections by ID.                                                                                                                                    |
| `duvo connections create --name <n> --auth-method <m> [--server-url <url>] [--type <slug>] [--custom-integration-id <uuid>] [--header KEY=VALUE …] [--secret-header KEY=VALUE …] [--json]`     | Create a user-provided connection (custom MCP server). `--header` / `--secret-header` are repeatable. OAuth-based integrations must use `duvo oauth …` instead.       |
| `duvo connections update <id> [--name <n>] [--server-url <url>] [--auth-method <method>] [--header KEY=VALUE …] [--secret-header KEY=VALUE …] [--clear-headers] [--share\|--unshare] [--json]` | Update a connection. `--share` is manager+ only. Sending an empty `--secret-header` value keeps the existing secret.                                                  |
| `duvo connections delete <id> [-y] [--json]`                                                                                                                                                   | Delete a connection and any triggers bound to it.                                                                                                                     |
| `duvo connections credentials <id> [--json]`                                                                                                                                                   | List configured header keys (sensitive values redacted).                                                                                                              |
| `duvo connections probe [--url <url>\|--slug <slug>] [--header KEY=VALUE …] [--json]`                                                                                                          | Probe an MCP server and list the tools it exposes. `--url` for custom MCP servers; `--slug` for catalog integrations (the backend resolves the URL and auth headers). |

### OAuth flows

| Command                                                                                                                                                                   | Purpose                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo oauth native start <provider> [--return-url <url>] [--reconnect-instance-id <id>] [--json]`                                                                         | Start a native OAuth connection (Gmail, Google Sheets, Outlook, …). Prints the authorization URL. `--reconnect-instance-id` re-authorizes an existing connection instead of creating a new one. |
| `duvo oauth mcp probe <url> [--header KEY=VALUE …] [--json]`                                                                                                              | Probe an MCP server URL and list its tools (same as `duvo connections probe`).                                                                                                                  |
| `duvo oauth mcp check --url <url> [--json]`                                                                                                                               | Probe an MCP server for OAuth Dynamic Client Registration support.                                                                                                                              |
| `duvo oauth mcp authorize --url <url> --name <n> [--custom-integration-id <id>] [--integration-type <type>] [--return-url <url>] [--reconnect-instance-id <id>] [--json]` | Start an OAuth-based connection with a remote MCP server via DCR. Prints the authorization URL. `--reconnect-instance-id` re-authorizes an existing connection instead of creating a new one.   |

## Clarity

Clarity commands auto-detect process version where possible. Legacy v1
processes support only `overview`, `status`, `captures`, `capture`, and
`export`. V2 snapshot and write commands require a Clarity v2 process.

### Read and inspect

| Command                                                                                                                         | Purpose                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo clarity list [--limit <n>] [--offset <n>] [--search <text>] [--status <s>] [--process-version <1\|2>] [--json] [--csv]`   | List Clarity processes for the current team. `--process-version` filters by schema version.                                                                             |
| `duvo clarity search <query> [--limit <n>] [--offset <n>] [--status <s>] [--process-version <1\|2>] [--json] [--csv]`           | Search Clarity processes by name. `--process-version` filters by schema version.                                                                                        |
| `duvo clarity get <process-id> [--json]`                                                                                        | Get a v2 Clarity process read model by ID (name, status, version, timestamps). Prefer `overview` for a fuller summary.                                                  |
| `duvo clarity status <process-id> [--json]`                                                                                     | Show process status, generation progress, selected version IDs, and health warnings.                                                                                    |
| `duvo clarity overview <process-id> [--current <selector>] [--proposal <selector>]`                                             | Show a summary-first process overview. Add `--json`, `--markdown`, or `--include-transcripts`.                                                                          |
| `duvo clarity versions <process-id> [--json]`                                                                                   | List v2 current-process and transformation-proposal version histories.                                                                                                  |
| `duvo clarity current <process-id> [--snapshot <selector>] [--json]`                                                            | Read a v2 current-process snapshot.                                                                                                                                     |
| `duvo clarity get-current-process <process-id> <snapshot-id> [--json]`                                                          | Get one current-process snapshot by exact ID (use `duvo clarity versions` to list IDs).                                                                                 |
| `duvo clarity proposal <process-id> [--snapshot <selector>] [--json]`                                                           | Read a v2 transformation-proposal snapshot.                                                                                                                             |
| `duvo clarity get-transformation-proposal <process-id> <proposal-id> [--json]`                                                  | Get one transformation-proposal snapshot by exact ID.                                                                                                                   |
| `duvo clarity list-snapshots <process-id> <current_process\|transformation_proposal> [--limit <n>] [--offset <n>] [--json]`     | List lightweight v2 snapshot versions of one kind for a process.                                                                                                        |
| `duvo clarity compare <process-id> [--current <selector>] [--proposal <selector>]`                                              | Compare selected current-process and transformation-proposal snapshots.                                                                                                 |
| `duvo clarity captures <process-id> [--include-transcripts] [--json] [--csv]`                                                   | List captures attached to a process.                                                                                                                                    |
| `duvo clarity capture <process-id> <capture-id> [--include-transcripts] [--json]`                                               | Read one capture attached to a process.                                                                                                                                 |
| `duvo clarity gaps <process-id> [--json]`                                                                                       | Group v2 proposal gaps and extra-capture requests by step.                                                                                                              |
| `duvo clarity evidence <process-id> [--citation <id>] [--json]`                                                                 | Print current-process evidence sources, or resolve one citation ID.                                                                                                     |
| `duvo clarity readiness <process-id> [--json]`                                                                                  | Summarize v2 automation-readiness ratings.                                                                                                                              |
| `duvo clarity facets <process-id> [--json]`                                                                                     | Build structured cost, risk, lineage, and automation facets for agents and scripts.                                                                                     |
| `duvo clarity export <process-id> [--json] [--include-transcripts]`                                                             | Export Clarity context as Markdown or JSON.                                                                                                                             |
| `duvo clarity doctor [process-id] [--json]`                                                                                     | Check auth, API reachability, command availability, and optional process-level context health.                                                                          |
| `duvo clarity tools [--json]`                                                                                                   | List the Clarity CLI tool map with commands and underlying public API endpoints.                                                                                        |
| `duvo clarity process-summaries [--limit <n>] [--offset <n>] [--detail <skeleton\|summary>] [--include-current-steps] [--json]` | List lightweight summaries for all Clarity processes accessible to the current user. `--detail skeleton` (default) omits step bodies; `--detail summary` includes them. |
| `duvo clarity list-extra-capture-requests <process-id> <proposal-id> [--limit <n>] [--offset <n>] [--json]`                     | List extra-capture requests attached to a transformation-proposal snapshot.                                                                                             |
| `duvo clarity artifact-chat-conversations <process-id> --snapshot-kind <kind> [--json]`                                         | List artifact-chat conversations for a current-process or transformation-proposal view.                                                                                 |
| `duvo clarity artifact-chat-messages <process-id> <conversation-id> [--json]`                                                   | Read messages for one artifact-chat conversation.                                                                                                                       |

Snapshot selectors are `live`, `latest`, or an exact snapshot ID from
`duvo clarity versions`.

### Write and mutate

| Command                                                                                                                                                                                 | Purpose                                                                                                                                                   |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo clarity generate-current-process <process-id> [--json]`                                                                                                                           | Start asynchronous generation for a new current-process snapshot.                                                                                         |
| `duvo clarity generate-transformation-proposal <process-id> [--current-process-id <id>] [--regenerate-from <id>] [--json]`                                                              | Start transformation-proposal generation, anchored to a current snapshot or regenerated from an existing proposal. Pass only one of the two optional IDs. |
| `duvo clarity save-current-process <process-id> --baseline-snapshot-id <id> --steps-file <path\|-> [--json]`                                                                            | Save edited current-process steps as the live snapshot. The steps file is a JSON array of steps, or an object with `steps` and narrative roots.           |
| `duvo clarity save-transformation-proposal <process-id> --baseline-snapshot-id <id> --steps-file <path\|-> [--current-process-id <id>] [--json]`                                        | Save edited transformation-proposal steps as the live snapshot. `--current-process-id` anchors the snapshot the edited proposal expects.                  |
| `duvo clarity send-artifact-chat-message <process-id> [--conversation-id <id>] --snapshot-kind <kind> [--baseline-snapshot-id <id>] [--message <text>] [--answer key=value …] [--json]` | Send an artifact-chat message, or answer a pending HITL question with `--answer` (repeatable, requires `--conversation-id`).                              |
| `duvo clarity decide-artifact-chat-patch <process-id> <message-id> [--accept\|--decline] [--json]`                                                                                      | Accept or decline an artifact-chat patch proposal.                                                                                                        |
| `duvo clarity promote-current-process <process-id> <snapshot-id> [--json]`                                                                                                              | Promote a current-process snapshot to the live version shown for the process.                                                                             |
| `duvo clarity promote-transformation-proposal <process-id> <proposal-id> [--json]`                                                                                                      | Promote a transformation-proposal snapshot to the live proposal.                                                                                          |
| `duvo clarity revert-current-process <process-id> <snapshot-id> [--json]`                                                                                                               | Delete a current-process snapshot when it should be reverted or removed.                                                                                  |
| `duvo clarity revert-transformation-proposal <process-id> <proposal-id> [--json]`                                                                                                       | Delete a transformation-proposal snapshot when it should be reverted or removed.                                                                          |
| `duvo clarity assign-extra-capture-request <process-id> <proposal-id> <request-id> [--user-id <id>] [--json]`                                                                           | Assign an extra-capture request to a team member with `--user-id`; omit it to unassign the request.                                                       |
| `duvo clarity postprocess <process-id> <snapshot-type> <snapshot-id> [--json]`                                                                                                          | Re-run postprocessing for `current_process` or `transformation_proposal`.                                                                                 |
| `duvo clarity build-automation <process-id> [--json]`                                                                                                                                   | Start automation generation from the latest transformation proposal.                                                                                      |
| `duvo clarity stop-current-process <process-id> [--json]`                                                                                                                               | Stop an in-flight current-process generation job.                                                                                                         |
| `duvo clarity stop-transformation-proposal <process-id> [--json]`                                                                                                                       | Stop an in-flight transformation-proposal generation job.                                                                                                 |
| `duvo clarity create-invite-link <process-id> [--json]`                                                                                                                                 | Create or regenerate a Clarity interview invite link that can be sent to someone.                                                                         |
| `duvo clarity import-artifact <process-id> <file> [--content-type <mime>] [--extra-capture-request-id <id>] [--json]`                                                                   | Import a local Miro SVG, XML, PNG, or JPEG export by creating a signed upload URL, uploading bytes, and completing the import.                            |

Supported artifact import content types are `image/svg+xml`,
`application/xml`, `text/xml`, `image/png`, and `image/jpeg`.

### Clarity process tags

`duvo clarity process-labels …` manages an org-scoped tag palette and the tags
assigned to individual processes. Every subcommand accepts `--org <org-id>` or
falls back to `DUVO_ORG_ID`.

| Command                                                                                                                | Purpose                                                                            |
| ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `duvo clarity process-labels list [--org <org-id>] [--limit <n>] [--offset <n>] [--search <text>] [--json]`            | List the org's process-tag palette.                                                |
| `duvo clarity process-labels create --value <text> [--org <org-id>] [--color-hue <0-360>] [--json]`                    | Create a process tag in the org palette.                                           |
| `duvo clarity process-labels update <label-id> [--org <org-id>] [--value <text>] [--color-hue <0-360>] [--json]`       | Rename a process tag or change its color.                                          |
| `duvo clarity process-labels delete <label-id> [--org <org-id>] [-y] [--json]`                                         | Delete a process tag from the org palette. Destructive — prompts for confirmation. |
| `duvo clarity process-labels list-process --process <process-id> [--json]`                                             | List tags assigned to a process.                                                   |
| `duvo clarity process-labels available --process <process-id> [--limit <n>] [--offset <n>] [--search <text>] [--json]` | List tags available to assign to a process.                                        |
| `duvo clarity process-labels assign --process <process-id> --label <label-id> […] [--json]`                            | Assign one or more tags to a process (repeatable `--label`).                       |
| `duvo clarity process-labels unlink --process <process-id> --label <label-id> […] [--json]`                            | Remove one or more tags from a process (repeatable `--label`).                     |

### Clarity process landscape

`duvo clarity landscape …` reads and curates the org-level Clarity process
landscape (hierarchical tree of area nodes and process nodes). Every
subcommand requires `--org <org-id>` or the `DUVO_ORG_ID` environment
variable, and an OAuth profile with an org-level role (API-key profiles have
no org visibility).

**Read commands:**

| Command                                                                                           | Purpose                                                             |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `duvo clarity landscape get [--org <org-id>] [--root <node-id>] [--json]`                         | Read the full process landscape (or a subtree). Default subcommand. |
| `duvo clarity landscape tree [--org <org-id>] [--root <node-id>] [--json]`                        | Read the raw process tree (depth-first flat list with depth field). |
| `duvo clarity landscape search <query> [--org <org-id>] [--root <node-id>] [--json]`              | Filter landscape nodes by name, owner, process name, or id.         |
| `duvo clarity landscape node <node-id> [--org <org-id>] [--json]`                                 | Read one landscape node by ID.                                      |
| `duvo clarity landscape captures [--org <org-id>] [--limit <n>] [--include-transcripts] [--json]` | List Clarity captures eligible for the process landscape.           |

**Write commands:**

| Command                                                                                                                                      | Purpose                                                                 |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `duvo clarity landscape create-area --name <name> [--org <org-id>] [--parent <node-id>] [--owner-label <label>] [--team <team-id>] [--json]` | Create an area node in the process landscape.                           |
| `duvo clarity landscape rename-node <node-id> [--org <org-id>] [--name <name>] [--owner-label <label>] [--clear-owner-label] [--json]`       | Rename a landscape node or update its owner label.                      |
| `duvo clarity landscape move-node <node-id> [--org <org-id>] [--parent <node-id>\|--root] [--json]`                                          | Move a node under a new parent or to the root.                          |
| `duvo clarity landscape delete-node <node-id> [--org <org-id>] [--json]`                                                                     | Delete a landscape node and its entire subtree.                         |
| `duvo clarity landscape generate [--org <org-id>] [--json]`                                                                                  | Start async generation of the process landscape from eligible captures. |
| `duvo clarity landscape propose-process --name <name> [--org <org-id>] [--description <text>] [--parent <node-id>] [--json]`                 | Propose a new process node in the landscape.                            |

### Clarity process links

`duvo clarity process-links …` manages directed links between landscape
process nodes. Every subcommand requires `--org <org-id>` or `DUVO_ORG_ID`.
Link types: `hands_off_to`, `shares_step_with`, `variant_of`. Link states:
`suggested`, `confirmed`.

| Command                                                                                                                                                                                       | Purpose                                                                                        |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `duvo clarity process-links list [--org <org-id>] [--node <node-id>] [--limit <n>] [--offset <n>] [--json]`                                                                                   | List process links in the organization. `--node` filters to links touching a specific node.    |
| `duvo clarity process-links create --source-node <id> --target-node <id> --type <type> [--org <org-id>] [--state <state>] [--confidence <0-1>] [--json]`                                      | Create a process link. Returns the existing link if one already exists for the same node pair. |
| `duvo clarity process-links update <link-id> [--org <org-id>] [--source-node <id>] [--target-node <id>] [--type <type>] [--state <state>] [--confidence <0-1>] [--clear-confidence] [--json]` | Update fields on an existing process link. Only supplied fields change.                        |
| `duvo clarity process-links delete <link-id> [--org <org-id>] [--json]`                                                                                                                       | Delete a process link.                                                                         |

## Pulse

`duvo pulse …` manages Pulse dashboards — AI-generated reports that synthesize
your team's data into a live HTML document. Each dashboard has an owner
(creator) and a version history; sending a message triggers a new generation.

| Command                                                                                                 | Purpose                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo pulse list [--team] [--limit <n>] [--offset <n>] [--json]`                                        | List your Pulse dashboards. `--team` lists dashboards a teammate published to your whole team instead of your own. `--shared` is deprecated and always returns no dashboards — use `--team`. |
| `duvo pulse create --message <text> [--json]`                                                           | Create a new Pulse dashboard from a prompt (max 10000 chars). Returns metadata including the new dashboard ID. Generation runs asynchronously — poll `pulse get`.                            |
| `duvo pulse get <id> [--json]`                                                                          | Get a Pulse dashboard's metadata and generation status.                                                                                                                                      |
| `duvo pulse html <id> [--version <version-id>]`                                                         | Print the rendered HTML document to stdout. Pipe to a file or open in a browser. `--version` pins to a historical version.                                                                   |
| `duvo pulse pdf <id> [-o\|--output <file>] [--theme <light\|dark>]`                                     | Download a Pulse dashboard as a PDF (default output `<id>.pdf`).                                                                                                                             |
| `duvo pulse snapshot <id> [-o\|--output <file>] [--theme <light\|dark>]`                                | Download a self-contained static HTML snapshot with data baked in (default output `<id>.html`).                                                                                              |
| `duvo pulse send-message <id> --message <text> [--json]`                                                | Send a follow-up instruction to a Pulse dashboard (max 10000 chars), triggering a new generation. Creator only.                                                                              |
| `duvo pulse stop <id> [--json]`                                                                         | Stop an in-flight dashboard generation turn (creator only). No-op if nothing is generating.                                                                                                  |
| `duvo pulse refresh <id> [--json]`                                                                      | Refresh a dashboard's data in place. Runs in the background — poll `pulse get` for status.                                                                                                   |
| `duvo pulse rename <id> --title <text> [--json]`                                                        | Rename a dashboard (1-120 chars, creator only).                                                                                                                                              |
| `duvo pulse share <id> --visibility <private\|team\|organization> [--permission <view\|edit>] [--json]` | Share a dashboard with your team or organization, or revert to private (creator only). `--permission` only applies when sharing.                                                             |
| `duvo pulse duplicate <id> [--json]`                                                                    | Duplicate a dashboard into a new idle clone. Conversation, shares, and connections are not copied.                                                                                           |
| `duvo pulse versions <id> [--json]`                                                                     | List the version history for a Pulse dashboard, newest first.                                                                                                                                |
| `duvo pulse restore <id> <version-id> [--json]`                                                         | Restore a Pulse dashboard to a previous version. The restored version becomes the new live version.                                                                                          |
| `duvo pulse messages <id> [--limit <n>] [--before <message-id>] [--json]`                               | List a dashboard's chat transcript (creator only).                                                                                                                                           |
| `duvo pulse answer <id> --tool-call-id <id> --answer key=value […] [--json]`                            | Answer a pending HITL question from the Pulse agent and resume the run (requires edit access). Find the tool call ID and question keys via `pulse messages --json`.                          |
| `duvo pulse delete <id> [-y] [--json]`                                                                  | Delete a Pulse dashboard (creator only). Destructive — prompts for confirmation.                                                                                                             |

### Pulse connections (data sources)

| Command                                                       | Purpose                                                                                     |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `duvo pulse connections list <id> [--json]`                   | List the connections attached to a Pulse dashboard.                                         |
| `duvo pulse connections attach <id> <connection-id> [--json]` | Attach one of your connections to a dashboard so the agent can use its data (creator only). |
| `duvo pulse connections detach <id> <connection-id> [--json]` | Detach a connection from a dashboard (creator only).                                        |

## Skills & plugins

| Command                                                                                                                                                                       | Purpose                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo skills list [--system] [--json]`                                                                                                                                        | List skills available to your team (or system skills only).                                                                           |
| `duvo skills system [--json]`                                                                                                                                                 | List global system skills available to all teams.                                                                                     |
| `duvo skills install <skill-id> [--json]`                                                                                                                                     | Copy a system skill (or any accessible skill) into your team's skill library.                                                         |
| `duvo skills create --name <n> --description <d> [--content <text>\|--content-file <path>] [--license <license>] [--compatibility <text>] [--allowed-tools <tools>] [--json]` | Create a skill from Markdown content.                                                                                                 |
| `duvo skills upload <file> [--json]`                                                                                                                                          | Create or update a skill by uploading a `SKILL.md` or a ZIP archive.                                                                  |
| `duvo skills delete <skill-id> [-y] [--json]`                                                                                                                                 | Delete a custom skill.                                                                                                                |
| `duvo skills files <skill-id> [--json]`                                                                                                                                       | List files inside a skill.                                                                                                            |
| `duvo skills file-get <skill-id> <path> [--json]`                                                                                                                             | Print the content of a file inside a skill.                                                                                           |
| `duvo skills file-update <skill-id> <path> [--content <text>\|--content-file <path>] [--json]`                                                                                | Update the content of a file inside a skill.                                                                                          |
| `duvo skills download <skill-id> [--output <path>]`                                                                                                                           | Download a custom skill as a ZIP.                                                                                                     |
| `duvo skills assignments <skill-id> [--json]`                                                                                                                                 | List agents whose live build references a skill.                                                                                      |
| `duvo skills generate --prompt <text> [--existing-content-file <path\|->] [--json]`                                                                                           | Generate a `SKILL.md` from a prompt (does not persist). `--existing-content-file` refines existing content instead of starting fresh. |
| `duvo plugins list [--json]`                                                                                                                                                  | List plugins referenceable by name in a revision.                                                                                     |

## Team

| Command                                                                                                               | Purpose                                                                                                                                                                                                          |
| --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo team current [--json]`                                                                                          | Get the team scoped to your credentials (shorthand for the common case).                                                                                                                                         |
| `duvo team get [--team <id>] [--json]`                                                                                | Get a team by ID (defaults to the team scoped to your credentials).                                                                                                                                              |
| `duvo team members [--team <id>] [--limit <n>] [--offset <n>] [--json]`                                               | List members of a team.                                                                                                                                                                                          |
| `duvo team use <team-id>`                                                                                             | Set the default team for the active OAuth profile. Subsequent commands resolve to this team unless overridden by `--team` or `DUVO_TEAM_ID`. API-key profiles reject this (the key is already scoped to a team). |
| `duvo teams list [--json]`                                                                                            | List every team your credentials can act on. For API-key callers this returns one team; for OAuth callers it returns all teams you're a member of. Useful before `duvo team use`.                                |
| `duvo teams org <org-id> [--json]`                                                                                    | List every team inside an organization. Requires an OAuth profile with an org-level role; API-key profiles are rejected (the key is team-scoped and has no org visibility).                                      |
| `duvo teams org-insights <org-id> [--start-date <date>] [--end-date <date>] [--json]`                                 | Org-level usage insights over a window (default: last 30 days). Same org-role requirement as `teams org`.                                                                                                        |
| `duvo teams org-metrics <org-id> [--start-date <date>] [--end-date <date>] [--json]`                                  | Org-level metrics over a window (default: last 7 days). Same org-role requirement as `teams org`.                                                                                                                |
| `duvo teams org-usage <org-id> [--start-date <date>] [--end-date <date>] [--granularity <day\|week\|month>] [--json]` | Org-level usage bucketed by `--granularity` (default `day`) over a window (default: last 30 days, capped at 365 days). Same org-role requirement as `teams org`.                                                 |

## Secrets (env-var secrets)

`duvo secrets …` manages team-scoped env-var secrets that are injected into runs at runtime.
Secrets hold one or more `KEY=VALUE` pairs; the values are encrypted at rest and are never
returned by the API. Attach secrets to specific revisions with `revision-secrets attach`.

| Command                                                                                                                                                                         | Purpose                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `duvo secrets list [--json]`                                                                                                                                                    | List secrets visible to you — team-shared plus your personal entries.                                              |
| `duvo secrets get <id> [--json]`                                                                                                                                                | Get a secret by ID. Returns metadata and env-var key names; values are never exposed.                              |
| `duvo secrets create --name <n> --value KEY=VALUE [--value KEY=VALUE …] [--service-slug <slug>] [--shared] [--json]`                                                            | Create a secret. `--value` is repeatable. `--shared` creates a team-shared secret (requires manager role).         |
| `duvo secrets update <id> [--name <n>] [--service-slug <slug>\|--clear-service-slug] [--shared\|--personal] [--set KEY=VALUE …] [--remove KEY …] [--rename OLD=NEW …] [--json]` | Update metadata or patch env vars. `--set` adds/overwrites; `--remove` deletes; `--rename` renames a key.          |
| `duvo secrets delete <id> [-y] [--json]`                                                                                                                                        | Soft-delete a secret. Deleting a team-shared secret requires manager role. Destructive — prompts for confirmation. |

## Credentials (browser logins)

`duvo credentials …` manages browser logins (username + password and/or TOTP secret) that the
browsing agent can use during runs. Passwords and OTP secrets are encrypted at rest and are
never returned by the API. Attach logins to specific revisions with `revision-logins attach`.

| Command                                                                                                                            | Purpose                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `duvo credentials list [--domain <domain>] [--json]`                                                                               | List browser logins visible to you — team-shared plus your personal logins. Passwords and OTP secrets are never returned.          |
| `duvo credentials get <id> [--json]`                                                                                               | Get a browser login by ID (passwords/OTP values are never exposed).                                                                |
| `duvo credentials create --domain <domain> [--username <u>] [--password <p>] [--otp-secret <s>] [--shared] [--json]`               | Create a browser login. At least one of `--password` or `--otp-secret` is required. `--shared` creates team-shared (manager role). |
| `duvo credentials update <id> [--domain <d>] [--username <u>] [--password <p>] [--otp-secret <s>] [--shared\|--personal] [--json]` | Update a browser login. Sharing toggles require manager role.                                                                      |
| `duvo credentials delete <id> [-y] [--json]`                                                                                       | Delete a browser login. Destructive — prompts for confirmation.                                                                    |

## Revision secrets & revision logins

These commands attach/detach team secrets and browser logins to a specific revision so the runs
spawned from it get the injected values at runtime.

### Revision secrets

| Command                                                                               | Purpose                                                                  |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `duvo revision-secrets list --agent <id> --revision <id> [--json]`                    | List env-var secrets attached to a revision.                             |
| `duvo revision-secrets attach --agent <id> --revision <id> --secret <id> [--json]`    | Attach a secret to a revision (its env vars will be injected into runs). |
| `duvo revision-secrets detach <secret-id> --agent <id> --revision <id> [-y] [--json]` | Detach a secret from a revision. Destructive — prompts for confirmation. |

### Revision logins

| Command                                                                                  | Purpose                                                                         |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `duvo revision-logins list --agent <id> --revision <id> [--json]`                        | List browser logins attached to a revision.                                     |
| `duvo revision-logins attach --agent <id> --revision <id> --credential <id> [--json]`    | Attach a browser login to a revision so the browsing agent can use it.          |
| `duvo revision-logins detach <credential-id> --agent <id> --revision <id> [-y] [--json]` | Detach a browser login from a revision. Destructive — prompts for confirmation. |

## Suggestions

`duvo suggestions …` manages connection suggestions on an Agent. The platform
automatically surfaces suggestions when it detects a tool call that failed
because a Connection was missing or not yet linked. Consuming a suggestion
stages the change into the Agent's draft revision; rejecting it dismisses it.

| Command                                                                                                | Purpose                                                                                                                                           |
| ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo suggestions list <agent-id> [--status <pending\|history>] [--limit <n>] [--offset <n>] [--json]` | List suggestions for an Agent. `--status pending` (default) is the active inbox; `--status history` shows consumed/dismissed/auto-cleared items.  |
| `duvo suggestions consume <suggestion-id> [--revision-id <id>] [--json]`                               | Apply a suggestion: stage its change into the Agent's draft revision. Pass `--revision-id` to target a specific draft rather than the newest one. |
| `duvo suggestions reject <suggestion-id> [-y] [--json]`                                                | Dismiss a pending suggestion. Destructive — prompts for confirmation.                                                                             |

## Bundled guides

`duvo guide …` reads version-matched CLI guides shipped alongside the installed
binary. Guides are Markdown files with frontmatter; they do not make API calls.
The main guide (`duvo`) is the recommended starting point for AI agents
unfamiliar with the CLI.

| Command                          | Purpose                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `duvo guide list [--json]`       | List the bundled guides shipped with this CLI version (also runs as `duvo guide`).                   |
| `duvo guide get <name> [--json]` | Print a guide by name. `--json` returns `{ name, description, body }`.                               |
| `duvo guide path [name]`         | Print the on-disk path to the bundled guides directory, or to one specific guide if `name` is given. |

## Low-level

| Command                                                                                   | Purpose                                                                                           |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `duvo api <METHOD> <path> [-f key=value …] [-F key=value …] [--input <path\|->] [--json]` | Make an authenticated request to any endpoint. See `api-escape-hatch.md` for the full flag table. |
