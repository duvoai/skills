# `duvo api` — Low-level escape hatch

`duvo api <METHOD> <path>` makes an authenticated request to any
endpoint in the Duvo public API. Modelled on `gh api` — same flag
shapes, same parsing rules. Use it when:

- A new endpoint shipped before a dedicated `duvo` command landed.
- You need a body shape the high-level command doesn't expose.
- You're debugging — `duvo api` shows the raw API response with no
  formatting in the way.

High-level commands (`duvo agents create`, etc.) are sugar over this —
they share the same HTTP client, auth resolution, output pipeline, and
exit codes.

## Synopsis

```text
duvo api <METHOD> <path>
  [-f, --field <key=value> ...]
  [-F, --raw-field <key=value> ...]
  [--input <path|->]
  [--json]
```

- `METHOD` is one of `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`
  (case-insensitive — `get` works too).
- `path` is a public API path starting with `/v1/…`. Don't include the
  host; the CLI prepends `apiBaseUrl` from the active profile.

## How fields become the request

- For **GET** and **HEAD**: every `-f` / `-F` field is appended to the
  **query string**.
- For **POST**, **PUT**, **PATCH**, **DELETE**: every `-f` / `-F`
  field is merged into the **JSON body**.
- `--input` overrides both — it sends a raw JSON body verbatim, and is
  rejected on GET/HEAD or when combined with `-f` / `-F`.

When the same key appears in both `-f` and `-F`, the typed (`-F`)
value wins regardless of argument order.

## `-f` vs `-F` — the typing distinction

`-f` always sends the value as a **string**. `-F` parses the value
into a typed JSON value. This is the single most useful disambiguation
the CLI offers; reach for it whenever the wire type matters.

| Flag                       | Input              | Resulting JSON value                  |
| -------------------------- | ------------------ | ------------------------------------- |
| `-f count=42`              | string             | `"42"`                                |
| `-F count=42`              | number             | `42`                                  |
| `-F price=3.14`            | number             | `3.14`                                |
| `-F enabled=true`          | boolean            | `true`                                |
| `-F enabled=false`         | boolean            | `false`                               |
| `-F deletedAt=null`        | null               | `null`                                |
| `-F config=@./config.json` | file (parsed JSON) | the parsed contents                   |
| `-F note=hello`            | non-matching       | `"hello"` _(falls through as string)_ |
| `-f`                       | (always)           | the literal string after the `=`      |

`-F` recognises typed values only for the **exact** tokens above. A
value like `-F status=draft` falls through as the string `"draft"` —
not an error, just a string.

The `@path` form reads the file as JSON, parses it, and inlines the
parsed value at that key. It is the right way to send a nested object
or an array as a single field value:

```bash
# Send build={"name":"v2","enabled":true} as part of the JSON body.
echo '{"name":"v2","enabled":true}' > build.json
duvo api POST /v1/agents -f name="Ops bot" -F build=@build.json
```

## Array fields — `key[]`

A field key ending in `[]` accumulates into a JSON **array** under the
base key. Repeat the flag once per element:

```bash
# Builds {"queue_ids": ["<id-a>", "<id-b>"]}
duvo api PUT /v1/agents/<id>/revisions/<rev>/integrations/<int>/queues \
  -F "queue_ids[]=<id-a>" -F "queue_ids[]=<id-b>"
```

Each element is typed by the same rules as a scalar `-F` (so
`-F "ids[]=1" -F "ids[]=2"` yields `[1, 2]`). A **single, non-`[]`**
field never becomes an array — `-F queue_ids=<id>` sends the bare
string, which a schema expecting an array will reject.

**Don't mix `key` and `key[]` for the same base key.** Combining them
(e.g. `-F tags=x -F "tags[]=y"`) is rejected with a `Conflicting field`
error rather than silently dropping a value — pick one syntax. When in
doubt, or for anything beyond a flat array of scalars, send the whole
body with `--input -`:

```bash
echo '{"queue_ids":["<id-a>","<id-b>"]}' \
  | duvo api PUT /v1/agents/<id>/revisions/<rev>/integrations/<int>/queues --input -
```

## `--input` — raw JSON body

When the request body doesn't decompose cleanly into key/value fields,
hand `duvo api` the whole payload:

```bash
duvo api POST /v1/agents --input ./agent.json
cat ./agent.json | duvo api POST /v1/agents --input -
```

`-` means stdin. `--input` is only valid for `POST` / `PUT` / `PATCH`
/ `DELETE` — `GET`/`HEAD` requests with `--input` are rejected.

`--input` cannot be combined with `-f` / `-F`. If you need to mix a
fixed JSON skeleton with a few computed fields, render the full JSON
yourself (e.g. with `jq`) and pipe it via `--input -`.

## Worked examples

```bash
# 1. GET with query params.
duvo api GET /v1/agents -F limit=20 -F offset=0
duvo api GET /v1/runs -f status=running -F limit=50

# 2. POST a flat body with mixed string + typed fields.
duvo api POST /v1/agents \
  -f name="Ops bot" \
  -F build[name]="v1" \
  -F build[enabled]=true

# 3. POST a nested body by loading JSON from a file.
duvo api POST /v1/agents \
  -f name="Ops bot" \
  -F build=@build.json \
  -F integrations=@integrations.json

# 4. POST a raw body verbatim.
duvo api POST /v1/agents --input body.json

# 5. PATCH from a templated body via stdin.
jq -n --arg name "New name" '{name: $name}' \
  | duvo api PATCH /v1/agents/<id> --input -

# 6. DELETE with a body (rare but supported).
duvo api DELETE /v1/something -F reason=null
```

## Output

`duvo api` returns the raw API response. Without `--json` it pretty-prints
the body for terminal readability; with `--json` it emits the JSON
unmodified for scripting.

```bash
duvo api GET /v1/agents --json | jq '.agents | length'
```

Exit codes match the rest of the CLI:

- `0` — success (2xx response).
- `1` — generic error (network failure, malformed input, 4xx other than `401`/`404`, 5xx).
- `2` — auth error (missing or invalid credentials, `401` from the API, unknown profile).
- `3` — not found (`404` from the API).

## Common pitfalls

- **Forgetting `-F` on numeric IDs.** `duvo api POST /v1/something -f
limit=20` sends `"limit": "20"` (a string), which a strict schema
  may reject. Use `-F limit=20` for numbers.
- **Mixing fields and `--input`.** They're mutually exclusive — pick
  one. If you need both, build the body upstream and use `--input`.
- **Path prefixes.** `apiBaseUrl` is an origin (host + port, no
  path). Always start `path` with `/v1/…`; don't embed it in the base
  URL.
- **Query-string serialization of complex `-F` values.** For
  GET/HEAD, an `-F` value that parses to an object or array is
  serialized as JSON in the query string (so `-F filter=@f.json` on a
  GET produces `?filter={"…":"…"}`). That's fine for endpoints
  expecting JSON query params, but watch for endpoints expecting
  repeated keys instead.
- **`DELETE` requests with bodies.** Supported, but not all upstream
  middleware is happy about them. If a `DELETE` 400s unexpectedly, try
  moving the parameters to the query string with `-f`/`-F` instead of
  `--input`.
