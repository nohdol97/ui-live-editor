# Harness Specification — Dashboard Agent (EN)

This document specifies the harness that runs the Dashboard Agent: the agent loop,
tool contracts, validation gates, sandbox/serving model, data bridge protocol,
context management, and LiteLLM integration. The companion system prompt lives at
`prompts/system-prompt.en.md`.

## 1. Architecture overview

```
┌─ Chat UI ─┐      ┌──────────── Harness (server) ────────────┐
│ user msgs │────▶ │ Agent loop ──▶ LiteLLM endpoint (LLM)    │
└───────────┘      │    │                                      │
                   │    ├─ tools: get_schema / run_query /     │
                   │    │         register_query / write_file  │
                   │    │         / read_file / delete_file /  │
                   │    │         list_files / list_queries    │
                   │    │                                      │
                   │ Validators (SQL gate, file gate,          │
                   │             post-turn build gate)         │
                   │    │                                      │
                   │ VFS (virtual project files, versioned)    │
                   │ Query registry (parameterized queries)    │
                   │ DuckDB engine (parquet native /           │
                   │                postgres via ATTACH)       │
                   └──────┬───────────────────────┬────────────┘
                          │ serves                │ postMessage
                   ┌──────▼──────┐         ┌──────▼──────┐
                   │ Preview tab │  iframe │  bridge.js  │
                   │ Query tab   │◀───────▶│ dataBridge  │
                   │ Code tab    │         └─────────────┘
                   └─────────────┘
```

One session = one injected dataset + one conversation + one virtual project.

## 2. Data layer

### 2.1 Single query engine: DuckDB

All SQL written by the agent is DuckDB dialect, regardless of the source:

- **Parquet**: registered as a DuckDB view over the file(s):
  `CREATE VIEW <table> AS SELECT * FROM read_parquet('<path>')` — executed by the
  harness at session bind time, never by the agent.
- **Postgres**: attached read-only via the `postgres` extension:
  `ATTACH '<dsn>' AS src (TYPE postgres, READ_ONLY)` — again harness-only. Tables
  are exposed as views in the session catalog so the agent sees one flat namespace.

Read-only is enforced at three layers:
1. SQL gate (AST allowlist — see §5.1),
2. DuckDB connection opened with `access_mode=READ_ONLY` where applicable,
3. For Postgres sources, a DB role with SELECT-only grants.

### 2.2 Schema context builder

At session start (cached; rebuilt only if the dataset changes) the harness builds
the `{{SCHEMA_CONTEXT}}` block injected into the system prompt. Per table:

- table name, row count
- per column: name, type, null fraction
- 3–5 sample values per column (from `USING SAMPLE`, truncated to 60 chars)
- for columns with cardinality ≤ 25: the full distinct value list
- for date/timestamp columns: min/max range

Budget: ≤ 4 KB per table, ≤ 24 KB total; beyond that, tables are listed
name-and-columns-only and the agent is told to explore with `run_query`.

## 3. Agent loop

```python
MAX_TOOL_ITERATIONS = 40   # tool-call rounds per user turn
MAX_REPAIR_ROUNDS   = 3    # post-turn build/render repair cycles

def run_turn(session, user_message):
    session.history.append(User(user_message))

    finished = agent_inner_loop(session)           # ── phase 1: agent works

    for _ in range(MAX_REPAIR_ROUNDS):             # ── phase 2: repair loop
        errors = post_turn_gate(session)           # build + smoke render (§5.3)
        if not errors:
            break
        session.history.append(ToolStyleNotice(
            "PLATFORM VALIDATION FAILED:\n" + format(errors) +
            "\nFix these issues now. Do not reply to the user until resolved."))
        agent_inner_loop(session)
    else:
        notify_user_kept_last_good_revision(session)

    publish_revision(session)                      # ── phase 3: publish (§7)

def agent_inner_loop(session):
    for i in range(MAX_TOOL_ITERATIONS):
        system = render_system_prompt(session)     # fresh SCHEMA/FILE_TREE/QUERIES
        resp = litellm_chat(system, session.history, TOOLS)
        if resp.tool_calls:
            for call in resp.tool_calls:           # sequential execution
                result = dispatch_with_validation(call)
                session.history.append(ToolResult(call.id, truncate(result)))
        else:
            session.history.append(Assistant(resp.text))
            return True
    session.history.append(SystemNudge(
        "Tool budget exhausted. Summarize state and what remains."))
    resp = litellm_chat(render_system_prompt(session), session.history, tools=None)
    session.history.append(Assistant(resp.text))
    return False
```

Key properties:

