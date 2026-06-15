# Workflows

End-to-end recipes for the most common things people do with `duvo`.
Each workflow is written as a shell session a user can paste into a
terminal (or adapt for CI), with `jq` used to pull IDs from JSON
output. Everything assumes `duvo login` has already stored a profile.

## 1. First-time setup

```bash
# 1. Install
npm install -g @duvoai/cli
duvo --version

# 2. Sign in via OAuth (opens browser). For headless contexts, use --api-key.
duvo login --name acme
# or, with an API key from https://app.duvo.ai/settings/api-keys:
# duvo login --name acme --api-key dv_live_...

# 3. Verify
duvo whoami
# Profile   acme
# User      Jane Doe <jane@acme.com>
# Auth type OAuth
```

## 2. Create an agent and start a run

```bash
# Create the agent non-interactively. --no-build skips creating an initial
# build (omit it to get a default starter build).
agent_id=$(
  duvo agents create \
    --name "Customer onboarding" \
    --input "Process new customer onboarding requests from the inbox." \
    --json \
  | jq -r .agent.id
)
echo "Agent: $agent_id"
```

**Option 1 — stream inline:** start the run and block until it finishes.

```bash
duvo runs start --agent "$agent_id" --message "Process today's queue." --follow
```

**Option 2 — capture ID, stream separately:** useful when you need the run ID for a later step.

```bash
run_id=$(
  duvo runs start --agent "$agent_id" --message "Process today's queue." --json \
  | jq -r .run.id
)
duvo runs messages "$run_id" --follow
```

**Option 3 — process NDJSON programmatically:** `--follow --json` emits each message as a JSON object on its own line.

```bash
duvo runs start --agent "$agent_id" --message "Process today's queue." --follow --json \
  | while IFS= read -r line; do
      echo "$line" | jq -r '.text_content // empty'
    done
```

For event-driven workflows, pass `--webhook-url` to `runs start` and
receive events asynchronously rather than polling or following.

## 3. Respond to a human-in-the-loop request

Some agents pause for human approval mid-run. When that happens:

```bash
# `request-id` is optional — if omitted, the CLI targets the run's
# pending request automatically.
duvo runs respond "$run_id" --approve
duvo runs respond "$run_id" --deny
duvo runs respond "$run_id" --answer "approval=yes" --answer "reason=Looks fine"
```

`--answer` is repeatable for multi-question request types.

## 4. Work with a queue

Create a queue, wire up the agents that produce and consume its cases,
then drop cases in and let the consumer work them.

### Queue roles: producer vs consumer

A queue connects two agent **roles**, and each role is an
_integration linked to the queue_ — not just a trigger:

- **Producer** — an agent with the `case-queue-producer` integration
  linked to the queue. It **creates** cases when it runs.
- **Consumer** — an agent with the `case-queue-consumer` integration
  linked to the queue **and** an enabled **case trigger**. New cases
  fire the consumer, which claims and resolves them.

⚠️ **The trap:** a case trigger _without_ the consumer integration
linked still fires the agent, but the run can't claim or resolve the
case — so the run fails and the case ends up `failed` (after retries),
and it keeps failing for **every** new case until you link the
integration. The trigger also gets **auto-disabled the next time you
promote a revision** whose live consumer slot doesn't consume the
queue (`"Case trigger disabled: queue no longer consumed by live
build"`), which can mask the root cause. The integration and the
trigger must go together — `duvo queues link-agent --role consumer`
does both in one step, which is exactly why it exists.

The `case-queue-producer` / `case-queue-consumer` integrations are
seeded per team (provider `default`) and appear in `duvo integrations
list`.

### Wire up producer and consumer agents

```bash
# 1. Create the queue.
queue=$(duvo queues create --name "Inbound invoices" --json | jq -r .queue.id)

# 2. Producer: attaches case-queue-producer on the agent's live revision
#    and maps the queue. The producer creates cases when it runs.
duvo queues link-agent "$queue" --agent <producer-agent-id> --role producer

# 3. Consumer: attaches case-queue-consumer, maps the queue, AND creates the
#    case trigger. --enable-trigger starts it picking up cases immediately.
duvo queues link-agent "$queue" --agent <consumer-agent-id> \
  --role consumer --enable-trigger --concurrency 3

# 4. Verify both ends are wired (watch the PROBLEMS column on either side).
duvo queues agents "$queue"

# Tear a binding back down — removes the queue from the role's slot; for
# consumers it also removes the case trigger for that queue.
duvo queues unlink-agent "$queue" --agent <consumer-agent-id> --role consumer
```

