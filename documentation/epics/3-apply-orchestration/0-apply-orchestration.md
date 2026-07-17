# 3. Apply Orchestration

## Summary

The `/apply <cv.md> <jd.md> <url>` slash command itself and the end-to-end behavioral flow that ties every other
feature together: parsing the CV and JD, detecting and handling login/sign-up pages, looping over form pages
(identify fields → plan values from CV filtered through the JD → fill → advance → judge success vs. another
step), Stuck Handling (pause for text guidance, abort with a visible-browser suggestion if unresolved), success
notification, and recording the outcome to the application log. This is the "brain" that drives the tools built
by features 1–2; those features supply the actions, this feature supplies the decisions and the user-facing
conversation.

## Requirements Covered

- Requirement 1 — the `/apply <cv.md> <jd.md> <url>` slash command.
- Requirement 2 — loading and parsing the CV and JD.
- Requirement 4 — login/sign-up page detection ahead of field analysis.
- Requirement 5 — field identification and the CV/JD-driven fill plan.
- Requirement 6 — retrying a field after a validation error.
- Requirement 7 — advancing to the next step once a page's fields are filled.
- Requirement 8 — success vs. another-form-step judgment, and the resulting stop/continue behavior.
- Requirement 9 (remainder) — the sign-up decision flow (ask for email if none in the CV, notify the user of
  reuse vs. new sign-up).
- Requirement 10 — Stuck Handling: pause and ask, then abort with the visible-browser fallback if unresolved.
- Requirement 12 (remainder) — recording each attempt's outcome via the application log.
- The full `User Interaction Model` in `Requirements.md`.
- Non-functional: Concurrency — one active application at a time.

## Dependencies

- `2-browser-session-tools` — drives the browser via its tools.
- `2-credentials-log-book` — consults it during sign-up handling.
- `2-application-log` — records the outcome of each attempt.

## Stories

See `documentation/features/stories/3-apply-orchestration.md` (added by `/stories`).
