# The Duvo code-step SDK

A code step is deterministic Python the platform invokes: case in, patch out.
There is no model at run time. The `duvo` package is how the program reaches
Duvo's tools; it is a client, not an API — which Connection, whose
credentials and what is allowed are all decided behind the relay, so no
credential is ever in the sandbox.

```python
import duvo  # or: from duvo import duvo
```

Both forms reach the same object. The package needs no installation and no
configuration.

## Queue (Consumer) — claim a case, write to it, settle it

```python
case = duvo.claim_case(queue_id=QUEUE_ID)

# Only an unbound run comes back empty. Guard anyway: the same program run on a
# schedule rather than against one case is the shape that hits it.
if case is None:
    raise SystemExit(0)

total = sum(line["amount"] for line in case["data"]["lines"])

duvo.update_case(case_id=case["id"], data={"computed_total": total})
duvo.complete_case(case_id=case["id"], reason=f"Reconciled at {total}")
```

- `duvo.claim_case(queue_id=...)` — take a case to work on. On a case-bound run
  this returns _that_ case, so it cannot be None; on an unbound run it pulls the
  next pending case, and returns None when the queue is empty.
- `duvo.update_case(case_id=..., data={...})` — write to the claimed case.
  Also takes `title` and label arguments.
- `duvo.complete_case(case_id=..., reason=...)` — settle as done.
- `duvo.fail_case(case_id=..., reason=...)` — settle as failed.
- `duvo.postpone_case(case_id=..., postpone_to=..., reason=...)` — defer it.

Exactly one of complete/fail/postpone settles a case. Settle every case the
program claims: a claimed case left unsettled blocks the queue.

## Queue (Producer) — file work into a queue

```python
duvo.add_cases(queue_id=EXCEPTIONS_QUEUE_ID, cases=[
    {"title": f"Unmatched {line['sku']}", "data": line},
])
```

## The four tools that exist on both queues

`list_cases`, `list_labels`, `attach_case_file` and `read_case_file` belong
to both, so they are namespaced rather than flat — say which you mean:

```python
duvo.consumer.list_cases(queue_id=QUEUE_ID)
duvo.producer.list_labels(queue_id=QUEUE_ID)
duvo.consumer.attach_case_file(case_id=case["id"], file_path="report.csv")
duvo.consumer.read_case_file(case_id=case["id"])
```

## Any other Connection, reached by its name

```python
rows = duvo.connections.snowflake.query(
    statement="select po_number from ap.open_pos"
)
duvo.connections.gmail.send(to="ap@example.com", subject="Reconciled", body=summary)

duvo.connections.names       # names this run resolved, sorted
duvo.connections["gmail"]    # for a name the attribute sugar cannot carry
duvo.connection("gmail")     # the same object, for a name held in a variable
duvo.call_tool("gmail", "send", to=..., subject=...)  # any tool, by name
```

Connections live under `duvo.connections`, not on `duvo` itself —
`duvo.gmail` raises `AttributeError`.

Hyphens in a slug become underscores (`google-sheets` →
`duvo.connections.google_sheets`).
Arguments are keywords passed through unchanged. An argument whose name is not
a Python identifier (a hyphen, or a reserved word like `from`) has to go
through `duvo.call_tool(...)`.

Only Connections attached to the step resolve. Every Connection the program
calls must be in the `mcpServers` you return.

## The dispatch input

```python
duvo.execution        # {"run_id", "code_step_id", "build_id", "attempt", "started_at", "timeout_ms"}
duvo.case_ref         # {"case_id", "queue_id"} when case-bound, else None
duvo.trigger          # {"source", "payload"?} when a dispatcher started it, else None
duvo.trigger_payload  # the structured trigger item itself, or None
duvo.policy           # the step's Policy values, resolved, or {}
duvo.files            # [{"name", "path"}] for the team files attached to the step
duvo.file_path(name)  # absolute path of one attached file, by the name it carries
duvo.input            # the whole document; duvo.input.as_dict() for it verbatim
duvo.run_id           # the agent_run this program is executing as
```

`case_ref` is identity, never case content — read the case with
`claim_case`. `trigger_payload` is the messy-to-typed boundary: validate the
item, then file typed cases with `add_cases`. Key any idempotency guard on
`duvo.execution["run_id"]`, which is stable across retries; `attempt` is
advisory and only good for logging.

Files attached to the step are already in the sandbox when the program starts
— the dispatch materialises every one of them or fails the run — so open one
by name and do not guard on it being there:
`open(duvo.file_path("vendors.csv"))`.

## Choosing the next step

A step whose author put it in options mode picks its own successor at run
time, from the set they declared:

```python
duvo.handover_options  # [{"id": "3f2a…", "name": "Approve & Pay"}, …]

if total > THRESHOLD:
    duvo.request_handover(target_agent_id=APPROVAL_STEP_ID)
else:
    duvo.complete_case(case_id=case["id"], reason="Under threshold")
```

Three rules. Do **not** also settle the case — `complete_case`,
`fail_case` and `postpone_case` move it out of claimed status and the target
step never receives it, so on a run that hands over the handover is the
outcome. The handover is recorded now and dispatched after the process exits
cleanly, so keep working normally afterwards and expect a crash to cancel it.
And a run hands over at most once: the same id twice is a no-op, a different
one raises.

A step with no declared options has a single fixed successor or none, and
calling `request_handover` on it raises. If the author marked the set
required, exiting without calling it fails the run.

## Errors

Every failure is a subclass of `duvo.DuvoError`:
`DuvoConfigurationError`, `DuvoRelayError`, `DuvoConnectionNotConnectedError`,
`DuvoInvalidRequestError`, `DuvoRateLimitedError`, `DuvoToolError`,
`DuvoToolNotAllowedError`, `DuvoUnknownConnectionError`,
`DuvoHandoverNotAllowedError`, `DuvoHandoverAlreadyRequestedError`.

Let an unexpected failure raise: a non-zero exit is how the platform records a
failed run, and swallowing it reports success for work that did not happen.
Catch only what the program can genuinely handle, and settle the case as
failed when the case itself is the problem.

`print()` is captured on both pipes and shown on the run's timeline in order
against the tool calls, so it is a real debugging channel.

## What the SDK does not do

No retries (the relay owns that), no case caching, and no tool-name checking —
a wrong tool name is an error from the server that owns the tool, not a local
one. There is no autocomplete or type checking to lean on, so be conservative:
use the tools named above rather than inventing neighbours.
