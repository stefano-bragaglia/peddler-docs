# 2. Session Lifecycle

## What it does

Implements the `open_session(url)` and `close_session()` tools. `open_session` launches a headless Playwright
browser, navigates to `url` (wrapped in the Retry Policy from story 1), and holds the resulting browser/page as a
single global session object. `close_session` tears that session down. Only one session may be open at a time
(per the Concurrency non-functional requirement).

## Inputs / outputs

- `open_session(url: str) -> SessionOpenResult` — on success, returns the current page's content (e.g. simplified
  HTML/text representation sufficient for the LLM to identify form fields) and a status of `"opened"`. On
  failure (retries exhausted), returns/raises a structured error (e.g. `{"status": "error", "reason": "..."}"`)
  rather than crashing the MCP server process.
- `close_session() -> SessionCloseResult` — returns `{"status": "closed"}` on success.

## Edge cases and failure modes

- `open_session` called while a session is already open → returns a structured error (e.g.
  `{"status": "error", "reason": "session already open"}`) rather than silently replacing it or opening a second
  browser.
- `close_session` called when no session is open → returns a structured no-op result (e.g.
  `{"status": "no_session"}`), not an exception that would crash the tool call.
- `open_session`'s navigation exhausts all 3 retries (per story 1) → returns a structured error identifying it as
  a navigation failure, distinguishable from e.g. an invalid-URL error.
- Process crashes/is killed with a session still open → next `open_session` call must not be blocked by stale
  state; session state lives only in this process's memory for the duration of one `/apply` invocation.

## Acceptance criteria

- Calling `open_session(url)` with a reachable URL returns `status: "opened"` plus non-empty page content.
- Calling `open_session(url)` a second time before `close_session()` returns a structured "already open" error
  and does not launch a second browser instance.
- Calling `close_session()` after a successful `open_session` returns `status: "closed"` and subsequent
  `close_session()` calls return `status: "no_session"`.
- Calling `open_session(url)` against an unreachable/timing-out URL results in exactly 3 navigation attempts
  (verified via the Retry Policy wrapper) before returning a structured navigation-failure error.

## Tasks

1. Create `src/peddler/browser/session.py` with a module-level (or class-wrapped) single-session state holder.
2. Implement `open_session`: reject if already open, launch Playwright headless browser, navigate wrapped in
   `retry_with_backoff` from story 1, capture page content, store session state.
3. Implement `close_session`: tear down the browser/page, clear session state, handle the no-session case.
4. Write `tests/browser/test_session.py` covering all four acceptance criteria, using a stub/fake Playwright
   layer (or a local test HTML page) so tests don't depend on real network access.

## Deliverables

`src/peddler/browser/session.py` (session state + `open_session`/`close_session` tool functions),
`tests/browser/test_session.py`.

## Dependencies

Story 1 (Retry Policy) — navigation is wrapped in it.