- The system prompt is re-rendered on **every** model call so `FILE_TREE` and
  `REGISTERED_QUERIES` are always current — the model never needs stale history
  to know project state.
- Validation happens inside `dispatch_with_validation`; a failed tool call
  returns a structured error string to the model (it repairs within the same loop).
- The post-turn gate catches what per-call validation cannot (cross-file import
  breakage, runtime/console errors).

**Turn concurrency and cancellation**: one active turn per session. User messages
arriving mid-turn are queued and delivered as the next user turn (no mid-turn
injection in v1). The user can cancel a running turn: in-flight LLM/tool work is
aborted, the VFS working state is kept, and nothing is published.

## 4. Tool contracts

All tools are exposed via OpenAI-style function calling. Errors are returned as
tool results with an `ERROR:` prefix plus an actionable message — never thrown.

### 4.1 `get_schema`
- **params**: `{}`
- **returns**: the same content as `{{SCHEMA_CONTEXT}}`. If unchanged since the
  last call, returns `"Schema unchanged — see the system prompt Session Context."`
  (saves tokens).

### 4.2 `run_query` — agent-side exploration only
- **params**:
  ```json
  { "sql": {"type": "string"},
    "max_rows": {"type": "integer", "default": 50, "maximum": 200} }
  ```
- **behavior**: SQL gate (§5.1) → execute with 30 s timeout → return column names,
  first `max_rows` rows (CSV-ish compact text), total row count, elapsed ms.
  Result text capped at 4 KB with a `(truncated)` marker.
- **side effect**: appended to the session *exploration log* (Query tab).

### 4.3 `register_query` — the only path data can reach the dashboard
- **params**:
  ```json
  { "id":          {"type": "string", "pattern": "^[a-z][a-z0-9_]{2,40}$"},
    "description": {"type": "string"},
    "sql":         {"type": "string"},
    "params": {"type": "array", "items": {
        "name":     {"type": "string", "pattern": "^[a-z][a-z0-9_]*$"},
        "type":     {"enum": ["string", "number", "boolean", "date", "string[]", "number[]"]},
        "required": {"type": "boolean", "default": false},
        "default":  {}
    }}}
  ```
- **behavior**: SQL gate → placeholder check (every `$name` in SQL must be a
  declared param and vice versa) → dry-run `EXPLAIN` with defaults/type-appropriate
  probe values → store in the registry (upsert by `id`). Returns the result columns
  and types so the agent can wire the UI without guessing.
- **note**: re-registering an existing `id` replaces it (this is the edit path).

### 4.4 `unregister_query`
- **params**: `{ "id": {"type": "string"} }` — removes from registry. Fails with a
  warning listing referencing files if any project file mentions the id (string scan).

### 4.5 `list_queries`
- **params**: `{}` — returns id, description, params, SQL for all registered queries.

### 4.6 `write_file`
- **params**: `{ "path": {"type": "string"}, "content": {"type": "string"} }`
- **behavior**: file gate (§5.2) → store in VFS (full replace). Returns
  `OK <path> (<bytes> bytes)` or `ERROR: ...`.

### 4.7 `read_file`
- **params**: `{ "path": {"type": "string"} }` — returns current VFS content.

### 4.8 `delete_file`
- **params**: `{ "path": {"type": "string"} }` — reserved paths rejected.

### 4.9 `list_files`
- **params**: `{}` — returns the tree with byte sizes and short content hashes.

## 5. Validation gates

### 5.1 SQL gate (applies to `run_query` and `register_query`)

1. Parse with the DuckDB parser (or sqlglot in `duckdb` dialect). Unparseable → error.
2. Exactly **one statement**; must be `SELECT` or `WITH ... SELECT`.
3. Reject on any of: `ATTACH`, `DETACH`, `COPY`, `EXPORT`, `INSTALL`, `LOAD`,
   `PRAGMA`, `SET`, `CALL`, `CREATE`, `INSERT`, `UPDATE`, `DELETE`, `DROP`,
   `ALTER`, `MERGE`.
4. Reject table functions that read paths/URLs (`read_parquet`, `read_csv*`,
   `read_json*`, `glob`, `httpfs` anything) — the session catalog views are the
   only data entry point.
5. Row cap: wrap as `SELECT * FROM (<q>) LIMIT <cap>` unless an inner LIMIT is
   already smaller. Caps: exploration 200, registered/bridge 10 000.
6. Parameters bound exclusively via prepared statements — the harness never
   string-substitutes values.

### 5.2 File gate (applies to `write_file` / `delete_file`)

