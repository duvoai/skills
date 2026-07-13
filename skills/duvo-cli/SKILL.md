---
name: duvo-cli
description: >
  Drive the Duvo public API from the terminal via the `duvo` CLI
  (`@duvoai/cli`). Use when the user wants to script Duvo — managing
  agents, runs, cases, queues, files, skills, connections, Clarity
  processes, Pulse dashboards, or hitting an arbitrary endpoint via
  `duvo api` — instead of clicking through the Duvo web UI or
  hand-crafting `curl` calls.
license: MIT
metadata:
  author: duvoai
  version: "1.4.0"
  website: https://duvo.ai
  docs: https://docs.duvo.ai
---

# Duvo CLI

`duvo` is the official command-line interface for the [Duvo](https://duvo.ai)
public API. It is published to npm as [`@duvoai/cli`](https://www.npmjs.com/package/@duvoai/cli):
a single binary with resource-grouped subcommands (`duvo agents …`,
`duvo runs …`, `duvo cases …`, …) and a low-level `duvo api <method> <path>`
escape hatch for endpoints that don't yet have a dedicated command.

Use the CLI when the user wants to **automate** or **script** Duvo. The
web UI at <https://app.duvo.ai> is the right answer when they want to
click around interactively; `duvo` is the right answer the moment they
ask for a shell snippet, a cron job, a CI step, or "how do I do X
without opening the browser?". Both surfaces talk to the same public
API — there is no functional gap to bridge by reaching for `curl`.

## Install

```bash
npm install -g @duvoai/cli
duvo --version
```

Requires Node.js ≥ 22.22.0. `npx @duvoai/cli <cmd>` also works without
a global install.

## Authentication

`duvo` keeps credentials as **named profiles** on disk. One profile per
(team, environment). The first profile added becomes the default; from
then on, every command uses the default unless overridden.

```bash
duvo login                       # OAuth flow — opens your browser
duvo login --name acme           # name the profile up front
duvo login --api-key <key>       # skip OAuth and store an API key directly
duvo whoami                      # confirm who's signed in
```

`duvo login` defaults to **browser-based OAuth** against the Duvo
production tenant. Pass `--api-key <key>` to store a long-lived API key
instead (grab one from <https://app.duvo.ai/settings/api-keys>).

Switching between profiles:

```bash
duvo profiles list               # show all profiles; `▶` marks the default
duvo profiles use                # interactively pick the default profile
duvo profiles use acme           # set a named profile as the default
duvo --profile acme whoami       # one-off override for a single command
```

**Resolution precedence** (the first match wins):

- Profile: `--profile <name>` flag → `DUVO_PROFILE` env → `defaultProfile` in config.
- Credential: `DUVO_API_KEY` env (bypasses the profile's stored credential lookup) → the profile's stored credential.
- Base URL: `DUVO_API_BASE_URL` env → the profile's `apiBaseUrl` → `https://api.duvo.ai`.
- Team: `--team <id>` flag → `DUVO_TEAM_ID` env → the OAuth profile's `defaultTeamId` (set via `duvo team use`) → the team derived from the active credential. API-key profiles are always pinned to the key's team and reject a different `--team` value.

## Reading the help

Every command supports `--help`. Reach for it first instead of guessing:

```bash
duvo --help                      # global help, lists every top-level group
duvo agents --help               # subcommand group
duvo agents create --help        # exact flags for one command
```

The CLI's help text is the source of truth — if a flag doesn't appear
there, it doesn't exist in your installed version. Always show the
user `duvo <command> --help` rather than copy-pasting flags from
memory if you're unsure.

## Output modes

Each command picks a default output shape based on what it returns:

- **Single resource** (`duvo agents get <id>`, `duvo runs get <id>`) →
  human-readable key/value text.
- **Collection** (`duvo agents list`, `duvo runs list`) → table with a
  fixed column set per resource.
- **`--json`** on either of the above → raw JSON to stdout. This is the
  **stable contract** for scripts — table and text formats can change
  between versions, JSON is safe to pipe into `jq` or `yq`.

```bash
duvo agents list --json | jq '.agents[].id'
```

For scripting, always pass `--json` and parse the response. Don't
parse the table output — column widths and truncation thresholds are
not part of the public contract. Envelope keys differ across
endpoints — `agents list` returns `.agents[]`, `queues list` returns
`.queues[]`, `runs list` returns `.data[]`, `agents create` returns
`.agent.id`. Inspect the shape with a one-off `--json` call before
hard-coding a jq path.

## The `duvo api` escape hatch

Any endpoint in the public API is reachable via `duvo api <METHOD>
<path>`, modelled directly on `gh api`. Reach for it when:

- A new public endpoint shipped but doesn't have a dedicated `duvo`
  command yet.
- You need a very narrow query-string or body shape that the
  high-level command doesn't expose.
- You're debugging — `duvo api` shows exactly what the API returned,
  with no formatting in the way.

```bash
duvo api GET /v1/agents -F limit=20 -F offset=0
duvo api POST /v1/agents -f name="Ops bot" -F build[name]="v1"
duvo api POST /v1/agents --input body.json
cat body.json | duvo api POST /v1/agents --input -
```

`-f` sends the value as a **string**; `-F` parses it as a typed value
(boolean, number, JSON `null`, or `@file` to load JSON from disk). See
`references/api-escape-hatch.md` for the full flag table.

High-level commands like `duvo agents create` are sugar over `duvo
api` — they share the same HTTP client, output pipeline, and exit
codes, so a script can mix the two freely.

## Resource groups

See `references/commands.md` for the full command tree with flags.
The top-level groups are:

- **Auth & profiles** — `login`, `logout`, `whoami`, `profiles …`
- **Agents** — `agents …`, `agents delete`, `agents models`, `agents set-model`, `agents case-triggers …`, `agents schedules …`, `agents triggers …`, `agents memory …`, `agents eval-scores`, `agents eval-rubrics …`
- **Suggestions** — `suggestions …` (Connection suggestions: list, consume, reject)
- **Agent folders** — `agent-folders …` (organize agents in a tree)
- **Revisions** — `revisions …`, `revision-integrations …` (versioned configs)
- **Runs** — `runs …` (start, get, message, stop, respond to HITL, evaluation)
- **Queues & cases** — `queues …`, `queue-labels …`, `cases …`
- **Files & sandboxes** — `files …`, `sandboxes …`
- **Connections & integrations** — `integrations …`, `connections …`, `oauth …`
- **Secrets & credentials** — `secrets …` (env-var secrets), `credentials …` (browser logins), `revision-secrets …`, `revision-logins …`
- **Clarity** — `clarity …` (process search, versions, captures, gaps, evidence, facets, export, generation, promotion, artifact imports, invite links, doctor, process landscape, process links, process tags, process summaries)
- **Pulse** — `pulse …` (create, get, list, send message, refresh, stop, rename, share, duplicate, pdf/snapshot export, version history, restore, messages, answer HITL, connections, delete Pulse dashboards)
- **Skills & plugins** — `skills …`, `plugins …`
- **Team** — `team current`, `team get`, `team members`, `team use`, `teams list`, `teams org`, `teams org-insights`, `teams org-metrics`, `teams org-usage`
- **Bundled guides** — `guide …` (version-matched CLI guides for AI agents)
- **Low-level** — `api <method> <path>`

For end-to-end recipes (create an agent → start a run → respond to
HITL; manage a queue and its cases; upload files for a sandboxed run;
authorize a custom MCP server) see `references/workflows.md`.

## Conventions you can rely on across commands

These hold for every command group, so call them out when explaining
the CLI rather than restating per-command:

- `--profile <name>` is a **global** flag that overrides the default
  profile for that single invocation.
- `--team <id>` is a **global** flag that overrides the resolved team
  for that single invocation (OAuth profiles only; API-key profiles
  reject a mismatched team).
- `--json` is available on every command that hits the API and is the
  shape to use in scripts.
- Destructive operations (`duvo logout`, `duvo profiles remove`,
  `duvo runs stop`, `duvo agents delete`, `duvo agent-folders delete`,
  `duvo cases delete`, `duvo cases clear`, `duvo cases bulk-delete`,
  `duvo skills delete`, `duvo queues delete`, `duvo secrets delete`,
  `duvo credentials delete`, `duvo revision-secrets detach`,
  `duvo revision-logins detach`, `duvo agents schedules delete`,
  `duvo agents eval-rubrics remove`, `duvo agents eval-rubrics replace`,
  `duvo connections delete`, `duvo queue-labels delete`,
  `duvo integrations custom delete`, `duvo clarity process-labels delete`,
  `duvo suggestions reject`, `duvo pulse delete`, …)
  prompt for confirmation in a TTY and refuse on a non-TTY stdin. Pass
  `-y` / `--yes` to skip the prompt — never pipe `yes` into the CLI to
  bypass the prompt; it explicitly refuses inferred consent from piped
  input.
- Exit codes: `0` success, `1` generic error, `2` auth error (missing
  or invalid key, unknown profile), `3` not found. Useful for CI
  scripts that want to branch on the failure mode.
- IDs are printed bare (no quotes) so they copy cleanly into shell
  pipelines and `$()` substitutions:
  ```bash
  agent=$(duvo agents create --name "Ops bot" --input "What to do" --json | jq -r .agent.id)
  duvo runs start --agent "$agent"
  ```
- Deprecation warnings print to **stderr** (so they don't corrupt
  `--json` stdout) and can't be silenced. If a user reports an
  unexpected warning, check whether their `@duvoai/cli` version is
  behind.

## Pitfalls and gotchas

- **Profile vs credential.** `--profile foo` changes which profile is
  read; `DUVO_API_KEY=…` overrides the **credential** but still uses
  the selected profile's `apiBaseUrl`. To target a totally different
  environment, set both `DUVO_API_KEY` and `DUVO_API_BASE_URL`.
- **`profiles use` with no argument** opens an interactive picker that
  only works in a TTY. In CI, always pass a name explicitly.
- **`agents create` is interactive by default.** When `--name` and
  `--input` are omitted and stdin is a TTY, the CLI prompts. In CI,
  pass both flags (and `--no-build` if you don't want an initial
  build) so it never blocks waiting for input.
- **`cases create --from-file -`** reads JSON from stdin. The file (or
  stdin) is a single case object or an array of up to 100 cases. Each
  case is `{ "title": "…", "data": "…", "labels": [{ "key": "…", "value": "…" }] }`.
  Labels with no `key` are tag-only labels.
- **Sandbox files vs team files.** `duvo files …` manages persistent
  team files (visible in the Files surface in the UI); `duvo sandboxes …`
  stages files for a single run (`duvo runs start --sandbox-id <id>`). Don't mix them.
- **Connections vs integrations.** `duvo integrations list` shows the
  team's catalog of integration types; `duvo connections list` shows
  the user's actual connected accounts (one per OAuth/credential
  flow). OAuth-based connections (Gmail, Slack, …) are started by
  `duvo oauth native start <provider>` or `duvo oauth composio start …`,
  not by `duvo connections create` — the latter is only for
  user-provided MCP servers.
- **Attached ≠ connected.** `revision-integrations attach` only creates
  the integration **slot** on the revision. For OAuth and user-provided
  integrations (HubSpot, Gong, Slack, custom MCP, …) the run can only
  use the slot once one of the user's connections is **pinned** to it —
  an attached slot with no pinned connection fails at runtime with "not
  connected", and nothing warns you at attach time. Find the connection
  with `duvo connections list --type <slug>`, pin it with
  `duvo revision-integrations connections pin`, and verify with
  `duvo revision-integrations connections list` (expect ≥ 1 entry per
  slot). Default integrations (browser, Exa, human-in-the-loop,
  `case-queue-producer`/`case-queue-consumer`, …) need no pin — but
  case-queue slots need a **queue mapped** instead, verified with
  `duvo revision-integrations case-queue-setup`. See workflow 7 and the
  setup checklist in workflow 4 of `references/workflows.md`.
- **Secrets vs connections vs credentials.** Three separate stores:
  `duvo secrets` holds env-var key/value pairs injected into runs at
  runtime; `duvo credentials` holds browser logins (domain + password
  - optional TOTP) used by the browsing agent; `duvo connections`
    holds OAuth/API-key connections to external services (Gmail, Slack,
    custom MCP, …). They don’t overlap — attach secrets with
    `revision-secrets`, logins with `revision-logins`, and connection
    slots with `revision-integrations connections pin`.
- **Multi-team OAuth profiles.** An OAuth login can belong to several
  teams. Use `duvo teams list` to see all teams, then `duvo team use <id>`
  to set the default, or `--team <id>` per-command. API-key
  profiles are always single-team and reject `--team`.
- **Reconnecting an expired OAuth connection.** Don't create a new
  connection for a stale one — pass `--reconnect-instance-id <id>` to
  `duvo oauth native start` or `duvo oauth mcp authorize` with the
  existing connection's ID (from `duvo connections list`) to
  re-authorize it in place instead of creating a duplicate.
- **Clarity has both read and explicit write commands.** Start with
  `duvo clarity overview <process-id>`, then use `versions`, `current`,
  `proposal`, `compare`, `gaps`, `evidence`, `readiness`, or `facets` to
  understand the process. Use the write commands only when the user
  actually wants to mutate Clarity state: generation, promotion, revert,
  postprocessing, automation build, extra-capture agent, invite
  links, or Miro artifact imports. Those v2-only commands require a
  Clarity v2 process; legacy v1 processes
  support only `overview`, `status`, `captures`, `capture`, and `export`.
  Default output is compact; transcripts and media URLs are included only
  when a JSON command explicitly passes `--include-transcripts`.
- **Clarity artifact imports use a two-phase API under the hood.** Use
  `duvo clarity import-artifact <process-id> <file>` for local Miro SVG,
  XML, PNG, or JPEG exports. It creates the signed URL, uploads bytes, and
  completes the import. For custom upload clients or artifact-chat
  workflows, use `duvo api` against the public route directly.

## When the CLI is the wrong tool

Steer users to a different surface in these cases:

- **They want to read a stream of run events as they happen.** The
  CLI's `runs get` / `runs messages` are point-in-time reads. Use
  `--webhook-url` on `runs start` to receive events asynchronously,
  or open the run in the web UI.
- **They want to edit an AOP or agent configuration interactively.**
  `revisions create` and `revisions update` accept a config file, but
  composing the config by hand is painful. Direct them to the
  Agent editor in the web UI for anything beyond a small targeted
  patch.
- **They want to run an agent in their own process rather than on
  Duvo.** That's not what the public API exposes; the CLI is a client
  to the hosted Duvo platform.

## In-skill references

- `references/commands.md` — full command tree with flags, grouped by
  resource. Use this when you need the exact flag a command expects.
- `references/workflows.md` — end-to-end recipes (create an agent and
  start a run; queue + cases lifecycle; sandboxed runs with uploaded
  files; authorize a custom MCP server).
- `references/api-escape-hatch.md` — `duvo api` flag reference, with
  worked examples of `-f` vs `-F`, `@file` typed fields, and
  `--input` for raw JSON bodies.

When in doubt, run `duvo <command> --help` — the installed binary is
the final source of truth.

## See also

- `aop-writer` — author or rewrite the AOP that ships in a Build (`duvo revisions create` / `duvo revisions update`).
- `run-debugger` — diagnose a failed Run; pairs with `duvo runs get`, `duvo runs messages`, and `duvo revisions get` for API-mode reads.
- `workflow-debugger` — audit an Agent or workflow across many Runs; pairs with `duvo runs list`, `duvo queues agents`, and `duvo revisions get`.

## Resources

- [Duvo](https://duvo.ai) — product website
- [Duvo documentation](https://docs.duvo.ai) — concepts, building Agents, Connections
- [Duvo API reference](https://docs.duvo.ai/api-reference) — every endpoint the CLI wraps
- [`@duvoai/cli` on npm](https://www.npmjs.com/package/@duvoai/cli) — versions, install instructions, changelog
- [Web app](https://app.duvo.ai) — interactive surface for everything the CLI scripts
- [API keys](https://app.duvo.ai/settings/api-keys) — issue and revoke API keys for `duvo login --api-key`
- [Public skill repository](https://github.com/duvoai/skills) — the MIT-licensed community release of this skill, packaged for installation in third-party Claude Code setups
