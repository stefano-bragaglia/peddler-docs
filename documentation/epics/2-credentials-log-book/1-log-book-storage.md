# 1. Log Book Storage

## What it does

A file-backed store for per-site credential entries, keyed by site. Provides the read/write primitives the MCP
tools (story 2) sit on top of. Not itself LLM-callable — internal to the package.

## Inputs / outputs

- `get(site: str) -> CredentialEntry | None` — `CredentialEntry` has `username: str`, `password: str`. Returns
  `None` if no entry exists for that site.
- `put(site: str, username: str, password: str) -> None` — creates or overwrites the entry for that site.
- Backing file: a local JSON file at a fixed path (e.g. `~/.peddler/credentials.json`), created on first write if
  it doesn't exist.

## Edge cases and failure modes

- Backing file doesn't exist yet → `get()` returns `None` for any site; `put()` creates the file and its parent
  directory.
- Backing file exists but is empty or contains invalid JSON → treated as corrupt; raise a clear exception rather
  than silently discarding the user's stored credentials.
- Site key normalization — the same site should match regardless of trivial URL variation (e.g.
  `https://acme.example.com/apply` vs `acme.example.com`); store keys by hostname only (lowercased), not full URL.
- `put()` for a site that already has an entry overwrites it (upsert semantics) — no separate "update" API needed.
- Concurrent writes from multiple `/apply` invocations at once are out of scope for v1 (single active session
  per the Concurrency non-functional requirement) — no file locking required.

## Acceptance criteria

- `get()` on an empty/missing store returns `None` for any site.
- `put()` followed by `get()` for the same hostname returns the stored username/password exactly.
- `get()` is case-insensitive and scheme/path-insensitive on the site argument (normalizes to hostname, lowercased).
- `put()` called twice for the same hostname with different credentials results in `get()` returning the second
  (most recent) set.
- Passing a corrupt backing file raises a specific, catchable exception (not a bare `JSONDecodeError` bubbling up
  uncaught) — e.g. a `CredentialStoreCorruptError`.

## Tasks

1. Create `src/peddler/credentials/store.py` with `CredentialEntry` and a hostname-normalization helper (parse
   URL/bare-hostname input, lowercase).
2. Implement `get`/`put` against a JSON file at a fixed path, creating the parent directory on first write.
3. Implement `CredentialStoreCorruptError` and raise it on invalid/empty JSON content instead of letting
   `JSONDecodeError` propagate.
4. Write `tests/credentials/test_store.py` covering all five acceptance criteria, using a temp directory for the
   backing file path.

## Deliverables

- `src/peddler/credentials/store.py` — `CredentialEntry`, `CredentialStore` (or equivalent functions), backing
  file path constant.
- `tests/credentials/test_store.py`

## Dependencies

None.
