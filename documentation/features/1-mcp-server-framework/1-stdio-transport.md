# 1. DONE - Stdio Transport

## What it does

Reads and writes newline-delimited JSON-RPC 2.0 messages over `stdin`/`stdout`, blocking on read until a full
line is available. This is pure message framing — it knows nothing about MCP methods, tools, or the registry; it
just moves JSON objects in and out.

## Inputs / Outputs

- `read_message() -> dict | None`: blocks reading one line from `stdin`, parses it as JSON, returns the resulting
  `dict`. Returns `None` on EOF (stdin closed).
- `write_message(message: dict) -> None`: serializes `message` to a single JSON line (no embedded newlines) and
  writes it to `stdout`, followed by a newline, then flushes.
- Construction takes the input/output streams explicitly (e.g. `Transport(stdin=sys.stdin, stdout=sys.stdout)`)
  so tests can substitute `io.StringIO`.

## Edge cases and failure modes

- A line that isn't valid JSON → raises a clear `TransportError` (not a raw `json.JSONDecodeError`) identifying
  the offending line, so the caller can decide how to respond (per JSON-RPC, an error response referencing
  `id: null` when the id can't be determined).
- A line that parses as JSON but isn't a JSON *object* (e.g. a bare number or array) → same `TransportError`.
- Empty line (blank line with no content) → skipped, read continues to the next line rather than erroring.
- `write_message` given a value containing a literal newline inside a string field → must not break framing
  (rely on `json.dumps`'s default escaping, verified by a test).
- EOF mid-read (stream closed) → `read_message` returns `None` rather than raising.

## Acceptance criteria

1. Writing `{"a": 1}` then reading it back through a shared `StringIO` round-trips to an equal dict.
2. `read_message` on an empty stream returns `None`.
3. `read_message` on a stream containing a blank line followed by a valid JSON line returns the parsed dict,
   skipping the blank line.
4. `read_message` on a line containing invalid JSON raises `TransportError`.
5. `read_message` on a line containing valid JSON that isn't an object (e.g. `"[1, 2]"`) raises `TransportError`.
6. `write_message` output ends with exactly one newline character and contains no embedded raw newline within
   the JSON payload itself.

## Tasks

1. Create `src/peddler/mcp/transport.py` with `TransportError` and a `Transport` class taking `stdin`/`stdout`
   streams in its constructor.
2. Implement `write_message` (serialize, write, newline, flush).
3. Implement `read_message` (readline loop skipping blank lines, JSON parse, type check for `dict`, `None` on
   EOF, `TransportError` on bad input).
4. Write `tests/mcp/test_transport.py` covering all six acceptance criteria using `io.StringIO`.

## Deliverables

- `src/peddler/mcp/transport.py` — `Transport` class and `TransportError` exception.
- `tests/mcp/test_transport.py`

## Dependencies

None.
