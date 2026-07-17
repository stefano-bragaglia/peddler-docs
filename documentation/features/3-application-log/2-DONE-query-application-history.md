# 2. DONE - Query Application History

## What it does

Provides the read path for the application log — an MCP tool that lists past `/apply` attempts, optionally
filtered by URL, so the user (or a future feature) can see application history without needing to locate and
parse the raw log file themselves.

## Inputs / outputs

- Input: `list_applications(url: str | None = None)`.
- Output: a list of entries (each `{"url", "timestamp", "outcome"}`), read from the same file `1-record-application`
  writes to, filtered to `url` if provided, otherwise all entries, in the order they were recorded (oldest first).

## Edge cases and failure modes

- Log file doesn't exist yet (no applications recorded so far) → returns an empty list, not an error.
- A malformed/corrupt line in the log file (e.g. from an interrupted process elsewhere) → that line is skipped
  rather than aborting the whole read; well-formed entries are still returned.
- `url` filter matching zero entries → returns an empty list.

## Acceptance criteria

- With no log file present, `list_applications()` returns an empty list.
- With N recorded entries and no `url` filter, all N are returned in recording order.
- With a `url` filter, only matching entries are returned.
- A log file containing one corrupt line alongside valid ones still returns all valid entries, skipping the
  corrupt one, without raising.

## Tasks

1. Add read/parse logic to `src/peddler/applog/store.py`: iterate the log file line by line, skip lines that
   fail JSON parsing, return well-formed entries in file order.
2. Add `list_applications(url=None)` to `src/peddler/applog/tools.py`, applying the optional URL filter and
   wiring into the `1-mcp-server-framework` registry.
3. Extend `tests/applog/test_store.py` and `tests/applog/test_tools.py` with the read-path cases: empty file,
   multiple entries, URL filtering, corrupt-line skipping.

## Deliverables

- `src/peddler/applog/store.py` — read/parse logic added alongside the append logic from story 1 (same file).
- `src/peddler/applog/tools.py` — the `list_applications` MCP tool wrapper, added alongside `record_application`.
- `tests/applog/test_store.py` — read/filter/corrupt-line tests added alongside story 1's tests.
- `tests/applog/test_tools.py` — tool-level tests for `list_applications` added alongside story 1's tests.

## Dependencies

`1-record-application` (reads the file that story writes); `1-mcp-server-framework` (tool registration/dispatch).
