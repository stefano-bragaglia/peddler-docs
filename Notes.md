# Notes

## Phase
<!-- onboarding | setup | requirements | features | stories | stage-a | stage-b | pr | publish | done -->
stage-a

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
<!-- status: proposed | approved | branched | stories-merged | pr-open | done -->
<!-- stories-merged: every story for this feature is `merged`, but the epic PR into main isn't open yet -->
| Feature | Branch | Status |
|---------|--------|--------|
| 1-mcp-server-framework | feature/1-mcp-server-framework | done |
| 2-credentials-log-book | feature/2-credentials-log-book | branched |
| 3-application-log | | approved |
| 4-browser-session-tools | | approved |
| 5-apply-orchestration | | approved |

## Stories
| Feature | Story | Branch | Status |
|---------|-------|--------|--------|
<!-- status: proposed | approved | tests | code | pr-open | merged -->
| 1-mcp-server-framework | 1-stdio-transport | story/1-mcp-server-framework/1-stdio-transport (deleted, merged) | merged |
| 1-mcp-server-framework | 2-tool-registry | story/1-mcp-server-framework/2-tool-registry (deleted, merged) | merged |
| 1-mcp-server-framework | 3-request-dispatch-and-lifecycle | story/1-mcp-server-framework/3-request-dispatch-and-lifecycle (deleted, merged) | merged |
| 2-credentials-log-book | 1-log-book-storage | story/2-credentials-log-book/1-log-book-storage (deleted, merged) | merged |
| 2-credentials-log-book | 2-password-generator | story/2-credentials-log-book/2-password-generator (deleted, merged) | merged |
| 2-credentials-log-book | 3-credential-tools | story/2-credentials-log-book/3-credential-tools | tests |
| 3-application-log | 1-record-application | | approved |
| 3-application-log | 2-query-application-history | | approved |
| 4-browser-session-tools | 1-retry-policy | | approved |
| 4-browser-session-tools | 2-session-lifecycle | | approved |
| 4-browser-session-tools | 3-fill-field | | approved |
| 4-browser-session-tools | 4-advance-page | | approved |
| 4-browser-session-tools | 5-fill-credential-field | | approved |
| 5-apply-orchestration | 1-apply-command | | approved |
| 5-apply-orchestration | 2-email-extractor | | approved |

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
- 2026-07-17: All 15 stories approved as drafted. Fixed `.claude/commands/stories.md` (vault-local, untracked)
  to match the two-tier branching model: step 2's now-pointless `project/` checkout removed (stories are
  documentation-only, and checking out `feature/<feature>` there could have disrupted an in-progress story
  branch elsewhere); step 5 no longer assigns `feature/<feature>` as a story's branch — that field stays blank
  until `/stage-a` creates the story's own `story/<feature>/<story>` branch.
- 2026-07-17: Fixed `project/.github/workflows/ci.yml`, which predated the two-tier branching model: its
  `pull_request` trigger only matched `branches: [main]`, so CI never ran on story-tier PRs (into
  `feature/<epic>`) — confirmed dead via `gh pr checks` on PR #1 showing no checks. Added `feature/**` to the
  `pull_request` branches filter, plus a `push` trigger on `story/**` so failures surface while iterating on
  Stage A/B, before a PR even opens. Committed directly onto the in-flight story branch
  `story/1-mcp-server-framework/1-stdio-transport` (unrelated to that story's own code, but the smallest way to
  verify the fix live and have it land in the epic branch on merge) rather than opening a separate infra PR.
- 2026-07-17: Added a `stories-merged` status to the Features enum (`CLAUDE.md`, `templates/Notes.template.md`,
  `.claude/commands/pr.md`), between `branched` and `pr-open`. Previously `branched` covered both "stories still
  in progress" and "every story merged, epic PR not opened yet" — indistinguishable without manually
  cross-checking every story row. `/pr <feature>/<story>`'s on-merge step now sets the feature to
  `stories-merged` once its last story merges; `/pr <feature>`'s epic-tier step 2 now gates on that status
  directly instead of re-deriving it from the Stories table each time.
- 2026-07-17: Applied branch protection to `peddler`'s (code repo) `main`: require PR before merge (0 required
  approvals — solo project), require the `test` status check up to date, block force-push/deletion,
  `enforce_admins: false`. `peddler-docs` (vault repo, private) cannot get any equivalent protection — both
  classic branch protection and rulesets return 403 for private repos on the free GitHub plan; accepted as a
  known limitation (fix would mean upgrading to Pro or making the vault repo public). Updated
  `.claude/commands/setup.md` (vault-local, gitignored) so future `/setup` runs configure both this branch
  protection and the corrected two-tier CI trigger (`pull_request` on `main`+`feature/**`, `push` on
  `story/**`) up front, instead of needing a manual fix later like this one did.
