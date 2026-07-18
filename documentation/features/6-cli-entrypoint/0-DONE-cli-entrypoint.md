# 6. DONE - CLI Entrypoint & MCP Server Wiring

## Summary

Closes the gap discovered after all five original features shipped: every tool was built and unit-tested in
isolation, but nothing ever assembled them into a runnable MCP server process, and nothing told Claude Code how
to find it — running `/apply` would fail immediately. This feature adds the `peddler-mcp` console script (the
actual MCP server, wiring every feature's tools — including the never-registered browser tools — into one
`ToolRegistry`) and the `peddler` console script (the user-facing launcher: checks Playwright is installed,
registers `peddler-mcp` with Claude Code, then hands off to `claude` in the target workspace).

## Requirements Covered

- Requirement 13 — the `peddler` launcher CLI (`--dir`, `--credentials`, `--applog`, `--` passthrough).
- Requirement 14 — the `peddler-mcp` console script wiring every feature's tools into one `ToolRegistry`.
- Requirement 15 — idempotent MCP registration via `claude mcp add ... --scope local`.
- Requirement 16 — process-replacement (`exec`) lifecycle, no manual process-pairing code.
- Requirement 17 — Playwright browser-binary preflight check.
- Non-functional: Packaging — `[project.scripts]` entry points for both console scripts.
- Non-functional: Platform — POSIX-only (`os.execvp`), by design.
- Non-functional: Concurrency (partial) — documents the file-locking mitigation-by-convention for simultaneous
  sessions; no code changes to `CredentialStore`/`ApplicationLog` themselves.

## Dependencies

- `1-mcp-server-framework` — the `Transport`/`ToolRegistry`/`Server` framework `peddler-mcp` wires everything
  into.
- `2-credentials-log-book`, `3-application-log`, `4-browser-session-tools` — the tool sets `peddler-mcp`
  registers. In particular, `4-browser-session-tools`'s tools were never registered against a `ToolRegistry` at
  all; this feature adds that missing wiring.
- `5-apply-orchestration` — not a hard dependency, but this feature is what actually makes `/apply` runnable
  end to end.

## Stories

See the numbered story files alongside this one in `documentation/features/6-cli-entrypoint/`.
