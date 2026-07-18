# Requirements

## Functional Requirements

1. The system provides a Claude CLI slash command `/apply <cv.md> <jd.md> <url>` that starts one application
   attempt for one role at one URL. The CV path is passed explicitly on every invocation (not a fixed, pre-
   configured location), so the user can point at different CVs across invocations.
2. The system loads and parses the CV at `<cv.md>` (Markdown) and the job description at `<jd.md>` (Markdown)
   before starting the browser session.
3. The system exposes an MCP server to Claude CLI with the following tools, all operating on a single browser
   session kept alive across calls within one `/apply` invocation:
   - `open_session(url)` — launches a headless browser and navigates to `url`.
   - `fill_field(field_id, value)` — sets a form field's value; returns success or any validation error the page
     surfaces for that field.
   - `advance_page()` — submits the current page/step; returns success (with the resulting page's content) or any
     field-specific errors the page surfaces (e.g. "invalid phone number" on a named field).
   - `close_session()` — ends the headless browser process. Called on success and on abort.
   - `read_credentials(site)` / `write_credentials(site, username, password)` — the only way to read or write the
     local credentials log book; the LLM never accesses that file directly.
4. On session start, the system detects whether the loaded page is a login/sign-up page (as opposed to the
   application form itself) and handles it before proceeding to field analysis (see Sign-up Handling below).
5. For each form page encountered, the system (via the LLM, using the page content returned by the tools)
   identifies the requested fields, builds a plan mapping each field to CV content filtered/tailored through the
   JD, and fills each field via `fill_field`.
6. If a field submission or page advance returns an error, the system adjusts the corresponding value and retries
   before moving on.
7. Once a page's fields are filled without outstanding errors, the system calls `advance_page()` to move to the
   next step.
8. After each `advance_page()` call, the system evaluates the resulting page content (LLM judgment, no hardcoded
   pattern matching) to decide between:
   - **Success** — the page reads as an application confirmation. The system notifies the user of the successful
     application and to check their email for follow-ups from HR, then calls `close_session()` and stops.
   - **Another form step** — the system repeats from requirement 5 for the new page. There is no hard cap on the
     number of pages/steps traversed; the system relies on the LLM's own judgment to recognize a flow that isn't
     converging (surfacing it as a Stuck Handling condition) rather than a fixed step counter.
9. **Sign-up Handling**: when a login/sign-up page is detected:
   - The system looks for a contact email in the CV; if none is found, it asks the user for one.
   - It calls `read_credentials(site)`. If an entry exists, it notifies the user that stored credentials are being
     reused and logs in with them.
   - If no entry exists, it notifies the user that a new account is about to be created with that email, generates
     a secure password (stdlib `secrets` module), performs the sign-up, and calls
     `write_credentials(site, username, password)` to store the new entry. The password itself is never printed
     in the CLI conversation output — it exists only in the credentials log book.
10. **Stuck Handling**: when the system encounters something it cannot resolve on its own (CAPTCHA, an ambiguous
    field, an unresolvable validation error), it pauses and asks the user for guidance in the CLI. The browser
    remains headless during this pause — guidance from the user is text-only (e.g. "try MM/YYYY format"); there
    is no direct visual/interactive hand-off to the user at this stage. If the user's guidance doesn't resolve
    it, the system aborts the application, reports the failure and its reason, calls `close_session()`, and
    suggests the user complete the application themselves by opening a normal (visible, non-headless) browser at
    the same URL — this is the only point at which a visible browser window appears.
11. The credentials log book is a local file, read and written exclusively through the `read_credentials` /
    `write_credentials` tools — never directly by the LLM.
12. The system maintains a separate, persistent local application log — one entry per `/apply` attempt, recording
    the URL, a timestamp, and the outcome (success / aborted / stuck-unresolved) — giving the user a history of
    applications and a basis for future duplicate-application checks.
13. The system provides a `peddler` console-script CLI (`argparse`-based) that launches Claude Code with the
    Peddler MCP server registered. It accepts `--dir` (default: current working directory — the folder to launch
    Claude Code in, the user's own job-search workspace, not Peddler's own source checkout), `--credentials`
    (default `~/.peddler/credentials.json`), `--applog` (default `~/.peddler/applications.log`), and forwards
    everything after a `--` separator verbatim to `claude` itself (e.g. `peddler -- --model opus`).
14. The system provides a `peddler-mcp` console-script entry point: the actual MCP server process, wiring every
    feature's tools (credentials, application log, and browser session tools — the last of which are not
    currently registered against any `ToolRegistry` at all) into one shared registry and serving them over the
    existing stdio JSON-RPC `Transport`/`Server` framework. This is spawned only by Claude Code, per its MCP
    server configuration — never invoked directly by the user.
