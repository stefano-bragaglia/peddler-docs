# 2. DONE - Peddler Launcher

## What it does

The `peddler` console script — the user-facing launcher. Checks Playwright's browser binaries are actually
usable, registers `peddler-mcp` (story 1) with Claude Code (idempotently, local scope), and hands off to `claude`
in the target workspace by replacing its own process (`exec`) — so closing the terminal tears the whole session,
MCP server included, down together via ordinary OS process/signal propagation.

## Inputs / outputs

- `--dir` (default: current working directory) — the folder to launch Claude Code in.
- `--credentials` (default: `~/.peddler/credentials.json`).
- `--applog` (default: `~/.peddler/applications.log`).
- Everything after a `--` separator is forwarded verbatim as additional arguments to the `claude` invocation.
- Output: none on success (the process is replaced by `claude`); a nonzero exit and a clear stderr message on
  any failure.

## Edge cases and failure modes

- `--dir` doesn't exist → clear error before attempting anything else, not a confusing failure further downstream.
- Playwright's browser binaries aren't installed → fails fast with a fix-it message (e.g. "run `uv run
  playwright install chromium`"), before attempting MCP registration or exec. Detected by actually trying to
  launch a browser via the same `browser_factory` seam `open_session` itself uses (story 4 of feature 4) —
  Playwright's own error already explains the fix, so no separate detection heuristic is built; a real attempt
  is caught and translated into this message rather than re-implemented as a path/version check.
- `peddler` is already registered locally for this scope (a prior invocation) → treated as success, not an error
  or a duplicate registration.
- The MCP registration command fails for any other reason (e.g. `claude` not on `PATH`) → fails loudly with the
  underlying error surfaced, and does not proceed to exec `claude`.
- `claude` itself isn't found at exec time → clear error, not a raw `OSError` traceback.

## Acceptance criteria

1. Given Playwright's browser binaries are available (mocked check succeeds) and MCP registration succeeds,
   calling the launcher execs `claude` in the resolved `--dir` (verified via an injectable exec function — real
   `os.execvp` replaces the process and can't be exercised directly in a unit test).
2. Given Playwright's browser binaries are unavailable (mocked check raises/fails), the launcher exits nonzero
   with a message mentioning `playwright install`, and does not attempt MCP registration or exec.
3. Given `peddler` is already registered locally (mocked "already registered" outcome), the launcher treats this
   as success and still proceeds to exec `claude`.
4. Given the MCP registration command fails for a genuine reason (mocked non-"already registered" failure), the
   launcher exits nonzero surfacing the underlying error, and does not exec `claude`.
5. `--credentials`/`--applog` default correctly when omitted, and are passed through as environment variables in
   the MCP registration when given explicitly.
6. Everything after `--` is forwarded verbatim as additional arguments in the final `claude` invocation.

## Tasks

1. Create `src/peddler/launcher.py` with `main(argv=None)` using `argparse` for `--dir`/`--credentials`/
   `--applog` and `--` passthrough capture.
2. Implement the Playwright preflight check as an injectable function (default: a real, brief browser
   launch-and-close using the same factory seam as `open_session`), so tests inject a fake that succeeds or
   raises.
3. Implement the MCP registration step (`claude mcp add ... --scope local`, env vars for credentials/applog paths)
   via an injectable subprocess-runner, distinguishing "already registered" (success) from any other failure
   (fails loudly).
4. Implement the final handoff via an injectable `exec_fn` (default `os.execvp`), so tests can verify the
   resolved argv without actually replacing the test process.
5. Add the `peddler = "peddler.launcher:main"` entry point to `project/pyproject.toml`'s `[project.scripts]`.
6. Write `tests/test_launcher.py` covering all six acceptance criteria using fakes for the browser check,
   subprocess runner, and exec function.

## Deliverables

- `src/peddler/launcher.py` — `main()` and its argparse/preflight/registration/exec logic.
- `tests/test_launcher.py`
- `project/pyproject.toml` — new `[project.scripts]` entry.

## Dependencies

Story 1 (`1-mcp-server-wiring`) — registers `peddler-mcp`, which this story's launcher points the MCP
registration at.
