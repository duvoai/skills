# Duvo Agent Skills

[![skills.sh](https://skills.sh/b/duvoai/skills)](https://skills.sh/duvoai/skills)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Official [agent skills](https://skills.sh) for [Duvo](https://duvo.ai) — the AI platform that maps your real operations, designs the better version, and runs them across your existing systems.

These skills teach Claude and other agents to operate Duvo directly: manage Agents, debug failed Runs, and write the AOPs that drive them.

There are two ways to install, depending on where you work:

- **[In the Claude app (plugin)](#use-in-the-claude-app-plugin)** — for Claude Desktop / claude.ai users. One install adds the skills *and* the Duvo connector; no terminal needed.
- **[In a coding agent (npx)](#quickstart-coding-agents)** — for Claude Code, Codex, Cursor, and other compatible tools. Installs skills individually or as a set, paired with the `duvo` CLI.

## Use in the Claude app (plugin)

If you build and run your Duvo Agents from the Claude desktop app or claude.ai, install the **Duvo plugin** — it bundles the skills together with the Duvo connector (`https://api.duvo.ai/v2/mcp`), so a single install wires up both the know-how and the data access:

1. Open **Customize → Plugins → "+" → Add marketplace → Add from a repository**.
2. Enter `duvoai/skills`.
3. Install the **Duvo** plugin from the marketplace.
4. The first time a skill reaches for Duvo, you'll be prompted to sign in (OAuth) — no CLI, no API keys.

In Claude Code the equivalent is:

```text
/plugin marketplace add duvoai/skills
/plugin install duvo@duvo
/reload-plugins
```

Notes:

- Plugin-installed skills are namespaced, e.g. `duvo:run-debugger`, `duvo:aop-writer`.
- The plugin ships every skill **except `duvo-cli`** — that one drives the terminal CLI, which doesn't apply inside the Claude app. Use the npx path below if you want it.
- Already added the Duvo connector manually? The plugin brings its own plugin-scoped copy of the same tools — remove the manually-added connector after installing, so you don't end up with two copies of every Duvo tool.
- **Updates:** when a new plugin version is released, the plugin manager shows an **Update** button — one click and you're current.

## Quickstart (coding agents)

Get from zero to a working setup in four steps — install the skills, add the CLI they call, and sign in:

```bash
# 1. Install the skills into your agent
npx skills add duvoai/skills

# 2. Install the Duvo CLI (used by the skills to talk to Duvo)
npm install -g @duvoai/cli

# 3. Authenticate — opens your browser for OAuth sign-in
duvo login

#    ...or sign in with an API key from https://app.duvo.ai/settings/api-keys
#    duvo login --api-key dv_live_...

# 4. Verify you're signed in
duvo whoami
```

Then just ask your agent — for example: *"Why did Run `run_abc123` fail?"*, *"Rewrite this AOP to be clearer"*, or *"List my failed runs from yesterday"*.

### Install a specific skill

```bash
npx skills add duvoai/skills --skill duvo-cli
```

### Install for a specific agent

Use `--agent` to target one or more agents (you can pass several, space-separated):

```bash
# Claude Code
npx skills add duvoai/skills --agent claude-code

# Codex
npx skills add duvoai/skills --agent codex

# Cursor
npx skills add duvoai/skills --agent cursor

# Multiple agents at once
npx skills add duvoai/skills --agent claude-code codex cursor
```

### Update installed skills

Skills installed via npx don't auto-update — run `npx skills update` to pull the latest versions.

## Available skills

Each skill is self-contained — install the whole set or just the one you need. All of them are available via npx; the Claude-app plugin ships all except `duvo-cli`.

| Skill | In plugin | Description |
| --- | :---: | --- |
| **duvo-cli** | — | Drive the Duvo platform from the terminal via the `duvo` CLI (`@duvoai/cli`) — manage agents, runs, cases, queues, files, skills, connections, and Clarity processes, or hit any endpoint with `duvo api`. |
| **aop-writer** | ✓ | Draft, rewrite, or critique a Duvo Agent AOP, returning a single Markdown document in the canonical GOAL / STEPS / NOTES shape. |
| **run-debugger** | ✓ | Investigate why a Duvo Run failed or produced the wrong outcome — reads the Run transcript and active Build, names the root cause from a fixed failure-mode taxonomy, and proposes one concrete fix. |
| **workflow-debugger** | ✓ | Audit a Duvo Agent — or a multi-Agent workflow connected by a Queue — across many Runs to find systemic inefficiencies and quality issues, then recommend concrete AOP and architecture changes. |
| **connection-doctor** | ✓ | Diagnose and health-check the Connections and credentials a Duvo Agent relies on — reads each Build's Connections and their live state, names the problem, and proposes one concrete fix. |
| **improve-agent** | ✓ | Run the guided, end-to-end loop that makes one Duvo Agent better — survey its setup, ground findings in recent Runs via the debugger skills, propose concrete changes, and apply them on your yes. |
| **improve-queue** | ✓ | Run the guided, end-to-end loop that makes one Duvo Queue — and the producer→consumer workflow around it — better, ending in applied AOP, topology, and trigger changes. |

## Learn more

- [Duvo website](https://duvo.ai)
- [Documentation](https://docs.duvo.ai)
- [Duvo MCP server](https://docs.duvo.ai/mcp/duvo-mcp-server) — the connector the plugin bundles
- [Duvo CLI on npm](https://www.npmjs.com/package/@duvoai/cli)

## License

MIT
