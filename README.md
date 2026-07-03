# ui-live-editor

Internal service: chat on the left, a live data dashboard on the right.
The user injects one dataset (Parquet via DuckDB, or Postgres); a dashboard agent
builds and iteratively edits an interactive dashboard rendered in a sandboxed
iframe, inspectable through **Preview / Query / Code** tabs.

This repo currently contains the two specification artifacts that define the
agent's behavior. The English versions are the production documents; the Korean
versions are faithful translations for readability.

## Documents

| Document | Purpose |
|---|---|
| [`prompts/system-prompt.en.md`](prompts/system-prompt.en.md) | Production system prompt template (EN) |
| [`prompts/system-prompt.ko.md`](prompts/system-prompt.ko.md) | 시스템 프롬프트 한글 대역본 (이해용) |
| [`harness/HARNESS.en.md`](harness/HARNESS.en.md) | Harness specification (EN): agent loop, tool contracts, validation gates, sandbox, data bridge, LiteLLM integration |
| [`harness/HARNESS.ko.md`](harness/HARNESS.ko.md) | 하네스 명세 한글 대역본 (이해용) |

## Design at a glance

- **One SQL dialect**: DuckDB is the only query engine; Postgres sources are
  attached read-only, so the agent (and prompt) deal with a single dialect.
- **Multi-file virtual project**: the agent manages `index.html` + `main.jsx` +
  `components/` etc. through `write_file`/`read_file` tools — full-file writes,
  no patches.
- **Registered-query data bridge**: dashboard code never contains SQL and has no
  network (`connect-src 'none'`). Data flows only through parameterized queries
  the agent registers, invoked from the iframe via `window.dataBridge.query()`
  over postMessage — which is what makes filters and drilldowns work safely.
- **Validate, then publish**: per-call gates (SQL AST allowlist, esbuild syntax,
  import allowlist) plus a post-turn build/smoke-render gate with an automatic
  repair loop; only green revisions reach the Preview tab.
- **Thin custom agent loop over LiteLLM**: no general-purpose coding agent —
  the tool surface is intentionally small and domain-specific, with a JSON-action
  fallback mode for models with weak native tool calling.
