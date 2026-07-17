# 2. Password Generator

## What it does

Generates a cryptographically secure random password for new site sign-ups, using the stdlib `secrets` module
(never `random`).

## Inputs / outputs

- `generate_password(length: int = 20) -> str` — returns a string mixing uppercase, lowercase, digits, and a
  small set of symbols, of the given length.

## Edge cases and failure modes

- `length` too small to satisfy "at least one of each character class" — either raise `ValueError` for
  unreasonably small lengths (e.g. `< 8`) or relax the guarantee; document whichever choice is made in the
  docstring so Stage B and Stage A agree.
- Must use `secrets.choice` / `secrets.SystemRandom`, not `random`, for every random choice (security
  requirement, not just style).
- Some sites reject certain symbols (e.g. no `<`, `>`, quotes) — since this cannot be known ahead of time, keep
  the symbol set conservative (e.g. `!@#$%^&*-_=+`) rather than the full punctuation set, to reduce the odds a
  generated password gets rejected by a target site's own validation.

## Acceptance criteria

- Generated password has the requested length.
- Generated password contains at least one uppercase letter, one lowercase letter, and one digit (for any
  `length >= 8`).
- Two consecutive calls produce different passwords (no fixed seed / determinism).
- No use of the `random` module anywhere in the implementation (only `secrets`).

## Tasks

1. Create `src/peddler/credentials/passwords.py` with `generate_password(length=20)`.
2. Implement the character-class guarantee (at least one uppercase/lowercase/digit) via `secrets.choice`, and
   raise `ValueError` for `length < 8`.
3. Write `tests/credentials/test_passwords.py` covering all four acceptance criteria, including a static-analysis
   style check (e.g. grep the module source in the test) that `random` is never imported/used.

## Deliverables

- `src/peddler/credentials/passwords.py` — `generate_password`.
- `tests/credentials/test_passwords.py`

## Dependencies

None.
