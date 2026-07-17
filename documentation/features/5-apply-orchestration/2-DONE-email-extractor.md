# 2. DONE - Email Extractor

## What it does

A small deterministic helper that scans CV markdown text for a contact email address, so the Apply Command's
Sign-up Handling step can look one up reliably via a regex rather than relying on the LLM to pattern-match it
correctly every time.

## Inputs / Outputs

- Input: `cv_text: str` — the full CV markdown content.
- Output: `str | None` — the first well-formed email address found, or `None` if none is present.

## Edge Cases and Failure Modes

- Multiple email addresses in the CV — returns the first one encountered in reading order (e.g. the one in a
  contact-details header, typically near the top).
- Near-miss strings that merely resemble an email (e.g. `not-an-email@`, missing a domain) must not be returned.
- No email present anywhere in the text — returns `None`, does not raise.
- Obfuscated emails (e.g. `name (at) domain (dot) com`) are explicitly out of scope — plain regex matching only.

## Acceptance Criteria

1. Given CV text containing one standard email address, returns that address.
2. Given CV text containing no email, returns `None`.
3. Given CV text containing multiple email addresses, returns the first one encountered in the text.
4. Does not match strings that merely resemble an email but are missing a required part (e.g. no domain, no
   `@`).

## Tasks

1. Create `src/peddler/email.py` with `extract_email(cv_text: str) -> str | None` using a standard email regex.
2. Handle the "first match in reading order" and "no match" cases.
3. Write `tests/test_email.py` covering all four acceptance criteria.

## Deliverables

- `src/peddler/email.py` — the `extract_email(cv_text: str) -> str | None` helper.
- `tests/test_email.py` — covers all four acceptance criteria above.

## Dependencies

None at the story level.
