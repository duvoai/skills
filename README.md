# Duvo Agent Skills

[![skills.sh](https://skills.sh/b/duvoai/skills)](https://skills.sh/duvoai/skills)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Official [agent skills](https://skills.sh) for [Duvo](https://duvo.ai) — the AI platform that maps your real operations, designs the better version, and runs them across your existing systems.

These skills teach your coding agent — Claude Code, Codex, Cursor, and other compatible tools — to operate Duvo directly: manage Agents, debug failed Runs, and write the AOPs that drive them, all without leaving your editor or terminal.

## Quickstart

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

Skills don't auto-update — run `npx skills update` to pull the latest versions.

## Available skills

Each skill is self-contained — install the whole set or just the one you need.

| Skill | Description |
| --- | --- |
| **duvo-cli** | Drive the Duvo platform from the terminal via the `duvo` CLI (`@duvoai/cli`) — manage agents, runs, cases, queues, files, skills, connections, and Clarity processes, or hit any endpoint with `duvo api`. |
| **aop-writer** | Draft, rewrite, or critique a Duvo Agent AOP, returning a single markdown document in the canonical GOAL / STEPS / NOTES shape. |
| **run-debugger** | Investigate why a Duvo Run failed or produced the wrong outcome — reads the Run transcript and active Build, names the root cause from a fixed failure-mode taxonomy, and proposes one concrete fix. |
| **workflow-debugger** | Audit a Duvo Agent — or a multi-Agent workflow connected by a Queue — across many Runs to find systemic inefficiencies and quality issues, then recommend concrete AOP and architecture changes. |

## Learn more

- [Duvo website](https://duvo.ai)
- [Documentation](https://docs.duvo.ai)
- [Duvo CLI on npm](https://www.npmjs.com/package/@duvoai/cli)

## License

MIT
