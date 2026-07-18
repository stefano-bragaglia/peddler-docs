# 7. Slash Command Installation

## Summary

Closes the gap discovered immediately after `6-cli-entrypoint` shipped: `peddler` made `/apply` *runnable* (an
MCP server now exists and gets registered), but never made it *discoverable* — Claude Code finds slash commands
from `.md` files in `.claude/commands/`, and nothing ever put `apply.md` there. Confirmed by hand: Claude Code
reported `Unknown command: /apply` even after a correct `peddler` install and run. This feature adds that missing
install step to the `peddler` launcher itself.

## Requirements Covered

- Requirement 18 — installs the packaged `apply.md` at user scope (`~/.claude/commands/apply.md`), overwrite-if-
  differs, overwrite-anyway on an unrecognized existing file, warning-not-fatal on failure.

## Dependencies

- `5-apply-orchestration` — the `apply.md` file this feature installs.
- `6-cli-entrypoint` — extends the `peddler` launcher this feature adds a step to.

## Stories

See the numbered story files alongside this one in `documentation/features/7-slash-command-installation/`.