`link-agent` operates on the agent's **live revision** by default; pass
`--revision <id>` to target a specific one. Without `--enable-trigger`
the consumer's trigger is created **disabled** and won't pick up cases
until enabled (`duvo agents case-triggers update`). To inspect or
re-map a slot's linked queues directly, use `duvo revision-integrations
queues list|set`.

⚠️ **The other trap — attached but unmapped.** If you attach
`case-queue-producer`/`case-queue-consumer` with
`revision-integrations attach` but never map a queue to the slot, the
integration looks present yet points at **no queue**. The run fails at
runtime and nothing tells you the slot was empty. `link-agent` avoids
this by attaching **and** mapping in one step; if you wire slots by hand
with `attach`, always follow up with `revision-integrations queues set`.

### Confirm the wiring (self-check)

Before starting work, confirm every case-queue slot on the revision
actually points at a queue:

```bash
duvo revision-integrations case-queue-setup \
  --agent <agent-id> --revision <revision-id>
```

It returns each case-queue slot's `linked_queue_count` — **a slot
reporting `0` is attached but maps to no queue and will fail at
runtime.** Map a queue with `revision-integrations queues set` and
re-check until every slot reports a non-zero count.
`duvo queues agents "$queue"` surfaces the same problem from the queue
side in its PROBLEMS column.

`case-queue-setup` checks **queue slots only** — it says nothing about
the agent's other integrations. If the revision also uses OAuth or
user-provided integrations (HubSpot, Gong, Slack, …), verify those
separately: every such slot needs a pinned connection (see the
checklist below and workflow 7).

### Setup checklist — run before declaring the agent ready

A case-queue agent is **not** set up until every line below passes.
Don't report success on attach/create exit codes alone — the failure
modes here are all silent at setup time and only surface as failed
runs later.

```bash
agent=<agent-id>
# The live revision is the one runs actually use — verify that one,
# not whatever draft you happen to have edited last.
revision=$(
  duvo revisions list --agent "$agent" --json \
  | jq -r '.builds[] | select(.status == "live") | .id'
)

# 1. Every case-queue slot points at a queue (no linked_queue_count: 0).
duvo revision-integrations case-queue-setup --agent "$agent" --revision "$revision"

# 2. Every OAuth / user-provided slot has ≥ 1 pinned connection.
#    List the slots, then check each non-default slot's connections.
duvo revision-integrations list --agent "$agent" --revision "$revision" --json
duvo revision-integrations connections list \
  --agent "$agent" --revision "$revision" --integration <slot-id> --json
# → an empty connections array means the slot will fail at runtime: pin
#   one with `duvo revision-integrations connections pin` (workflow 7).
#   If the user has no connection of that type at all
#   (`duvo connections list --type <slug>` is empty), stop and tell them
#   to connect the account first — you cannot pin what doesn't exist.

# 3. Consumer agents: the case trigger exists and is enabled.
duvo agents case-triggers list "$agent"

# 4. Both ends look right from the queue side (PROBLEMS column empty).
duvo queues agents <queue-id>

# 5. Smoke test: drop one test case and watch the consumer pick it up.
#    Delete the test case afterwards (duvo cases delete <case-id> -y).
duvo cases create --queue <queue-id> --title "setup smoke test"
duvo runs list --agent "$agent" --limit 1
```

### Add and inspect cases

```bash
# Add a single case.
duvo cases create --queue "$queue" \
  --title "INV-1042" \
  --data "Vendor: Acme; amount: 1240.00 USD; due: 2026-06-01"

# Add many cases from a JSON file (or stdin).
cat > cases.json <<'JSON'
[
  { "title": "INV-1043", "data": "Vendor: BetaCo; amount: 880.00 EUR" },
  { "title": "INV-1044", "data": "Vendor: GammaInc; amount: 2300.00 USD",
    "labels": [{ "key": "priority", "value": "high" }] }
]
JSON
duvo cases create --queue "$queue" --from-file cases.json

# Reading from stdin instead:
echo '[{"title":"INV-1045"}]' | duvo cases create --queue "$queue" --from-file -

# Inspect cases.
duvo cases list --queue "$queue" --status pending,processing
duvo cases get <case-id>

# Re-process a batch on a consumer agent by hand (the trigger does this
# automatically once the consumer is wired up above).
duvo cases bulk-reprocess --queue "$queue" \
  --agent <consumer-agent-id> \
  --ids "<id1>,<id2>,<id3>"

