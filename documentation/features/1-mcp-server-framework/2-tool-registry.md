# 2. DONE - Tool Registry

## What it does

A pure in-memory registry mapping a tool name to its JSON Schema (for parameters) and its Python handler
function. Provides registration and lookup; has no knowledge of stdio or JSON-RPC.

## Inputs / Outputs

- `register(name: str, description: str, parameters_schema: dict, handler: Callable[[dict], dict]) -> None` —
  adds a tool. Handler takes the parsed `arguments` dict from a `tools/call` request and returns a plain `dict`
  result.
- `get(name: str) -> ToolSpec | None` — returns the registered tool spec (name, description, schema, handler) or
  `None` if not found.
- `list_tools() -> list[dict]` — returns `[{"name": ..., "description": ..., "inputSchema": ...}, ...]` for every
  registered tool, in registration order — the shape expected in an MCP `tools/list` response.

## Edge cases and failure modes

- Registering two tools under the same `name` → raises `ValueError` (no silent overwrite — a naming collision is
  a programming error, not a runtime condition to handle gracefully).
- `get`/dispatch-time lookup of an unregistered name → returns `None`, letting the caller (the dispatch story)
  decide how to turn that into a JSON-RPC error.
- `parameters_schema` isn't itself validated for being well-formed JSON Schema — that's out of scope for this
  story (garbage schema in means garbage schema out to `tools/list`; MCP clients validate on their end).

## Acceptance criteria

1. Registering a tool then calling `get` with that name returns a spec with the same description, schema, and
   handler reference.
2. `get` with an unregistered name returns `None`.
3. Registering the same name twice raises `ValueError` and the original registration is left intact (`get` still
   returns the first one).
4. `list_tools()` on an empty registry returns `[]`.
5. `list_tools()` after registering two tools returns both, each with keys `name`, `description`, `inputSchema`,
   in the order they were registered.

## Tasks

1. Create `src/peddler/mcp/registry.py` with a `ToolSpec` dataclass (`name`, `description`, `parameters_schema`,
   `handler`) and a `ToolRegistry` class.
2. Implement `register` (collision check → `ValueError`), `get`, and `list_tools` (preserving registration
   order — a plain `dict` keyed by name is already insertion-ordered in Python).
3. Write `tests/mcp/test_registry.py` covering all five acceptance criteria.

## Deliverables

- `src/peddler/mcp/registry.py` — `ToolRegistry` class and `ToolSpec` dataclass.
- `tests/mcp/test_registry.py`

## Dependencies

None.
