---
name: duvo-cli
description: >
  Drive the Duvo platform from the terminal via the `duvo` CLI (`@duvoai/cli`).
  Use when the user wants to script Duvo — managing agents (Assignments), runs
  (Jobs), case queues, files, connections, or hitting any endpoint via
  `duvo api`. Covers authentication, resource commands, output modes, and
  end-to-end workflows.
license: MIT
metadata:
  author: duvoai
  version: "1.0.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Duvo CLI

`duvo` is the official command-line interface for the [Duvo](https://duvo.ai)
public API. Published to npm as
[`@duvoai/cli`](https://www.npmjs.com/package/@duvoai/cli): a single binary
with resource-grouped subcommands (`duvo agents ...`, `duvo runs ...`,
`duvo cases ...`, ...) and a low-level `duvo api <method> <path>` escape hatch
for endpoints that don't yet have a dedicated command.

Use the CLI when the user wants to **automate** or **script** Duvo. The web UI
at <https://app.duvo.ai> is the right answer for interactive use; `duvo` is the
right answer the moment they ask for a shell snippet, a cron job, a CI step, or
"how do I do X without opening the browser?".

## Install

```bash
npm install -g @duvoai/cli
duvo --version
```

Requires Node.js >= 22. `npx @duvoai/cli <cmd>` also works without a global
install.

## Authentication

```bash
duvo login                       # OAuth flow — opens your browser
duvo login --api-key <key>       # store an API key directly
duvo whoami                      # confirm who's signed in
```

Get an API key from <https://app.duvo.ai/settings/api-keys>.

### Profiles

`duvo` supports named profiles for multiple teams/environments:

```bash
duvo profiles list               # show all profiles
duvo profiles use acme           # set default profile
duvo --profile acme whoami       # one-off override
```

**Resolution precedence** (first match wins):

- Profile: `--profile` flag > `DUVO_PROFILE` env > config default
- Credential: `DUVO_API_KEY` env > profile's stored credential
- Base URL: `DUVO_API_BASE_URL` env > profile's `apiBaseUrl` > `https://api.duvo.ai`

## Output modes

- **Single resource** (`duvo agents get <id>`) — human-readable key/value text
- **Collection** (`duvo agents list`) — table format
- **`--json`** on any command — raw JSON to stdout (the stable contract for scripts)

```bash
duvo agents list --json | jq '.agents[].id'
```

Always use `--json` for scripting. Envelope keys differ by endpoint — inspect
with a one-off `--json` call before hard-coding a jq path.

## Resource groups

- **Auth & profiles** — `login`, `logout`, `whoami`, `profiles ...`
- **Agents** — `agents ...`, `agents case-triggers ...`, `agents schedules`
- **Agent folders** — `agent-folders ...`
- **Revisions** — `revisions ...`, `revision-integrations ...`
- **Runs** — `runs ...` (start, get, message, stop, respond to HITL)
- **Queues & cases** — `queues ...`, `queue-labels ...`, `cases ...`
- **Files & sandboxes** — `files ...`, `sandboxes ...`
- **Connections & integrations** — `integrations ...`, `connections ...`, `oauth ...`
- **Skills & plugins** — `skills ...`, `plugins ...`
- **Team** — `team get`, `team members`
- **Low-level** — `api <method> <path>`

## The `duvo api` escape hatch

Any public API endpoint is reachable via `duvo api <METHOD> <path>`:

```bash
duvo api GET /v1/agents -F limit=20
duvo api POST /v1/agents -f name="Ops bot" -F build[name]="v1"
duvo api POST /v1/agents --input body.json
```

`-f` sends values as strings; `-F` parses as typed values (boolean, number,
JSON null, or `@file` to load JSON from disk).

## Common workflows

### Create an agent and start a run

```bash
agent=$(duvo agents create --name "Invoice checker" --input "Check pending invoices" --json | jq -r .agent.id)
duvo runs start --agent "$agent"
```

### Manage a case queue

```bash
queue=$(duvo queues create --name "Inbox" --json | jq -r .queue.id)
duvo cases create --queue "$queue" --title "Case #1" --data '{"order_id":"A123"}'
duvo cases list --queue "$queue"
```

### Respond to Human-in-the-Loop

```bash
duvo runs respond --run <run_id> --request <request_id> --response "Approved"
```

## Conventions

- `--help` on every command — always check before guessing flags
- Exit codes: `0` success, `1` error, `2` auth error, `3` not found
- Destructive operations prompt for confirmation; pass `-y` to skip
- IDs print bare for easy shell pipeline use

## When the CLI is the wrong tool

- **Streaming run events** — use `--webhook-url` on `runs start` or the web UI
- **Interactive SOP editing** — use the Assignment editor in the web UI
- **Running an agent in your own process** — the CLI is a client to the hosted
  platform, not a local runtime

## Resources

- [Duvo documentation](https://docs.duvo.ai)
- [API reference](https://docs.duvo.ai/api-reference)
- [npm package](https://www.npmjs.com/package/@duvoai/cli)
- [Web app](https://app.duvo.ai)
