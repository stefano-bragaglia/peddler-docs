# Notes

## Phase
<!-- onboarding | setup | requirements | features | stories | stage-a | stage-b | pr | publish | done -->
requirements

## Project
name: peddler
python: 3.14

## Repos
<!-- Filled in during /setup -->
docs (vault root): https://github.com/stefano-bragaglia/peddler-docs, private
code (project/): https://github.com/stefano-bragaglia/peddler, public

## Publish
<!-- Filled in during /publish. Keeps the step resumable after /clear. -->
last published version:
status: <!-- unpublished | built | published | failed -->

## Open Questions
<!-- Add a question when blocked. Clear (delete the line) when answered. -->
<!-- Format: - [ ] Q: <question> / A: <answer> -->
- [ ] Q: Where does the canonical CV Markdown file live — fixed configured-once path, or passed per invocation?
- [ ] Q: Should Peddler keep a persistent application log (site, timestamp, outcome) separate from the credentials log book?
- [ ] Q: Stuck Handling — does the browser stay headless (text-only guidance) or switch to visible so the user can intervene directly (e.g. CAPTCHA)?
- [ ] Q: Is printing a generated sign-up password in plaintext CLI output acceptable, or should it only go to the credentials log book?
- [ ] Q: Crash/timeout/network-drop mid-application — auto-retry, or hand off to Stuck Handling?
- [ ] Q: Should there be a hard cap on the number of form pages/steps traversed before giving up?

## Features
<!-- status: proposed | approved | branched | done -->

## Stories
| Feature | Story | Branch | Status |
|---------|-------|--------|--------|
<!-- status: proposed | approved | tests | code | merged -->

## Decisions
<!-- Key choices made and why. Future agents use this to avoid re-litigating. -->
- 2026-07-17: Description.md approved (user-edited to final form). Stack: Python 3.14 + uv, stdlib-only by
  default, third-party deps require explicit user approval. Approved exception: Playwright for headless browser
  automation, with Selenium as agreed fallback if Playwright doesn't work out. MCP server itself built on stdlib
  (stdio JSON-RPC), no exception needed there.
- 2026-07-17: Project named **Peddler** (mascot: chibi bike courier carrying a "CV" box) — picked over
  Usher/Bellhop/Concierge/Rider/Runner/Cadet for the door-to-door pun on "pedal" without mail/font baggage
  (Courier) or gig-economy connotation (Rider). Logo stored at `project/docs/logo.png` and vault-root `logo.png`.
- 2026-07-17: `/setup` complete. Vault repo `peddler-docs` (private) and code repo `peddler` (public) created and
  pushed. `project/` scaffolded as a `src/`-layout uv package (Python 3.14, hatchling build backend) with
  ruff/radon/pytest gates wired into a pre-commit hook and GitHub Actions CI (pytest exit code 5 — no tests
  collected — tolerated pre-stage-a, not treated as a gate failure).

## Next Action
<!-- One sentence. What should happen next, and who does it (agent or user). -->
Run /requirements to derive documentation/Requirements.md from Description.md.