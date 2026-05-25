# Command Reference

Full command tree for `duvo` (`@duvoai/cli`) as of the most recent
published version. Cross-reference against `duvo <command> --help` from
the installed binary if a flag here doesn't match — the binary is
authoritative.

Global flags available everywhere:

- `--profile <name>` — override the default profile for this invocation.
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

| Command                                                                                                                                                                                                        | Purpose                                                                                                |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `duvo agents list [--limit <n>] [--offset <n>] [--json]`                                                                                                                                                       | List agents on your team. `--limit` is 1-100 (default 20).                                             |
| `duvo agents get <id> [--json]`                                                                                                                                                                                | Show details for one agent.                                                                            |
| `duvo agents create [--name <n>] [--input <text>] [--config <path>] [--no-build] [--json]`                                                                                                                     | Create a new agent. Prompts for missing fields in a TTY. `--no-build` skips creating an initial build. |
| `duvo agents update <id> [--name <n>] [--enable-slack\|--disable-slack] [--enable-microsoft-teams\|--disable-microsoft-teams] [--enable-agentic-memory\|--disable-agentic-memory] [--thread-id <id>] [--json]` | Update display name or delivery settings.                                                              |
| `duvo agents schedules <agent-id> [--json]`                                                                                                                                                                    | List schedules you own for an agent.                                                                   |

### Case triggers (per agent)

| Command                                                                                                                                            | Purpose                                     |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `duvo agents case-triggers list <agent-id> [--json]`                                                                                               | List case triggers configured for an agent. |
| `duvo agents case-triggers get <agent-id> <trigger-id> [--json]`                                                                                   | Get a case trigger by ID.                   |
| `duvo agents case-triggers create <agent-id> --queue <id> [--concurrency <n>] [--disable] [--json]`                                                | Create a case trigger bound to a queue.     |
| `duvo agents case-triggers update <agent-id> <trigger-id> [--queue <id>] [--enable\|--disable] [--concurrency <n>] [--clear-concurrency] [--json]` | Toggle, rebind, or rescale a trigger.       |
| `duvo agents case-triggers delete <agent-id> <trigger-id> [-y] [--json]`                                                                           | Delete a case trigger.                      |

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
| `duvo revisions update <revision-id> --agent <id> [--json]`                         | Update a revision's config (handover targets are derived from the SOP). |
| `duvo revisions promote <revision-id> --agent <id> [--description <text>] [--json]` | Promote a revision to the agent's live build.                           |

### Revision integrations

`duvo revision-integrations …` manages the integration slots on a
revision and the connections pinned to each slot.

| Command                                                                                                                      | Purpose                                                       |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `duvo revision-integrations list --agent <id> --revision <id> [--json]`                                                      | List integrations attached to a revision.                     |
| `duvo revision-integrations attach --agent <id> --revision <id> --integration <slug> […] [--json]`                           | Attach one or more integrations to a revision.                |
| `duvo revision-integrations remove <integration-id> --agent <id> --revision <id> [-y] [--json]`                              | Remove an integration slot from a revision.                   |
| `duvo revision-integrations connections list --agent <id> --revision <id> --integration <id> [--json]`                       | List your pinned connections for a slot.                      |
| `duvo revision-integrations connections pin <connection-id> --agent <id> --revision <id> --integration <id> [--json]`        | Pin one of your connections to a revision's integration slot. |
| `duvo revision-integrations connections unpin <connection-id> --agent <id> --revision <id> --integration <id> [-y] [--json]` | Unpin a connection from a slot.                               |

## Runs

