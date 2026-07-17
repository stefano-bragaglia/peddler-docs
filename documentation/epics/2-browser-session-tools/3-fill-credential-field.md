# 3. Fill Credential Field

## What it does

Implements `fill_credential_field(field_id, site)` — fills a form field (typically a password field) with the
credential stored for `site`, resolved entirely server-side. Unlike `fill_field`, the value is never supplied by
the caller and never appears in the tool's result: the plaintext password stays inside this function's scope,
read directly from the credential store and applied to the page. This is how a login step gets a stored password
onto the page without that password ever entering the LLM's context or any conversation transcript.

## Inputs / outputs

- `fill_credential_field(field_id: str, site: str) -> FillFieldResult` — same result shape as `fill_field`
  (`{"status": "ok"}` or `{"status": "error", "field": field_id, "reason": "..."}"`), but the value filled is
  looked up internally for `site` rather than passed in. The result object never contains a `password`/value key
  or the literal password string under any circumstance.

## Edge cases and failure modes

- No stored credentials for `site` → `{"status": "error", "reason": "no credentials for site"}`, not an
  exception.
- No session open → `{"status": "error", "reason": "no session open"}`, same convention as `fill_field`.
- `field_id` not found on the current page → structured "field not found" error, consistent with `fill_field`.
- The credential store raises `CredentialStoreCorruptError` → surfaced as a structured tool error, not an
  unhandled exception.

## Acceptance criteria

- Given stored credentials for a site and an open session with a password field, calling
  `fill_credential_field(field_id, site)` returns `status: "ok"`, and reading the field's value back **from the
  page itself** (not from the tool's return value) confirms the stored password was applied.
- The tool's returned dict never contains the literal password string or a `password`/`value` key — asserted
  directly on the tool's response object in the test.
- Given no stored credentials for `site`, returns a structured "no credentials for site" error.
- Given no open session, returns a structured "no session open" error.

## Tasks

1. Add `fill_credential_field(field_id, site)` to `src/peddler/browser/fields.py`, importing the credential
   store's `get(site)` read primitive from `src/peddler/credentials/store.py`.
2. Implement the internal password lookup and field fill, sharing the field-locating logic from `fill_field`
   without ever returning the looked-up value.
3. Implement the "no credentials for site" / "no session open" / "field not found" structured error paths.
4. Extend `tests/browser/test_fields.py` with credential-field tests, including an explicit assertion that the
   returned dict never contains the password value or a `password` key.

## Deliverables

Extends `src/peddler/browser/fields.py` with `fill_credential_field`; imports the credential store's `get(site)`
read primitive from `src/peddler/credentials/store.py` (epic `2-credentials-log-book`, story
`1-log-book-storage`). Test additions in `tests/browser/test_fields.py`.

## Dependencies

Story 2 (Session Lifecycle), same as `fill_field`/`advance_page`. **Cross-epic**: also depends on
`2-credentials-log-book`'s `1-log-book-storage` story — this is the one point where these two wave-2 epics are
no longer fully independent; `1-log-book-storage` must be complete before this specific story can be implemented
(Stage A can still be written against a stub/fake store, but Stage B needs the real one).