15. `peddler` ensures `peddler` is registered as an MCP server via `claude mcp add ... --scope local`, with the
    resolved `--credentials`/`--applog` paths passed through as environment variables — never a `.mcp.json` file
    written into `--dir` itself (that's the user's own workspace, not Peddler's, and a `.mcp.json` living there
    risks getting accidentally committed if it happens to be a git repo). This is idempotent: if `peddler` is
    already registered at local scope (a prior invocation), treat that as success rather than re-adding or
    erroring; any other failure from `claude mcp add` itself (e.g. `claude` not on `PATH`) fails loudly with the
    underlying error.
16. `peddler` launches Claude Code by replacing its own process (`exec`, not spawning a child it has to track),
    so that closing the terminal / ending the Claude Code session tears down the MCP server subprocess too via
    ordinary OS process/signal propagation — not custom lifecycle-tracking code. `peddler` never opens a new
    terminal window; it takes over the shell it was invoked from.
17. Before registering the MCP server and launching Claude Code, `peddler` checks that Playwright's browser
    binaries are installed and fails fast with a clear fix-it message (e.g. "run `uv run playwright install
    chromium`") if not — surfacing a missing-browser problem immediately at launch rather than several turns into
    a conversation, the first time `/apply` calls `open_session`.

## Non-Functional Requirements

- **Language/runtime**: Python 3.14, managed with `uv`.
- **Dependencies**: standard-library-only by default. Any new third-party dependency requires explicit user
  approval before being added to `project/pyproject.toml`; if declined, an alternative must be proposed instead.
  Approved exception: **Playwright** for headless browser automation, with **Selenium** as the agreed fallback if
  Playwright doesn't work out. The MCP server itself is implemented with stdlib alone (JSON-RPC over stdio).
- **Security**: the LLM never has direct file access to the credentials log book — only the two dedicated tools
  do. Generated passwords use a cryptographically secure source (stdlib `secrets`) and are never echoed into the
  CLI conversation output — they exist only in the log book. Encrypting the log book at rest is explicitly
  deferred (not a v1 requirement — see `Description.md → Out of Scope`).
- **Reliability**: browser automation must surface page/tool errors back to the LLM rather than crashing the
  process outright, so the LLM can retry or hand off to Stuck Handling. On a browser crash, timeout, or network
  drop specifically, the system auto-retries the failed operation up to 3 attempts with a short backoff before
  falling back to Stuck Handling.
- **Scope boundaries** (carried over from `Description.md`): no CAPTCHA solving, no anti-bot/stealth evasion
  beyond normal honest form-filling, no cover-letter generation.
- **Concurrency**: v1 supports one active application session (one open browser) at a time per `/apply`
  invocation; concurrent `/apply` runs are out of scope for now. `peddler` makes running multiple simultaneous
  sessions (different `--dir`s) easier to reach for, but neither `CredentialStore.put()` nor
  `ApplicationLog.append()` has file locking — for now this is mitigated by convention, not code: run one
  application at a time, or point simultaneous sessions at distinct `--credentials`/`--applog` paths. Real file
  locking is deferred (see `Description.md → Out of Scope`).
- **Packaging**: `peddler` and `peddler-mcp` are registered as `[project.scripts]` entry points in
  `project/pyproject.toml`, so installing the package (`uv tool install peddler` or equivalent) puts both on
  `PATH`.
- **Platform**: the launcher's `exec`-based process replacement (`os.execvp`) is a POSIX facility — macOS/Linux
  only. Windows support is out of scope unless requested (matches the development machine).

## User Interaction Model

The user interacts entirely through Claude CLI, in a single conversational session per application attempt.

```
> /apply ~/cv.md ~/jobs/acme-backend-swe/jd.md https://acme.example.com/careers/apply/1234

Peddler: Reading job description and your CV...
Peddler: Opening https://acme.example.com/careers/apply/1234 ...
Peddler: This looks like a sign-up page. No stored credentials found for acme.example.com.
         I'll create an account using <email from CV>. A secure password has been generated
         and saved to the credentials log book (not shown here).
Peddler: Signed up, now on the application form (step 1 of 3).
Peddler: Filling: Full name, Email, Phone, Years of experience, Cover letter summary...
Peddler: Step 1 submitted. Now on step 2 of 3 (skills & experience)...
...
Peddler: Step 3 submitted. This page reads as a success confirmation:
         "Thank you for applying to Acme Corp!"
Peddler: 🎉 Application submitted successfully. Check your email for follow-ups from HR.
```

If something goes wrong:

```
Peddler: Step 2 rejected the "Phone" field: "invalid phone number format". Retrying with a reformatted value...
Peddler: Still stuck — this page shows a CAPTCHA I can't solve. Can you help?
> [user provides guidance, or none is available]
Peddler: Unable to proceed. Aborting this application.
         Reason: CAPTCHA challenge on step 2, unresolved.
         Suggest applying manually — opening a visible browser at the URL for you now.
```

Before any of that, the user starts a session with the `peddler` launcher from their own terminal:

```
$ peddler --dir ~/job-search --credentials ~/job-search/.peddler/credentials.json
[peddler] checking playwright browser binaries... ok
[peddler] registered MCP server (local scope) for /Users/you/job-search
[peddler] starting claude in /Users/you/job-search ...

> /apply ~/cv.md ~/jobs/acme-backend-swe/jd.md https://acme.example.com/careers/apply/1234
...
```

Closing that terminal (or exiting Claude Code normally) ends the whole session, MCP server included.

## Questions: Requirements

1. *Q: Where does the canonical CV Markdown file live — a fixed, configured-once path (e.g. `~/.peddler/cv.md`),
   or is it passed explicitly per invocation (similar to `jd.md`)?*
   _A: Passed explicitly per invocation, like `jd.md`. The trigger becomes `/apply <cv.md> <jd.md> <url>`._
2. *Q: Should Peddler keep a persistent application log (site, timestamp, outcome: success/aborted) separate from
   the credentials log book, so a user can see application history / avoid re-applying to the same URL?*
   _A: Yes — keep a separate local application log recording every `/apply` attempt (URL, timestamp, outcome)._

## Questions: Interaction

1. *Q: During Stuck Handling, does the browser stay headless while the system asks the user for guidance (user
   can only respond with text, e.g. "try MM/YYYY format"), or does it switch to a visible window at that point so
   the user can see/interact with the actual page directly — which would be necessary for something like solving
   a CAPTCHA themselves?*
   _A: Stays headless — guidance is text-only during the pause. The browser only ever becomes visible in the
   fallback path already described (a fresh, visible browser opened for the user after an unresolved abort), not
   during the pause-and-ask step itself. This means a CAPTCHA can't actually be solved during the pause — it
   will typically end up as an unresolved abort that falls through to the visible-browser fallback._
2. *Q: Is it acceptable for a generated sign-up password to be printed in plaintext in the CLI conversation
   output, or should it instead be written only to the credentials log book and never echoed into the chat
   transcript?*
   _A: Never echo it. The password is written straight to the credentials log book via `write_credentials` and
   is not printed in the chat transcript, since that transcript may be logged/persisted elsewhere less securely
   than the log book itself._

## Questions: Edge Cases

1. *Q: What should happen if the browser/page crashes, times out, or the network drops mid-application — treat
   it as an automatic retry, or immediately hand off to Stuck Handling (pause and ask the user)?*
   _A: Auto-retry first. On a crash/timeout/network-drop, the system retries the failed operation (re-navigate /
   reopen the session as needed) up to 3 attempts with a short backoff between them; only after retries are
   exhausted does it hand off to Stuck Handling. (3 attempts is a starting default — adjust in `/features` or
   `/stories` if it proves too aggressive or too lax in practice.)_
2. *Q: Should there be a hard cap on the number of form pages/steps the system will traverse before giving up
   (to guard against an infinite loop if success detection never triggers), and if so, roughly how many steps?*
   _A: No hard cap. The system relies on the LLM's own judgment across steps rather than a fixed page limit;
   if the flow genuinely seems to be going nowhere, that's expected to surface as a Stuck Handling condition
   through the LLM's own reasoning rather than a hardcoded counter._

## Questions: CLI Entrypoint

1. *Q: Naming — is `peddler` (launcher) / `peddler-mcp` (server) the right pair of console-script names, or would
   you prefer something else (e.g. to make the "don't run this one yourself" distinction more obvious)?*
   _A: Keep `peddler` / `peddler-mcp`._
2. *Q: Where should `peddler` register the MCP server — a `.mcp.json` file written into `--dir` (visible,
   possibly tracked if that directory happens to be its own git repo), or Claude Code's user/local-scope config
   (e.g. via `claude mcp add ... --scope local`, invisible inside `--dir`)? This matters because `--dir` is the
   user's own workspace, not Peddler's — an unwanted tracked file there is a real annoyance, not just cosmetic.*
   _A: `claude mcp add ... --scope local` — no file written into the user's own workspace at all._
3. *Q: If `--dir` already has an MCP server config with something unexpected in it (malformed JSON, a `peddler`
   entry already pointing somewhere else) — fail loudly and ask the user to fix it, or overwrite it?*
   _A: Since registration is now via `claude mcp add --scope local` (per Q2), the only realistic collision is
   `peddler` already being registered locally from a prior invocation — treat that as success (idempotent, don't
   re-add or error), by checking `claude mcp list` (or equivalent) first. Any other genuine failure from
   `claude mcp add` itself (e.g. `claude` not on `PATH`) fails loudly with the underlying error, never silently
   swallowed._
4. *Q: Should `peddler` check that Playwright's browser binaries are actually installed (`playwright install
   chromium`) before handing off to Claude Code, so a missing-browser failure surfaces immediately at launch with
   a clear message — rather than several turns into a conversation, the first time `/apply` calls
   `open_session`?*
   _A: Yes — check and fail fast with a clear fix-it message (e.g. "run `uv run playwright install chromium`")
   before launching Claude Code._
5. *Q: The existing Concurrency non-functional requirement assumes one active `/apply` at a time — framed around
   a single invocation. `peddler` makes running multiple simultaneous sessions (different `--dir`s, different
   terminals) easier to stumble into. If two sessions share the same `--credentials`/`--applog` path (the
   defaults, unless each is given a distinct override) and both happen to write at once, `CredentialStore.put()`
   and `ApplicationLog.append()` have no file locking (explicitly out of scope for v1, per the existing
   Concurrency requirement). Is relying on the user to point separate sessions at separate `--credentials`/
   `--applog` paths an acceptable mitigation, or does this need real cross-process locking now?*
   _A: For now, rely on the user to either use separate `--credentials`/`--applog` paths per simultaneous session,
   or apply to one job at a time. Real file locking is deferred — recorded in `Description.md → Out of Scope` as
   a future improvement, not a requirement for this iteration._
