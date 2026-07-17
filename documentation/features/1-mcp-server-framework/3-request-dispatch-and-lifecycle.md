# 3. Request Dispatch & Lifecycle

## What it does

Glues the transport (story 1) and the registry (story 1) together into the actual MCP server loop: reads one
JSON-RPC request at a time, handles the `initialize` handshake, `tools/list`, and `tools/call` methods, invokes
the matching registered handler for `tools/call`, and writes back a JSON-RPC response — turning any handler
exception or unknown-tool/unknown-method condition into a structured error result instead of crashing the
process.

## Inputs / Outputs

- `Server(transport: Transport, registry: ToolRegistry)` — constructed with the two prior stories' components.
- `handle_one() -> bool` — reads one request via `transport.read_message()`, processes it, writes a response via
  `transport.write_message()`. Returns `False` when `read_message()` returned `None` (EOF — caller should stop
  looping), `True` otherwise.
- `serve_forever() -> None` — calls `handle_one()` in a loop until it returns `False`.
- Supported request `method` values and their responses:
  - `"initialize"` → result includes a fixed `protocolVersion` and empty/minimal `capabilities`/`serverInfo`
    (exact shape driven by whatever Claude CLI's MCP stdio transport expects at implementation time; the
    version/capabilities strings themselves are a stage-b implementation detail, not re-litigated here).
  - `"tools/list"` → result `{"tools": registry.list_tools()}`.
  - `"tools/call"` with `params: {"name": ..., "arguments": ...}` → looks up the tool in the registry, calls its
    handler with `arguments`, and returns `{"content": [...], "isError": false}` on success or
    `{"content": [{"type": "text", "text": <error message>}], "isError": true}` if the handler raised, or if
    `name` isn't registered.
  - Any other `method` → JSON-RPC error response, `code: -32601` ("Method not found").

## Edge cases and failure modes

- `tools/call` naming an unregistered tool → `isError: true` result (per MCP tool-call convention — this is a
  *tool result* error, not a transport-level JSON-RPC error), message identifying the missing tool name.
- The handler itself raises an exception → caught, turned into an `isError: true` result carrying
  `str(exception)`; the server process must not crash and must continue serving subsequent requests.
- A request missing a `method` key, or with a `params` shape the method doesn't expect (e.g. `tools/call` without
  `name`) → JSON-RPC error response, `code: -32600` ("Invalid Request") or `-32602` ("Invalid params") as
  appropriate, never an uncaught exception.
- A malformed line from the transport (raises `TransportError`) → since there's no request `id` to reply to,
  logged to `stderr` (never `stdout`, which is reserved for protocol messages) and the read loop continues to
  the next line rather than terminating the process.
- Every response (success, tool error, or protocol error) must carry the same `id` as its corresponding request
  (JSON-RPC correlation) — except the `TransportError` case above, where no `id` could be parsed at all.

## Acceptance criteria

1. Sending an `initialize` request yields a response with a `result` key and no `error` key.
2. Registering a tool, then sending `tools/list`, yields `result.tools` containing that tool's `name`,
   `description`, and `inputSchema`.
3. Sending `tools/call` for a registered tool with valid arguments invokes the handler and returns
   `result.isError == False` with the handler's return value reflected in `result.content`.
4. Sending `tools/call` for a registered tool whose handler raises `ValueError("bad phone number")` returns
   `result.isError == True` with `"bad phone number"` present in `result.content`, and the server continues to
   respond to a subsequent request afterward (process/loop didn't die).
5. Sending `tools/call` for an unregistered tool name returns `result.isError == True` mentioning that name.
6. Sending a request with an unknown `method` returns a top-level `error.code == -32601`.
7. Sending `tools/call` with `params` missing the required `name` key returns a top-level `error` (not an
   uncaught exception), with `code` in `{-32600, -32602}`.
8. Every non-malformed-line response's `id` matches the request's `id`.
9. `serve_forever()` on a transport whose stream reaches EOF after N valid requests processes all N and then
   returns without raising.

## Tasks

1. Create `src/peddler/mcp/server.py` with JSON-RPC error-code constants and a `Server` class.
2. Implement `handle_one`: read → dispatch by `method` → build response → write; catch `TransportError` from the
   read step and log to `stderr` instead of responding.
3. Implement the `initialize`, `tools/list`, and `tools/call` method handlers, including the try/except around
   the registered handler call that converts exceptions to `isError: true` results.
4. Implement `serve_forever` as a thin loop over `handle_one`.
5. Write `tests/mcp/test_server.py` covering all nine acceptance criteria, using a fake/stub `Transport` and a
   real `ToolRegistry` with test handlers (including one that raises).

## Deliverables

- `src/peddler/mcp/server.py` — `Server` class (`handle_one`, `serve_forever`), JSON-RPC error-code constants.
- `tests/mcp/test_server.py`

## Dependencies

- `1-stdio-transport`
- `2-tool-registry`
