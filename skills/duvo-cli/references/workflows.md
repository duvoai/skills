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

# Start a run with an initial message.
run_id=$(
  duvo runs start --agent "$agent_id" --message "Process today's queue." --json \
  | jq -r .run.id
)

# Poll the run until it reaches a terminal state.
while :; do
  status=$(duvo runs get "$run_id" --json | jq -r .run.status)
  echo "Run $run_id: $status"
  case "$status" in
    completed|failed|interrupted) break ;;
  esac
  sleep 5
done

# Inspect what happened.
duvo runs messages "$run_id"
```

For event-driven workflows, pass `--webhook-url` to `runs start` and
receive events asynchronously rather than polling.

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

## 4. Work with a case queue

Create a queue, drop cases into it, and delegate them to a consumer
agent.

```bash
# 1. Create the queue.
queue=$(duvo queues create --name "Inbound invoices" --json | jq -r .queue.id)

# 2. Add a single case.
duvo cases create --queue "$queue" \
  --title "INV-1042" \
  --data "Vendor: Acme; amount: 1240.00 USD; due: 2026-06-01"

# 3. Add many cases from a JSON file (or stdin).
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

# 4. Inspect cases.
duvo cases list --queue "$queue" --status pending,processing
duvo cases get <case-id>

# 5. Delegate a batch to a consumer agent.
duvo cases bulk-delegate --queue "$queue" \
  --agent <consumer-agent-id> \
  --ids "<id1>,<id2>,<id3>"

# 6. Manage a label vocabulary on the queue.
duvo queue-labels create --queue "$queue" --key priority --value high
duvo cases labels assign <case-id> --queue "$queue" --label "priority=urgent"
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

# 2. Attach integrations to the revision.
duvo revision-integrations attach \
  --agent "$agent_id" --revision "$revision" \
  --integration slack --integration google_sheets

# 3. Pin specific connections to integration slots.
duvo revision-integrations connections pin <connection-id> \
  --agent "$agent_id" --revision "$revision" --integration <slot-id>

# 4. Promote the revision to live.
duvo revisions promote "$revision" \
  --agent "$agent_id" \
  --description "Roll out Slack notifications"
```

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

## 9. Use `duvo api` for endpoints without a high-level command

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

## 10. CI-friendly invocation

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