| Command                                                                                                                                                                                  | Purpose                                                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo runs list [--limit <n>] [--offset <n>] [--sort-order <asc\|desc>] [--status <s>] [--agent <id>] [--user <id>] [--case-queue <id>] [--source <source>] [--search <query>] [--json]` | List runs for your team with filters.                                                                                                          |
| `duvo runs get <id> [--json]`                                                                                                                                                            | Get an agent run by ID.                                                                                                                        |
| `duvo runs start --agent <id> [--message <text>] [--sandbox-id <id>] [--webhook-url <url>] [--json]`                                                                                     | Start a new run. `--sandbox-id` pre-stages files; `--webhook-url` streams events.                                                              |
| `duvo runs messages <id> [--limit <n>] [--offset <n>] [--json]`                                                                                                                          | List messages for a run.                                                                                                                       |
| `duvo runs send-message <id> --message <text> [--json]`                                                                                                                                  | Post a follow-up message to an in-progress run.                                                                                                |
| `duvo runs respond <run-id> [request-id] [--approve\|--deny\|--answer <question=answer>] [--json]`                                                                                       | Respond to a human-in-the-loop request. `request-id` defaults to the run's pending request. `--answer` is repeatable for multi-question forms. |
| `duvo runs stop <id> [-y] [--json]`                                                                                                                                                      | Stop a running agent run. Destructive — prompts for confirmation.                                                                              |

## Queues, queue labels, cases

### Case queues

| Command                                                                                                                        | Purpose                                                        |
| ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| `duvo queues list [--limit <n>] [--offset <n>] [--json]`                                                                       | List case queues.                                              |
| `duvo queues get <id> [--json]`                                                                                                | Get a queue by ID.                                             |
| `duvo queues create --name <name> [--description <text>] [--folder-id <id>] [--json]`                                          | Create a queue.                                                |
| `duvo queues update <id> [--name <n>] [--description <text>\|--clear-description] [--folder-id <id>\|--clear-folder] [--json]` | Update a queue.                                                |
| `duvo queues delete <id> [-y] [--json]`                                                                                        | Delete a queue. Interrupts running jobs and removes all cases. |
| `duvo queues agents <queue-id> [--json]`                                                                                       | List producer/consumer agents bound to a queue.                |

### Queue labels (label vocabulary on a queue)

| Command                                                                                                           | Purpose                                                                 |
| ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `duvo queue-labels list --queue <id> [--limit <n>] [--offset <n>] [--json]`                                       | List labels with case counts. `--limit` is 1-1000 (default 1000).       |
| `duvo queue-labels create --queue <id> --value <text> [--key <text>] [--color-hue <0-360>] [--json]`              | Create a label without assigning it. Omit `--key` for a tag-only label. |
| `duvo queue-labels update <label-id> --queue <id> [--key <text>] [--value <text>] [--color-hue <0-360>] [--json]` | Rename a label, change its key, or change its color.                    |
| `duvo queue-labels delete <label-id> --queue <id> [-y] [--json]`                                                  | Delete a label and unlink it from all cases.                            |

### Cases

| Command                                                                                                                                                 | Purpose                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `duvo cases list --queue <id> [--limit <n>] [--offset <n>] [--status <list>] [--search <text>] [--sort-by <field>] [--sort-order <asc\|desc>] [--json]` | List cases in a queue with filters.                                 |
| `duvo cases get <id> [--json]`                                                                                                                          | Get a case by ID with its event history.                            |
| `duvo cases create --queue <id> [--title <text>] [--data <text>] [--label <key=value>] [--from-file <path\|->] [--json]`                                | Create one case (with `--title`) or up to 100 (with `--from-file`). |
| `duvo cases delete <id> [-y] [--json]`                                                                                                                  | Delete a case. Interrupts associated running jobs.                  |
| `duvo cases clear --queue <id> [-y] [--json]`                                                                                                           | Delete every case in a queue.                                       |
| `duvo cases bulk-delete --queue <id> --ids <list> [-y] [--json]`                                                                                        | Delete multiple cases (1-100, comma-separated IDs).                 |
| `duvo cases bulk-delegate --queue <id> --agent <id> --ids <list> [-y] [--json]`                                                                         | Delegate cases to a queue consumer agent.                           |
| `duvo cases bulk-retry --queue <id> --ids <list> [-y] [--json]`                                                                                         | Reset cases to `pending` (interrupts active runs).                  |
| `duvo cases bulk-update-status --queue <id> --ids <list> --status <completed\|failed> [-y] [--json]`                                                    | Bulk-update case statuses.                                          |
| `duvo cases runs <case-id> [--json]`                                                                                                                    | List jobs that have worked on a case.                               |
| `duvo cases runs recent-messages <case-id> <run-id> [--json]`                                                                                           | Recent messages for one run on a case.                              |
| `duvo cases labels list <case-id> --queue <id> [--json]`                                                                                                | List labels assigned to a case.                                     |
| `duvo cases labels assign <case-id> --queue <id> --label <"key=value"\|"value"> [--label …] [--json]`                                                   | Assign one or more labels to a case (repeatable).                   |
| `duvo cases labels unlink <case-id> --queue <id> --label-id <id> [--label-id …] [--json]`                                                               | Remove one or more labels from a case (repeatable).                 |

`cases create --from-file` accepts a single case object or an array of
up to 100. Shape per case:

```json
{
  "title": "Case title",
  "data": "Optional free-form data passed to the assignment on claim",
  "labels": [{ "key": "priority", "value": "high" }, { "value": "urgent" }]
}
```

Labels are `{"key": "…", "value": "…"}` — omit `key` (or set it to
`""`) for a tag-only label. Pass `-` as the path to read from stdin.

## Files & sandboxes

### Team files (persistent)

| Command                                                                            | Purpose                                                                                      |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `duvo files list [--json]`                                                         | List team files.                                                                             |
| `duvo files upload-url --file-name <name> --content-type <mime> [--json]`          | Get a presigned upload URL for a new team file. Response carries `filePath` and `signedUrl`. |
| `duvo files download-url <path> [--json]`                                          | Get a presigned download URL.                                                                |
| `duvo files content <path> [--json]`                                               | Print the content of a text file (binary files rejected).                                    |
| `duvo files content-set <path> [--content <text>\|--content-file <path>] [--json]` | Update the content of a text file.                                                           |
| `duvo files rename --path <old-path> --new-path <new-path> [--json]`               | Rename a team file.                                                                          |
| `duvo files delete <path> [-y] [--json]`                                           | Delete a team file.                                                                          |

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

| Command                                                                                         | Purpose                                                                |
| ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `duvo integrations list [--json]`                                                               | List the team's integration catalog.                                   |
| `duvo integrations custom create --name <n> --auth-method <method> --server-url <url> [--json]` | Create a custom integration type in the team catalog.                  |
| `duvo integrations custom delete <custom-integration-id> [-y] [--json]`                         | Delete a custom integration type. Cascade-removes related connections. |

### Your connections

| Command                                                                                                                                                                                        | Purpose                                                                                                                                                         |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `duvo connections list [--type <type>] [--json]`                                                                                                                                               | List your connections (filter by integration type).                                                                                                             |
| `duvo connections get <id> [--json]`                                                                                                                                                           | Get one of your connections by ID.                                                                                                                              |
| `duvo connections create --name <n> --auth-method <m> [--server-url <url>] [--type <slug>] [--custom-integration-id <uuid>] [--header KEY=VALUE …] [--secret-header KEY=VALUE …] [--json]`     | Create a user-provided connection (custom MCP server). `--header` / `--secret-header` are repeatable. OAuth-based integrations must use `duvo oauth …` instead. |
| `duvo connections update <id> [--name <n>] [--server-url <url>] [--header KEY=VALUE …] [--secret-header KEY=VALUE …] [--clear-headers] [--composio-mcp-id <id>] [--share\|--unshare] [--json]` | Update a connection. `--share` is manager+ only.                                                                                                                |
| `duvo connections delete <id> [-y] [--json]`                                                                                                                                                   | Delete a connection and any triggers bound to it.                                                                                                               |
| `duvo connections credentials <id> [--json]`                                                                                                                                                   | List configured header keys (sensitive values redacted).                                                                                                        |
| `duvo connections probe --url <url> [--header KEY=VALUE …] [--json]`                                                                                                                           | Probe an MCP server URL and list the tools it exposes.                                                                                                          |

### OAuth flows

| Command                                                                                                  | Purpose                                                                                           |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `duvo oauth native start <provider> [--name <n>] [--json]`                                               | Start a native OAuth connection (Gmail, Google Sheets, Outlook, …). Prints the authorization URL. |
| `duvo oauth composio start --auth-config <id> --callback-url <url> [--app <slug>] [--name <n>] [--json]` | Start a Composio-backed OAuth connection. Returns a redirect URL and `connected_account_id`.      |
| `duvo oauth composio connect --app <slug> --name <n> --connected-account-id <id> [--json]`               | Finalize a Composio-backed connection after the start step completes.                             |
| `duvo oauth mcp probe <url> [--header KEY=VALUE …] [--json]`                                             | Probe an MCP server URL and list its tools (same as `duvo connections probe`).                    |
| `duvo oauth mcp check --url <url> [--json]`                                                              | Probe an MCP server for OAuth Dynamic Client Registration support.                                |
| `duvo oauth mcp authorize --url <url> --name <n> [--integration-type <type>] [--json]`                   | Start an OAuth-based connection with a remote MCP server via DCR. Prints the authorization URL.   |

## Skills & plugins

| Command                                                                                                                    | Purpose                                                              |
| -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `duvo skills list [--system] [--json]`                                                                                     | List skills available to your team (or system skills only).          |
| `duvo skills system [--json]`                                                                                              | List global system skills available to all teams.                    |
| `duvo skills create --name <n> --description <d> [--content <text>\|--content-file <path>] [--license <license>] [--json]` | Create a skill from Markdown content.                                |
| `duvo skills upload <file> [--json]`                                                                                       | Create or update a skill by uploading a `SKILL.md` or a ZIP archive. |
| `duvo skills delete <skill-id> [-y] [--json]`                                                                              | Delete a custom skill.                                               |
| `duvo skills files <skill-id> [--json]`                                                                                    | List files inside a skill.                                           |
| `duvo skills file-get <skill-id> <path> [--json]`                                                                          | Print the content of a file inside a skill.                          |
| `duvo skills file-update <skill-id> <path> [--content <text>\|--content-file <path>] [--json]`                             | Update the content of a file inside a skill.                         |
| `duvo skills download <skill-id> [--output <path>]`                                                                        | Download a custom skill as a ZIP.                                    |
| `duvo skills assignments <skill-id> [--json]`                                                                              | List assignments whose live build references a skill.                |
| `duvo skills generate --prompt <text> [--name <n>] [--json]`                                                               | Generate a `SKILL.md` from a prompt.                                 |
| `duvo plugins list [--json]`                                                                                               | List plugins referenceable by name in a revision.                    |

## Team

| Command                                                                 | Purpose                                                         |
| ----------------------------------------------------------------------- | --------------------------------------------------------------- |
| `duvo team get [--team <id>] [--json]`                                  | Get a team by ID (defaults to the team scoped to your API key). |
| `duvo team members [--team <id>] [--limit <n>] [--offset <n>] [--json]` | List members of a team.                                         |

## Low-level

| Command                                                                                   | Purpose                                                                                           |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `duvo api <METHOD> <path> [-f key=value …] [-F key=value …] [--input <path\|->] [--json]` | Make an authenticated request to any endpoint. See `api-escape-hatch.md` for the full flag table. |