# Manage a label vocabulary on the queue.
duvo queue-labels create --queue "$queue" --key priority --value high
duvo cases labels assign <case-id> --queue "$queue" --label "priority=urgent"

# Unassign a label by key=value (symmetric with assign) or by ID.
duvo cases labels unlink <case-id> --queue "$queue" --label "priority=urgent"
```

`bulk-*` operations accept 1-100 IDs at a time, comma-separated, via
`--ids`.

## 5. Sandboxed runs with uploaded files

When an agent needs files staged in advance (e.g. a PDF to summarize,
a CSV to transform), use a **sandbox**:

```bash
# 1. Create a sandbox.
sandbox=$(duvo sandboxes create --json | jq -r .sandbox_id)

# 2. Upload up to 10 MB per file via the small-file path.
duvo sandboxes upload "$sandbox" ./report.pdf
duvo sandboxes upload "$sandbox" ./customers.csv

# 3. For larger files, get a presigned URL and PUT the bytes yourself.
#    Inspect the response shape with --json first; pull the upload URL
#    from whichever field your server version uses.
duvo sandboxes prepare-upload-url "$sandbox" --path big-file.bin --json

# 4. List what's staged.
duvo sandboxes files "$sandbox"

# 5. Start a run that consumes the sandbox.
duvo runs start --agent "$agent_id" --sandbox-id "$sandbox"
```

A sandbox is per-run staging — don't confuse it with `duvo files …`,
which manages persistent team-wide files visible in the Duvo Files
surface.

## 6. Authorize a custom MCP server

Two paths, depending on whether the MCP server supports OAuth Dynamic
Client Registration (DCR):

```bash
# Probe what tools the server exposes (no auth needed):
duvo connections probe --url https://mcp.example.com --header "X-Key=demo"

# (a) DCR-capable MCP server — let Duvo register itself.
duvo oauth mcp check --url https://mcp.example.com
duvo oauth mcp authorize \
  --url https://mcp.example.com \
  --name "Example MCP" \
  --integration-type custom_mcp
# Follow the printed authorization URL to grant access.

# (b) Non-OAuth MCP server (or you already have an API key) —
#     store the credentials yourself.
duvo connections create \
  --name "Example MCP" \
  --auth-method api_key \
  --server-url https://mcp.example.com \
  --secret-header "Authorization=Bearer dv_xxx" \
  --header "X-Tenant=acme"
```

`--secret-header` values are encrypted at rest; `--header` values are
stored as plaintext. Both flags are repeatable.

## 7. Promote a new revision

```bash
# 1. Create a revision from a config file (or stdin).
revision=$(
  duvo revisions create \
    --agent "$agent_id" \
    --name "v2 — added Slack notifications" \
    --config-file ./agent-config.json \
    --json \
  | jq -r .build.id
)

# 2. Attach integrations to the revision. This only creates the SLOTS —
#    OAuth and user-provided integrations are not usable yet.
duvo revision-integrations attach \
  --agent "$agent_id" --revision "$revision" \
  --integration slack --integration google_sheets

# 3. Pin one of the user's connections to every OAuth / user-provided
#    slot. First find the slot IDs on the revision…
duvo revision-integrations list \
  --agent "$agent_id" --revision "$revision" --json

#    …then find the user's connection for each integration type…
duvo connections list --type slack --json

#    …and pin it to the slot.
duvo revision-integrations connections pin <connection-id> \
  --agent "$agent_id" --revision "$revision" --integration <slot-id>

# 4. Verify: every OAuth / user-provided slot reports ≥ 1 pinned
#    connection. An empty list here means runs will fail at runtime.
duvo revision-integrations connections list \
  --agent "$agent_id" --revision "$revision" --integration <slot-id>

# 5. Promote the revision to live.
duvo revisions promote "$revision" \
  --agent "$agent_id" \
  --description "Roll out Slack notifications"
```

⚠️ **The third trap — attached but unpinned.** Attaching an OAuth or
user-provided integration (HubSpot, Gong, Slack, custom MCP, …) makes
it _look_ configured: it shows up in `revision-integrations list` and
as a chip in the web UI. But the run resolves tools through the
**pinned connection**, and without one the slot silently resolves to
nothing — the run fails with "not connected" and no setup-time command
ever warned you. Always follow `attach` with `connections pin`, and
verify with `connections list` per slot before telling the user the
agent is ready. If `duvo connections list --type <slug>` shows the
user has no connection of that type, pinning is impossible — they must
connect the account first (`duvo oauth …` or the web UI's Connections
page). Default integrations (browser, Exa, human-in-the-loop,
`case-queue-producer`/`case-queue-consumer`) are the exception: they
need no pin, but case-queue slots need a queue mapped instead (see
workflow 4).

Promotion is a switch to the live build — existing runs continue on
the revision they started with.

## 8. Manage team files

```bash
# Get a presigned URL for a new team file. Both --file-name and
# --content-type are required.
upload=$(duvo files upload-url \
  --file-name policy.md --content-type text/markdown --json)
