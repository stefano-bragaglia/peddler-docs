# 1. DONE - Install Apply Command

## What it does

Extends the `peddler` launcher with a step that installs the packaged `/apply` command file at
`~/.claude/commands/apply.md`, so Claude Code actually discovers it — closing the gap where the command existed
inside the installed package but was never copied anywhere Claude Code looks.

## Inputs / outputs

- No new CLI flags. Runs as part of `peddler`'s existing startup sequence, before MCP registration.
- Reads the packaged `apply.md` (`peddler.commands` package data) and writes to `~/.claude/commands/apply.md`,
  creating the `~/.claude/commands/` directory if it doesn't exist.
- Output: none on success; a warning printed to stderr on failure (not fatal — `peddler` still proceeds).

## Edge cases and failure modes

- `~/.claude/commands/apply.md` doesn't exist yet → created, directory included.
- `~/.claude/commands/apply.md` exists with content identical to the packaged version → no-op, not rewritten.
- `~/.claude/commands/apply.md` exists with *different* content (an older `peddler` version, a hand edit, or an
  unrelated file that happens to live at that path) → overwritten unconditionally; `peddler` owns this path by
  convention, no attempt to detect "is this really ours."
- Write fails (e.g. permissions on `~/.claude/`) → a warning is printed, and `peddler` continues to the
  Playwright check, MCP registration, and `claude` handoff — this step failing doesn't block the rest of launch.

## Acceptance criteria

1. Given `~/.claude/commands/apply.md` doesn't exist, running the install step creates
   `~/.claude/commands/apply.md` with content identical to the packaged `apply.md`, creating the parent directory
   if needed.
2. Given `~/.claude/commands/apply.md` already exists with content identical to the packaged version, running the
   install step again does not rewrite the file (verified via an injected/fake filesystem or mtime/write-count
   check — no unnecessary write on a no-op run).
3. Given `~/.claude/commands/apply.md` exists with different content, running the install step overwrites it with
   the packaged version.
4. Given the write fails (simulated permission error), the install step returns/logs a warning rather than
   raising, and `main()`'s overall flow continues past it to the next step (Playwright check).
5. `main()` calls this install step before the Playwright check and MCP registration (verified via call-order
   tracking in the test, using the same injectable-fake pattern as the rest of `launcher.py`).

## Tasks

1. Add an `_install_apply_command()` (or similarly named) function to `src/peddler/launcher.py`: reads the
   packaged `apply.md` (e.g. via `importlib.resources` against `peddler.commands`), compares against
   `~/.claude/commands/apply.md` if it exists, writes only when different (creating the parent directory as
   needed), and catches/warns on any `OSError` rather than propagating it.
2. Wire it into `main()` as an injectable step (matching the existing `check_playwright`/`register_mcp`/`exec_fn`
   pattern), called first, before the Playwright check.
3. Extend `tests/test_launcher.py` with the five acceptance criteria above, using `tmp_path` for the fake home
   directory (never touching the real `~/.claude/commands/` in tests).

## Deliverables

- `src/peddler/launcher.py` — extended with the command-install step.
- `tests/test_launcher.py` — extended with the new acceptance-criteria tests.

## Dependencies

None at the story level — this feature has only one story.
