# 4. DONE - Advance Page

## What it does

Implements the `advance_page()` tool. Submits/advances the current page (e.g. clicking a "Next"/"Submit"
control), wrapped in the Retry Policy from story 1 for the page-transition itself, and returns either the new
page's content or any field-specific errors the page surfaced instead of transitioning.

## Inputs / outputs

- `advance_page() -> AdvancePageResult` — on a successful transition, returns
  `{"status": "advanced", "content": "<new page content>"}`. If the page stays put and surfaces field errors
  instead, returns `{"status": "error", "field_errors": {"<field_id>": "<message>", ...}}`.

## Edge cases and failure modes

- No session open → structured "no session open" error, same convention as Fill Field.
- Page has multiple simultaneous field errors (e.g. two required fields left empty) → all are returned together
  in `field_errors`, not just the first one found, so the orchestration layer can fix them all before retrying.
- The transition itself times out or the browser crashes mid-transition → retried per the Retry Policy (story 1)
  up to 3 attempts; if still failing after that, returns a structured navigation-failure error (distinct from a
  `field_errors` validation failure) so the caller can tell "the form rejected my data" apart from "the browser
  couldn't complete the action."
- Page transitions to a page with no recognizable form and no recognizable success indicator (e.g. a generic
  error page) → still returns `status: "advanced"` with that page's content; judging whether it's success,
  another form, or something gone wrong is explicitly out of scope for this tool (LLM judgment, per
  `5-apply-orchestration`) — this tool only reports what happened mechanically.

## Acceptance criteria

- Given an open session with all required fields validly filled, calling `advance_page()` returns
  `status: "advanced"` with the new page's content.
- Given an open session with an invalid/missing required field, calling `advance_page()` returns
  `status: "error"` with a `field_errors` map containing at least that field.
- Given a page with two invalid fields, calling `advance_page()` returns both in `field_errors`.
- Given no open session, calling `advance_page()` returns a structured "no session open" error.
- Given a transition that fails 3 times transiently (simulated timeout), `advance_page()` calls the underlying
  transition exactly 3 times (per the Retry Policy) before returning a structured navigation-failure error.

## Tasks

1. Create `src/peddler/browser/navigation.py` with an `advance_page()` tool function.
2. Implement the submit/click action wrapped in `retry_with_backoff` from story 1.
3. Implement field-error collection (scan the resulting page for validation messages, collect all of them) versus
   a genuine navigation-failure error path.
4. Write `tests/browser/test_navigation.py` covering all five acceptance criteria against local test HTML
   fixtures (valid submit, single invalid field, two invalid fields, simulated transient failure).

## Deliverables

`src/peddler/browser/navigation.py` (`advance_page` tool function), `tests/browser/test_navigation.py`.

## Dependencies

Story 2 (Session Lifecycle) — operates on the session it manages. Has no dependency on Story 3 (Fill Field);
sequenced after it here purely as an arbitrary tie-break between two stories that could be built in either
order.
