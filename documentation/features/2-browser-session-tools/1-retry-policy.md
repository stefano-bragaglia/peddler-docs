# 1. Retry Policy

## What it does

A generic auto-retry-with-backoff wrapper applied around transient, network/browser-crash-prone operations
(initial navigation, page transitions). Retries a failed operation up to 3 attempts total, with a short backoff
between attempts (e.g. exponential: ~0.5s, ~1s, ~2s), before letting the final failure propagate to the caller.

## Inputs / outputs

Input — a zero-arg callable (or callable + args) representing the operation to attempt, and a tuple of exception
types considered "transient" (timeout, connection/crash errors). Output — the callable's return value on
success, or the last exception re-raised after the 3rd failed attempt.

## Edge cases and failure modes

- Operation fails all 3 attempts → the original exception propagates unchanged (not wrapped/swallowed), so the
  caller can distinguish a retry-exhausted transient failure from other errors.
- Operation raises a *non*-transient exception type (e.g. a plain validation error, not in the transient set) →
  no retry, propagates immediately on first failure.
- Operation succeeds on attempt 2 or 3 → no further attempts made, no residual side effects from prior failed
  attempts (caller's operation must itself be safely retryable/idempotent — this story only provides the
  wrapper, not idempotency guarantees for what's wrapped).

## Acceptance criteria

- Given an operation that fails twice then succeeds, the wrapper returns the success value and calls the
  operation exactly 3 times.
- Given an operation that fails all 3 attempts with a transient exception type, the wrapper re-raises that same
  exception type after the 3rd attempt and calls the operation exactly 3 times.
- Given an operation that raises a non-transient exception type, the wrapper calls the operation exactly once and
  re-raises immediately, with no backoff delay incurred.
- Given an operation that succeeds on the first attempt, the wrapper calls it exactly once with no delay.

## Tasks

1. Create `src/peddler/browser/retry.py` with a `retry_with_backoff(operation, transient_exceptions, attempts=3)`
   function (or equivalent decorator).
2. Implement the transient/non-transient branching and the backoff delay (injectable sleep function so tests
   don't actually wait).
3. Write `tests/browser/test_retry.py` covering all four acceptance criteria, asserting call counts and using a
   fake/injected sleep to avoid real delays in the test suite.

## Deliverables

`src/peddler/browser/retry.py` (the retry-with-backoff wrapper), `tests/browser/test_retry.py`.

## Dependencies

None (wave 1 — depends only on feature `1-mcp-server-framework` at the feature level, no intra-feature story
dependency).
