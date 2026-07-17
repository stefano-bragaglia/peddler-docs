# 1. MCP Server & Tool Framework

## Summary

The foundational local MCP server that Claude CLI connects to for `/apply`. Implements the JSON-RPC-over-stdio
transport and tool registration/dispatch machinery, using stdlib alone (no third-party MCP SDK needed for this
piece). Every other feature adds tools onto this framework rather than building its own transport — this is what
makes tool calls (browser actions, credential lookups, logging) visible to and callable by the LLM in the first
place. Includes error propagation so a tool's failure surfaces back to the LLM as a structured result rather than
crashing the server process.

## Requirements Covered

- Requirement 3 (the MCP server itself, and its tool-exposure mechanism — the concrete tools it hosts are added
  by other features).
- Non-functional: stdlib-only implementation of the server (`Requirements.md → Non-Functional → Dependencies`).

## Dependencies

None — this is the first wave; every other feature depends on it.

## Stories

See `documentation/features/stories/1-mcp-server-framework.md` (added by `/stories`).
