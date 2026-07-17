# 1. Apply Command

> Scope note: per `Description.md`'s architecture ("the LLM is the orchestrator ... the browser-automation tools
> are dumb executors"), almost all of the behavior this feature covers (deciding what a page is asking for,
> planning field values, judging success vs. another step, deciding when something is "stuck") happens through
> the LLM's own reasoning during the conversation — it is not deterministic code to write and unit-test. The only
> genuine code/deliverable surface for this feature is (a) the command definition that instructs the LLM through
> that protocol, and (b) one small deterministic helper worth extracting from LLM guesswork (see story 2, Email
> Extractor). Kept to two stories rather than inventing speculative modules for logic that actually lives in the
> prompt.

## What it does

Defines the `/apply <cv.md> <jd.md> <url>` Claude Code slash command. Its body is the complete orchestration
protocol instructing the LLM: read the CV and JD (via Claude Code's built-in file-reading, no dedicated Peddler
tool needed for this), call `open_session(url)`, detect and handle a login/sign-up page (using the email
extracted per story 2, `read_credentials`/`write_credentials`, and — when logging in with a stored credential —
`fill_credential_field` so the plaintext password never enters the LLM's context), loop over form pages —
identify requested fields, plan values from the CV filtered through the JD, `fill_field` each one, retry on a
field error, `advance_page` — judge success vs. another step from the resulting page content, Stuck Handling
(pause for text-only guidance, abort with a visible-browser suggestion if unresolved, never attempt to solve a
CAPTCHA itself), auto-retry up to 3 attempts with a short backoff on a tool-reported crash/timeout/network-drop
before treating it as stuck, `close_session()` on success or abort, and record the final outcome (via the
application log's recording tool, epic `2-application-log`) as one of `success` / `aborted` / `stuck-unresolved`.

## Inputs / Outputs

- Input: the command's `$ARGUMENTS`, positionally `<cv.md path> <jd.md path> <url>`.
- Output: no runtime return value — the command body is prompt text that shapes the LLM's subsequent tool-calling
  behavior for the rest of the conversation.

## Edge Cases and Failure Modes

- Missing or extra positional arguments (e.g. only two of the three paths given).
- `<cv.md>` or `<jd.md>` path doesn't exist or isn't readable — must surface as a clear error rather than
  silently proceeding with an empty CV/JD.
- `<url>` is malformed or unreachable.
- `/apply` invoked while another application attempt is already active in the same session — must refuse per the
  single-active-session concurrency rule (`Requirements.md → Non-Functional → Concurrency`).
- A tool call fails in a way not covered by the crash/timeout/network-drop auto-retry (e.g. a genuine validation
  error) — must not be swallowed by the retry loop; retries are only for the specific reliability class of
  errors named in `Requirements.md`.

## Acceptance Criteria

(checked by parsing the command file's content)

1. The command file declares the trigger `/apply <cv.md> <jd.md> <url>` and documents all three positional parts
   of `$ARGUMENTS`.
2. The body text references every tool by name: `open_session`, `fill_field`, `fill_credential_field`,
   `advance_page`, `close_session`, `read_credentials`, `write_credentials`, and the application log's recording
   tool.
3. The body text explicitly states generated sign-up passwords are never echoed into the conversation output —
   only written via `write_credentials` — and that logging in with a stored credential uses
   `fill_credential_field` (site-keyed, server-side password lookup) rather than ever obtaining the plaintext
   password via `read_credentials` and passing it to `fill_field`.
4. The body text explicitly states Stuck Handling keeps the browser headless (text-only guidance) and that a
   visible browser only ever appears as the post-abort fallback.
5. The body text explicitly states the 3-attempt auto-retry-with-backoff policy for crash/timeout/network-drop
   errors specifically (distinct from field-validation-error retries, which are unbounded LLM-driven corrections).
6. The body text explicitly states there is no hard cap on the number of form pages/steps traversed.
7. The body text lists the three outcome values the recording tool call must use: `success`, `aborted`,
   `stuck-unresolved`.

## Tasks

1. Write `src/peddler/commands/apply.md` — the command trigger line, `$ARGUMENTS` documentation, and the full
   orchestration protocol body covering every point in acceptance criteria 2-7.
2. Wire the command into the package so installing `peddler` registers it with Claude Code (packaging detail —
   consult how Claude Code discovers project-level slash commands from an installed package).
3. Write `tests/commands/test_apply_command.py` that loads the command file's content and asserts (via
   substring/regex checks) each acceptance-criterion item is present.

## Deliverables

- `src/peddler/commands/apply.md` — the command definition (packaged so installing `peddler` registers it with
  Claude Code).
- `tests/commands/test_apply_command.py` — loads the command file and asserts each acceptance-criterion item
  above is present (substring/regex checks against the file content).

## Dependencies

None (wave 1). Assumes the tools named above exist, by name and signature, per `Requirements.md` — their internal
implementations belong to epics `1-mcp-server-framework`, `2-browser-session-tools`, `2-credentials-log-book`,
and `2-application-log`.