- 2026-07-17: PR #2 (`feature/1-mcp-server-framework` → `main`, merged via GitHub UI) landed one story early —
  only `1-stdio-transport` was `merged` at the time; `2-tool-registry` and `3-request-dispatch-and-lifecycle`
  were still `approved`. This jumped `/pr <feature>`'s gate (all stories `merged`). No technical harm: the
  epic branch's tip is an ancestor of `main`'s new tip, so finishing stories 2/3 on the same epic branch and
  opening a later PR into `main` will diff cleanly, showing only their new work. Feature status stays
  `branched` (not `done`) until that real final PR merges — `main` having story 1's code early doesn't count
  as the feature being done. No revert applied; decided to just continue forward.
- 2026-07-17: Found and fixed a systemic gap: no command (or `CLAUDE.md`) ever instructed committing the
  **vault-root** repo itself after updating `Notes.md`/`documentation/` — only `project/` commits were
  specified anywhere. This is why feature 1's `Notes.md` status updates and `DONE -` markers sat uncommitted
  through several subsequent commands until caught and committed by hand. Added a general rule to
  `CLAUDE.md → Notes.md Contract` (every command's final action, when it edits `Notes.md`/`documentation/`,
  is to commit those changes to the vault repo — separate from any `project/` commit), and added an explicit
  commit step to every command file that touches `Notes.md`/`documentation/`: `describe`, `requirements`,
  `features`, `stories`, `stage-a`, `stage-b`, `pr` (both story- and epic-tier, including the on-merge steps),
  `setup`, `publish`.

- 2026-07-17: `/pr 2-credentials-log-book/1-log-book-storage` found `feature/2-credentials-log-book` had never been pushed to origin (created locally back at `/features`, before any story existed to push it) — `gh pr create` failed with "Base ref must be a branch" until it was pushed. Pushed it before opening the story PR. Same is likely true of the other three not-yet-started epic branches (`3-application-log`, `4-browser-session-tools`, `5-apply-orchestration`); their first story PR will hit the same thing and should just push the epic branch first, same as here — not a bug, just a step worth expecting.

- 2026-07-17: Diagnosed why CI never ran on PR #6 (`story/.../1-log-book-storage` → `feature/2-credentials-log-book`): `feature/2-credentials-log-book` (and the other three not-yet-started epic branches) had been branched off `main` at `/features` approval time, *before* commit `7c309e8` (the fix making CI trigger on epic/story-tier PRs, not just PRs into `main`) later landed on `main` via feature 1's merge — so those four branches were still running the pre-fix, `main`-only-trigger workflow. Root-caused as a process bug, not a one-off: branching every approved feature immediately, regardless of when work on it actually starts, means any of them can go stale relative to later `main` changes.
  - **Fix (harness, permanent):** `CLAUDE.md → Branching Model`, `.claude/commands/features.md`, and `.claude/commands/stage-a.md` changed so epic branches are created **lazily** — by `/stage-a`, off current `main`, the first time work starts on any of that feature's stories — instead of in bulk by `/features` at approval time. `/features` now only sets feature status to `approved`; `/stage-a` creates+pushes `feature/<feature>` on demand and flips status to `branched`. `/setup` already configures CI trigger + branch protection on `main` once, permanently — lazy branching is what makes every future epic branch actually inherit that current state instead of freezing whatever `main` looked like at approval time.
  - **Remediation (this project, mid-way through):** deleted the three stale, never-started, never-pushed local epic branches (`feature/3-application-log`, `feature/4-browser-session-tools`, `feature/5-apply-orchestration`) — no work existed on them, so nothing lost; `/stage-a` will recreate them off current `main` when each is actually started. `feature/2-credentials-log-book` already has in-flight work (PR #6 open), so instead merged current `main` (fast-forward) into it, then merged that into `story/2-credentials-log-book/1-log-book-storage` and pushed both — confirmed fixed: PR #6's `test` check now runs and passes (previously reported no checks at all).

- 2026-07-17: Found the vault-root repo (docs repo, `peddler-docs`) had never been pushed — 12 commits sat local-only despite `CLAUDE.md`'s Notes.md Contract requiring every command to commit `Notes.md`/`documentation/` changes there. Every command's "commit to the vault-root repo" instruction never said to push, so it never happened. Fixed the harness (`CLAUDE.md → Notes.md Contract`, and the commit step in every command that touches `Notes.md`/`documentation/`: `describe`, `requirements`, `features`, `stories`, `stage-a`, `stage-b`, `pr` — all four commit points, `setup`, `publish`) to require `git push` immediately after every such commit, no exceptions. Pushed the 12 pending commits to sync `origin/main` on `peddler-docs` now.

## Next Action
<!-- One sentence. What should happen next, and who does it (agent or user). -->
Run /stage-b 2-credentials-log-book/3-credential-tools (tests written on story/2-credentials-log-book/3-credential-tools; production code not yet written). Last story for this feature — once merged, feature becomes stories-merged and ready for /pr 2-credentials-log-book.