signed_url=$(echo "$upload" | jq -r .signedUrl)
file_path=$(echo "$upload" | jq -r .filePath)

# PUT the bytes at the signed URL.
curl -X PUT \
  -H "Content-Type: text/markdown" \
  --data-binary @./policy.md "$signed_url"

# Read it back.
duvo files list
duvo files content "$file_path"

# Edit it in place.
duvo files content-set "$file_path" --content-file ./policy.md

# Rename and finally delete.
duvo files rename --path "$file_path" --new-path policies/policy.md
duvo files delete policies/policy.md -y
```

## 9. Work with a Clarity process

Use `duvo clarity` when a user wants terminal access to Clarity
processes, including importing Miro exports or sending interview links
without opening the web UI.

```bash
# 1. Find the process and inspect its current state.
process_id=$(
  duvo clarity search "invoice approval" --json \
  | jq -r '.processes[0].id'
)
duvo clarity overview "$process_id"
duvo clarity versions "$process_id"

# 2. Import a local Miro export into the process.
duvo clarity import-artifact "$process_id" ./miro-process-map.svg

# If the import is meant to fill an open extra-capture request, bind it:
duvo clarity import-artifact "$process_id" ./exception-path.xml \
  --extra-capture-request-id <request-id>

# 3. Create an interview link and send the printed URL to the interviewee.
duvo clarity create-invite-link "$process_id"

# 4. Regenerate documentation after new source material is available.
duvo clarity generate-current-process "$process_id"

# When a new current-process snapshot exists, anchor a transformation
# proposal to it. Pull the snapshot ID from `duvo clarity versions`.
duvo clarity generate-transformation-proposal "$process_id" \
  --current-process-id <current-snapshot-id>

# 5. Promote snapshots after review.
duvo clarity promote-current-process "$process_id" <current-snapshot-id>
duvo clarity promote-transformation-proposal "$process_id" <proposal-snapshot-id>
```

For custom upload clients or artifact-chat workflows without a high-level
CLI command, use `duvo api` against the public route directly. Prefer
`import-artifact` for normal local files because it performs all three
artifact-import steps in one command.

## 10. Use `duvo api` for endpoints without a high-level command

```bash
# Any GET with query params.
duvo api GET /v1/agents -F limit=20 -F offset=0

# A POST with mixed string/typed fields.
duvo api POST /v1/agents \
  -f name="Ops bot" \
  -F build[name]="v1" \
  -F build[enabled]=true

# A POST with a raw JSON body from a file or stdin.
duvo api POST /v1/agents --input ./agent.json
cat ./agent.json | duvo api POST /v1/agents --input -
```

`duvo api` shares the same auth resolution, output pipeline, and exit
codes as the high-level commands — so scripts can mix `duvo agents
create` and `duvo api POST …` freely.

## 11. CI-friendly invocation

A handful of guarantees for scripts and CI pipelines:

- Always pass `--json` and parse with `jq` / `yq`. Table output is for
  humans and not a stable contract.
- Pre-set the profile non-interactively: store a profile during
  initial setup and select it via `--profile <name>` or `DUVO_PROFILE`
  in every invocation. Alternatively, set `DUVO_API_KEY` (and
  `DUVO_API_BASE_URL` if not prod) directly in the environment — no
  config file needed.
- Skip confirmation prompts on destructive commands with `-y` /
  `--yes`. Don't pipe `yes` into the CLI — it explicitly refuses
  inferred consent on a non-TTY stdin.
- Branch on exit codes when retrying makes sense: `1` is a generic
  error, `2` is an auth problem, `3` is a missing resource.
- Deprecation warnings go to **stderr**, so they don't corrupt
  `--json` output captured on stdout.

```yaml
# Example GitHub Actions step
- name: Sync Duvo cases from CRM
  env:
    DUVO_API_KEY: ${{ secrets.DUVO_API_KEY }}
  run: |
    set -euo pipefail
    queue_id="$(duvo queues list --json | jq -r '.queues[] | select(.name=="Onboarding") | .id')"
    ./scripts/build-cases.sh > cases.json
    duvo cases create --queue "$queue_id" --from-file cases.json --json | jq .
```
