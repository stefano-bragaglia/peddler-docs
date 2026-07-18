# 1. DONE - MCP Server Wiring

## What it does

The `peddler-mcp` console script: wires every feature's tools (credentials, application log, and browser session
tools — the last of which were never registered against any `ToolRegistry` at all) into one shared registry, and
serves them over the existing stdio JSON-RPC `Transport`/`Server` framework. This is the actual MCP server
process — spawned only by Claude Code once registered (story 2), never invoked directly by the user.

## Inputs / outputs

- Reads (optional) environment variables `PEDDLER_CREDENTIALS_PATH` / `PEDDLER_APPLOG_PATH`; falls back to
  `DEFAULT_CREDENTIALS_PATH` / `DEFAULT_APPLOG_PATH` when unset.
- No CLI arguments — `main()` takes no user-facing input; it's invoked by Claude Code with the env vars already
  set (by the `peddler` launcher, story 2).
- Output: none (blocks forever serving stdio requests, same as the existing `Server.serve_forever()`).

## Edge cases and failure modes

- Env vars unset → the corresponding default path is used, matching existing `CredentialStore`/`ApplicationLog`
  behavior.
- Every browser tool (`open_session`, `close_session`, `fill_field`, `fill_credential_field`, `advance_page`)
  currently has its own standalone signature (e.g. `fill_field(field_id, value)`), not the registry's expected
  `arguments: dict -> dict` handler shape — each needs a small adapter closure translating between the two,
  mirroring the pattern already used for the credentials/applog tools' `_make_*` handler factories.

## Acceptance criteria

1. `build_registry()` returns a `ToolRegistry` containing all 9 expected tool names: `read_credentials`,
   `write_credentials`, `record_application`, `list_applications`, `open_session`, `close_session`, `fill_field`,
   `fill_credential_field`, `advance_page`.
2. For each of the 5 browser tools, calling its registered handler with the equivalent `arguments` dict produces
   the same result as calling the underlying function directly (adapter round-trip correctness), verified with
   fakes consistent with the existing browser-tool tests' patterns (fake page / fake session state).
3. `build_registry()` honors `PEDDLER_CREDENTIALS_PATH`/`PEDDLER_APPLOG_PATH` when set (verified via monkeypatched
   env vars + `tmp_path`), and falls back to the existing default paths when unset.
4. `main()` wires a `Transport` and `build_registry()`'s registry into a `Server` and correctly serves at least
   one request/response round-trip — e.g. a `tools/list` request (fed via injectable/fake stdin+stdout in the
   test, not real process streams) returns all 9 registered tools.

## Tasks

1. Create `src/peddler/mcp/main.py` with `build_registry() -> ToolRegistry`: instantiates `CredentialStore` and
   `ApplicationLog` against the env-driven (or default) paths, calls `register_credential_tools`/
   `register_applog_tools`, and registers adapter handlers for the 5 browser tools with appropriate JSON schemas.
2. Add `main()` to the same module: builds a `Transport(sys.stdin, sys.stdout)` and a `Server(transport,
   build_registry())`, then calls `serve_forever()`.
3. Add the `peddler-mcp = "peddler.mcp.main:main"` entry point to `project/pyproject.toml`'s `[project.scripts]`.
4. Write `tests/mcp/test_main.py` covering all four acceptance criteria.

## Deliverables

- `src/peddler/mcp/main.py` — `build_registry()`, `main()`.
- `tests/mcp/test_main.py`
- `project/pyproject.toml` — new `[project.scripts]` entry.

## Dependencies

None at the story level within this feature (first story) — depends on the already-merged frameworks/tools from
features 1–4.
