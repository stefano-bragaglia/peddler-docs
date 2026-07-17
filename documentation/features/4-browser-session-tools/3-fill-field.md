# 3. Fill Field

## What it does

Implements the `fill_field(field_id, value)` tool. Locates the named field on the current page (from the open
session) and sets its value, then returns whether the page accepted it or surfaced a validation error for that
field.

## Inputs / outputs

- `fill_field(field_id: str, value: str) -> FillFieldResult` — returns `{"status": "ok"}` on successful fill, or
  `{"status": "error", "field": field_id, "reason": "<page-surfaced validation message>"}` if the page rejects
  the value (e.g. "invalid phone number format") or the field cannot be located.

## Edge cases and failure modes

- No session is currently open → structured error (e.g. `{"status": "error", "reason": "no session open"}"`), no
  exception raised to the MCP transport layer.
- `field_id` does not match any field on the current page → structured error identifying the field as not found,
  distinct from a validation error.
- The page surfaces a validation error only after the field loses focus / on a later `advance_page()` call rather
  than immediately — `fill_field` itself may return `status: "ok"` in that case; the delayed error is expected to
  surface via `advance_page()` instead (documented so the orchestration feature, `5-apply-orchestration`, knows
  not to assume `fill_field`'s success is the final word on validity).
- Field type variations (text input, select/dropdown, checkbox, radio) — `value` semantics differ per type (e.g.
  a select's `value` is the option to choose); acceptance criteria below only need to cover a plain text input
  and one non-text control (e.g. a checkbox) to establish the pattern.

## Acceptance criteria

- Given an open session on a page with a text field `field_id`, calling `fill_field(field_id, "some value")`
  returns `status: "ok"` and the field's value in the page reflects `"some value"`.
- Given a field that immediately rejects a value (e.g. a client-side validated phone field), calling
  `fill_field` with an invalid value returns `status: "error"` with a human-readable `reason` string.
- Given no open session, calling `fill_field` returns a structured "no session open" error rather than raising.
- Given a `field_id` not present on the current page, calling `fill_field` returns a structured "field not found"
  error.
- Given a checkbox-type field, calling `fill_field(field_id, "true")` (or equivalent boolean-ish value) checks it,
  verified by reading the checkbox's checked state back from the page.

## Tasks

1. Create `src/peddler/browser/fields.py` with a `fill_field(field_id, value)` tool function.
2. Implement field lookup against the current session's page, "no session open" and "field not found" error
   paths.
3. Implement value-setting dispatch by field type (text input vs. checkbox at minimum), and validation-error
   capture into the structured error result.
4. Write `tests/browser/test_fields.py` covering all five acceptance criteria against local test HTML fixtures
   (text field, validated phone field, checkbox).

## Deliverables

`src/peddler/browser/fields.py` (`fill_field` tool function and field-type dispatch),
`tests/browser/test_fields.py`.

## Dependencies

Story 2 (Session Lifecycle) — operates on the session it manages.
