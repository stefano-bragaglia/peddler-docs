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
<!-- status: proposed | approved | branched | pr-open | done -->
| Feature | Branch | Status |
|---------|--------|--------|
| 1-mcp-server-framework | feature/1-mcp-server-framework | branched |
| 2-credentials-log-book | feature/2-credentials-log-book | branched |
| 3-application-log | feature/3-application-log | branched |
| 4-browser-session-tools | feature/4-browser-session-tools | branched |
| 5-apply-orchestration | feature/5-apply-orchestration | branched |

## Stories
| Feature | Story | Branch | Status |
|---------|-------|--------|--------|
<!-- status: proposed | approved | tests | code | pr-open | merged -->

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
- 2026-07-17: Features approved (originally as wave-numbered `1-mcp-server-framework` / `2-*` / `3-apply-orchestration`).
  All five `feature/<name>` branches created in `project/`. Also updated `.claude/commands/stories.md`
  (vault-local, untracked) to require a **Deliverables** section per story.
- 2026-07-17: Workflow simplified to **no parallelisation** (`project/` is a single local working directory with
  no room for multiple git worktrees, so forking Stage A/B risks two stories clobbering each other's files).
  Reworked feature/story numbering from dependency-wave (ties allowed) to strict unique sequential build order.
  Renumbered accordingly: `1-mcp-server-framework`, `2-credentials-log-book`, `3-application-log`,
  `4-browser-session-tools` (moved after credentials-log-book since its `fill_credential_field` story depends on
  credentials-log-book's storage), `5-apply-orchestration`. Story numbers within each feature were similarly
  de-tied (e.g. browser-session-tools' three tied "3"s became 3/4/5). `documentation/features/` (5 epics, 15
  stories) and `project/`'s branches renamed to match. `CLAUDE.md`'s *Parallelism Rules* replaced with *Sequential
  Execution Only*.
- 2026-07-17: Adopted a two-tier branching model (new `CLAUDE.md → Branching Model` section): epic branch
  `feature/<feature>` off `main` (unchanged); new **story branch** `story/<feature>/<story>` off the epic
  branch, created by `/stage-a` and used by both Stage A and Stage B (previously both committed straight onto
  the epic branch). `/pr` now has two tiers: `/pr <feature>/<story>` merges a story branch into its epic branch
  (gate: story `code`; on merge: story `merged`, branch deleted); `/pr <feature>` merges the epic branch into
  `main` (gate: every story `merged`, tightened from the old `code` gate; on merge: feature `done`). Added
  `pr-open` to both the Features and Stories status enums (it was already used by `/pr` but missing from the
  documented enum).

## Next Action
<!-- One sentence. What should happen next, and who does it (agent or user). -->
Run /stories review/approval, then /stage-a for each story in numbered order, starting with 1-mcp-server-framework/1-stdio-transport.