- **path**: normalized; must be relative, no `..`, depth ≤ 4, extension in
  `.html .js .jsx .css .json .svg .md`.
- **reserved**: `bridge.js`, `vendor/**` — writes rejected.
- **size**: ≤ 200 KB per file, ≤ 40 files per project.
- **syntax**: esbuild parse check (`loader: jsx` for `.js/.jsx`, plus html/css/json
  checks). Errors returned verbatim (file:line:message).
- **imports** (js/jsx): each specifier must be (a) a relative path (`./x`, `../x`)
  ending in `.js` or `.jsx` that stays inside the project after normalization, or
  (b) an exact key of the import map allowlist. Anything else → error listing the
  allowed names. CSS imports from JS are rejected (browser modules cannot load
  CSS; the prompt directs the agent to `<link>` tags).
  **This is a form check only** — whether the target file exists is deliberately
  NOT checked at write time (files may be written in any order); existence is
  verified by the post-turn build gate (§5.3).
- **index.html**: must contain `<script src="/bridge.js">` before the first module
  script — enforced as an error (the platform depends on load order).

### 5.3 Post-turn gate

Runs after the agent's inner loop finishes:

1. **Build check**: esbuild bundle resolution from `index.html`'s module entries —
   catches broken cross-file imports, missing files, syntax that per-file checks
   passed but resolution breaks.
2. **Reference check**: every `dataBridge.query("<id>"` literal in the project must
   match a registered query id.
3. **Smoke render** (recommended, can be phase 2): headless Chromium loads the
   preview for 5 s; fail on any uncaught exception, console.error, or bridge
   protocol error. The test host page implements the real parent side of the
   bridge (same registry, same validators) so bridge calls execute genuinely.
   Collected messages go back to the agent in the repair prompt.

Publish policy: **only a green post-turn gate publishes a new revision** to the
Preview tab. If all repair rounds fail, the previous good revision stays live and
the UI shows "update failed — showing last working version" (the agent is told to
explain this to the user).

## 6. Serving, sandbox, and the data bridge

### 6.1 Serving and publish-time compilation

- **Publishing a revision compiles it.** Every `.jsx` file (and `.js` containing
  JSX) is transpiled with esbuild (`transform` API — per-file, the module graph is
  preserved, no bundling). Compiled output is stored under the **original source
  path** and served with `Content-Type: text/javascript`, so specifiers like
  `./components/Chart.jsx` keep working verbatim and no import-rewriting pass is
  needed. The Code tab always shows the agent's source, never the compiled output.
- VFS served at `GET /artifact/{session_id}/{path}` from the **published revision**
  (not the working copy).
- All `/artifact` and `/vendor` responses carry `Access-Control-Allow-Origin: *`:
  the sandboxed iframe runs with an opaque origin (no `allow-same-origin`), so its
  module fetches arrive with `Origin: null` and would otherwise fail CORS.
- At serve time the harness injects into `index.html`'s `<head>`:
  - the **import map** (bare specifiers → `/vendor/<lib>@<pinned>/...`), and
  - CSP meta (defense in depth alongside the HTTP header).
- Allowed libraries are self-hosted under `/vendor/` as **pre-bundled ESM builds**
  (browsers cannot load CJS — build each lib once with esbuild into a single ESM
  file), pinned versions only. Recommended baseline set: `react`, `react-dom`,
  `echarts`, `date-fns` (adjust to taste; keep the list short — every entry grows
  the prompt and the attack surface).

### 6.2 Sandbox

- `<iframe sandbox="allow-scripts" src="/artifact/{sid}/index.html?rev={n}">`
  — no `allow-same-origin`: the content runs with an opaque origin.
- CSP (HTTP header): `default-src 'none'; script-src 'self'; style-src 'self'
  'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'none'`.
  With `connect-src 'none'`, fetch/XHR/WebSocket are dead — data can only flow
  through postMessage, so library control + data control are enforced by
  infrastructure, not by trusting generated code.

### 6.3 Data bridge protocol

`/bridge.js` (platform-owned, injected requirement in index.html) exposes:

```ts
window.dataBridge.query(queryId: string, params?: object):
  Promise<{ columns: string[]; rows: unknown[][]; rowCount: number; truncated: boolean }>
```

postMessage envelope (child ↔ parent):

```jsonc
// child → parent
{ "type": "ui-live:query", "requestId": "r-17", "queryId": "sales_by_month",
  "params": { "region": "EU" } }

// parent → child (success)
{ "type": "ui-live:result", "requestId": "r-17", "ok": true,
  "columns": ["month", "revenue"], "rows": [["2026-01", 12345.6]],
  "rowCount": 12, "truncated": false }

// parent → child (failure)
{ "type": "ui-live:result", "requestId": "r-17", "ok": false,
  "error": { "code": "PARAM_TYPE", "message": "region must be a string" } }
```

