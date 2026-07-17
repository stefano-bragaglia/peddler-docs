# Notes

## Phase
<!-- onboarding | setup | requirements | features | stories | stage-a | stage-b | pr | publish | done -->
stories

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

## Features
<!-- status: proposed | approved | branched | done -->
| Feature | Branch | Status |
|---------|--------|--------|
| 1-mcp-server-framework | feature/1-mcp-server-framework | branched |
| 2-browser-session-tools | feature/2-browser-session-tools | branched |
| 2-credentials-log-book | feature/2-credentials-log-book | branched |
| 2-application-log | feature/2-application-log | branched |
| 3-apply-orchestration | feature/3-apply-orchestration | branched |

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
- 2026-07-17: Requirements.md complete, all 6 open questions answered: CV path is passed per invocation
  (`/apply <cv.md> <jd.md> <url>`); a separate persistent application log is kept alongside the credentials log
  book; Stuck Handling keeps the browser headless (text-only guidance), only going visible after an unresolved
  abort; generated passwords are never echoed into the CLI transcript, only written to the credentials log book;
  crashes/timeouts/network drops auto-retry up to 3 attempts with backoff before falling back to Stuck Handling;
  no hard cap on form-page/step count, relying on LLM judgment instead.
- 2026-07-17: Features approved: `1-mcp-server-framework` (wave 1); `2-browser-session-tools`,
  `2-credentials-log-book`, `2-application-log` (wave 2, independent, parallelisable); `3-apply-orchestration`
  (wave 3, depends on all of wave 2). All five `feature/<name>` branches created in `project/`. Also updated
  `.claude/commands/stories.md` (vault-local, untracked) to require a **Deliverables** section per story.

## Next Action
<!-- One sentence. What should happen next, and who does it (agent or user). -->
Run /stories <feature> for each feature (can be parallelised) — start with 1-mcp-server-framework.