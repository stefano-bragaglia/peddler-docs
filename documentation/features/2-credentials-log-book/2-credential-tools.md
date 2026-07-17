# 2. Credential Tools

## What it does

Registers `read_credentials(site)` and `write_credentials(site, username, password)` as MCP tools against the
server framework from epic `1-mcp-server-framework`, backed by the storage from story `1-log-book-storage`.
These two tools are the *only* way the LLM (or any other code path) touches the credentials file — there is no
other exposed path to it.

## Inputs / outputs

- `read_credentials(site: str) -> {"found": bool, "username": str | None}` — deliberately does **not** return the
  password in the tool result surfaced to the LLM's context; the password stays server-side. Confirmed: logging
  in with a stored password is done via `2-browser-session-tools`'s `fill_credential_field(field_id, site)`,
  which reads the password directly from this feature's store and applies it to the page without it ever
  entering the LLM's context or a conversation transcript.
- `write_credentials(site: str, username: str, password: str) -> {"ok": bool}` — stores the entry via
  `1-log-book-storage`.

## Edge cases and failure modes

- `read_credentials` for a site with no stored entry → `{"found": false, "username": None}`, not an error.
- `write_credentials` called with an empty `username` or `password` → reject with a structured tool error
  (`{"ok": false, "error": "..."}`), not a silent no-op or an uncaught exception.
- Underlying store raising `CredentialStoreCorruptError` → surfaced to the LLM as a structured tool error so it
  can report the problem to the user rather than crashing the MCP server process.

## Acceptance criteria

- Calling `read_credentials` for a never-seen site returns `found: false` and no exception.
- Calling `write_credentials` then `read_credentials` for the same site returns `found: true` with the matching
  username.
- `write_credentials` with a blank username or password returns a structured failure, verified by a test, not an
  unhandled exception.
- A corrupt backing store surfaces as a structured tool error from both tools, not an unhandled exception
  crashing the server.

## Tasks

1. Create `src/peddler/credentials/tools.py` with `read_credentials`/`write_credentials` handler functions.
2. Wire both into the `1-mcp-server-framework` tool registry (name, description, JSON Schema parameters).
3. Implement the blank-username/password validation and the `CredentialStoreCorruptError` → structured-error
   translation.
4. Write `tests/credentials/test_tools.py` covering all four acceptance criteria.

## Deliverables

- `src/peddler/credentials/tools.py` — tool handler functions/registration for `read_credentials` and
  `write_credentials`, wired into the framework exposed by `1-mcp-server-framework`.
- `tests/credentials/test_tools.py`

## Dependencies

`1-log-book-storage` (this feature); the tool-registration mechanism from `1-mcp-server-framework`.