Parent-side enforcement per request:
- messages accepted only when `event.source === iframe.contentWindow`; unknown
  `type` values are ignored (the parent replies with `targetOrigin: "*"` — an
  opaque origin cannot be targeted more precisely, which is why source checking
  matters);
- `queryId` must be registered **in this session**;
- params validated against the declared types (missing required → error;
  undeclared params → error);
- execution via prepared statement, 10 s timeout, result cap 10 000 rows / 5 MB;
- concurrency ≤ 4 in-flight per session — **`bridge.js` queues excess requests
  client-side** (a dashboard with six charts firing on load must degrade to
  sequential loading, not to errors);
- every bridge execution is appended to the Query tab log.

Parameter wire formats (JSON over postMessage): `date` = `"YYYY-MM-DD"` string,
`string[]`/`number[]` = JSON arrays; the parent converts to DuckDB types when
binding.

## 7. Revisions and UI tabs

Each green post-turn gate snapshots `{files, queries}` as revision *n*.

- **Preview**: iframe pointed at the latest green revision (`?rev=` busts cache).
- **Query**: two sections — *Registered queries* (id, description, params, SQL,
  last execution stats) and *Exploration log* (agent's `run_query` history with
  duration/row counts/status).
- **Code**: read-only file tree + viewer of the published revision, with a
  revision picker for diffing/rollback.

Restoring an old revision is a **UI action** (picker → restore copies that
snapshot into the working state and republishes). When the user asks the *agent*
to revert, it simply rebuilds; a `restore_revision` tool is a possible later
addition, not v1.

## 8. Context management

- **Tool-result truncation**: `run_query` ≤ 4 KB; `read_file` full but counted
  against the turn budget; repeated `get_schema` returns a pointer (§4.1).
- **No duplicated project state**: because the system prompt always carries the
  live `FILE_TREE` + `REGISTERED_QUERIES`, after a turn completes the harness
  replaces `write_file` **contents** in older history with
  `"(content written — current version via read_file)"`. Keep the last 2 turns
  verbatim. This keeps long multi-turn sessions cheap without confusing the model.
- **History compaction**: when the conversation exceeds ~60 % of the model
  context, summarize the oldest turns into a single note (what was requested,
  what was built, decisions made) and drop the raw messages.

## 9. LiteLLM integration

- OpenAI-compatible `/chat/completions` with `tools` + `tool_choice: "auto"`,
  `temperature: 0.2`, generous `max_tokens` (file writes are large).
- Retries: 3 with exponential backoff on 5xx/timeout; on hard failure the user
  gets an honest error and session state is preserved.
- Tool calls executed sequentially even if the model emits several in one message.
- **Truncated output** (`finish_reason: "length"` while emitting a tool call): the
  partial call is discarded and the model gets an error result — "output was
  truncated; write smaller files / split this file" — rather than a half-written
  file entering the VFS.
- **Malformed tool-call JSON**: returned to the model as a parse-error tool result
  (with the parser message), never silently dropped.
- **Fallback for weak tool-calling models** (decide per internal model after
  measurement): disable native tools; describe tools in the system prompt; model
  must reply with exactly one fenced JSON action
  `{"tool": "...", "arguments": {...}}` or `{"final": "..."}`; harness parses,
  executes, feeds back. Same loop, same validators — only transport changes.

## 10. Limits (defaults, tune in config)

| Knob | Default |
|---|---|
| Tool iterations / user turn | 40 |
| Repair rounds / turn | 3 |
| Exploration query timeout / row cap | 30 s / 200 rows |
| Bridge query timeout / row cap | 10 s / 10 000 rows, 5 MB |
| Bridge concurrency per session | 4 |
| File size / file count | 200 KB / 40 |
| Tool-result size in history | 4 KB |
| LLM retries | 3 |
| Registered queries per session | 50 |

## 11. Failure modes

| Situation | Behavior |
|---|---|
| Validator rejects same call repeatedly | After 3 identical failures the error message tells the agent to change approach or explain the blocker to the user |
| Post-turn gate red after all repairs | Keep last good revision live; banner in Preview; agent instructed to explain |
| Query timeout | Error suggests aggregating further / adding filters |
| LLM endpoint down | Surface to user; conversation and revision state kept |
| Dataset missing what the user asked for | Agent must say so (prompt rule) — harness adds no magic |
