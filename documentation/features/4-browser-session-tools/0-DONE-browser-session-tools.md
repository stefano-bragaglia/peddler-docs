# 4. DONE - Browser Session Tools

## Summary

The Playwright-backed tools that give the LLM control over a single headless browser session: opening it,
filling fields, advancing pages, and closing it. These are the "dumb executors" the orchestration logic
(feature 5) drives — they don't decide *what* to fill in, only *how* to perform a browser action and report back
what happened, including page-surfaced validation errors. Also owns the reliability behavior around transient
browser/network failures: an auto-retry with backoff before giving up.

## Requirements Covered

- Requirement 3 — the `open_session(url)`, `fill_field(field_id, value)`, `advance_page()`, and `close_session()`
  tools.
- Requirement 6 (partial) — surfacing field/page errors accurately enough for retries to be possible.
- Non-functional: Reliability — auto-retry up to 3 attempts with a short backoff on browser crash, timeout, or
  network drop before falling back to Stuck Handling (`Requirements.md → Non-Functional → Reliability`).
- Non-functional: Dependencies — approved use of Playwright, with Selenium as the agreed fallback if Playwright
  doesn't work out (`Requirements.md → Non-Functional → Dependencies`).
- Non-functional: Concurrency — one active browser session at a time.

## Dependencies

- `1-mcp-server-framework` — tools are registered against that server.
- `2-credentials-log-book` — one story (`fill_credential_field`) reads that feature's credential store directly;
  this is why this feature is sequenced after it rather than immediately after `1-mcp-server-framework`.

## Stories

See the numbered story files alongside this one in `documentation/features/4-browser-session-tools/`.
