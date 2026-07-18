# Description

## Problem

Applying to jobs online is repetitive: read a job description, tailor your CV's content to it, then manually transcribe the relevant bits into a web form — sometimes across several pages, sometimes behind a sign-up wall.
This project automates that transcription step end-to-end from inside Claude CLI.

## Inputs

- A CV in Markdown format (local file), the user's canonical CV content.
- A job description in Markdown format (local file, `jd.md`), passed per invocation.
- The URL of the application page for that specific role, passed per invocation.

## Trigger

A Claude CLI slash command:

```
/apply <jd.md> <url>
```

## Intended Architecture

A local MCP server, connected to Claude CLI, exposes a small set of tools that wrap a headless browser session:

- A tool to **open/navigate** to a URL and keep the browser session alive across subsequent tool calls (state persists between calls — the browser is not restarted per call).
- A tool to **fill a field** — given a field identifier and a text value, inserts it into the corresponding form field and reports back success, or any error surfaced by the page (e.g. "invalid phone number format") so the LLM can correct and retry.
- A tool to **advance the page** (submit / move to next step) once the relevant fields for the current page have been filled. Reports success of the specific errors found (e.g. "invalid phone number format" in field XYZ)
- A tool to **close the browser session** — ends the headless browser process. Used on success, and on abort after a stuck condition can't be resolved.
- Tools to **read/write a local credentials log book** (see Sign-up handling below) — these are the only things allowed to touch that file; the LLM never reads or writes it directly.

The LLM (Claude, driven by the CLI) is the orchestrator: it reads the JD and CV, decides what the form is asking for, decides what CV content (filtered/tailored through the lens of the JD) answers each field, and calls the tools in sequence. The browser-automation tools are dumb executors; the LLM does the planning.

## Behavior / Flow

1. Parse the JD and the CV.
2. Open the given URL in a headless browser session; keep it open for the duration of the application.
3. If it is a Log in/Sign up page, perform the appropriate action (log in if credentials are available, or sign up and store credentials -- see [[#Sign-up Handling]]), then advance to the actual form page.
4. Analyze the current page's content to identify the form fields being requested.
5. Build a plan mapping each field to a value drawn from the CV, filtered/tailored through the JD (e.g. which experience/skills to foreground).
6. Fill each field via the fill-field tool. If a field submission errors (e.g. bad format), the error is reported back to the LLM, which adjusts the value and retries.
7. Once the current page's fields are filled, advance to the next page/step. If a field submission errors (e.g. bad format), the error is reported back to the LLM, which adjusts the value and retries.
8. On the newly loaded page, the LLM judges from the page content whether it reads as a success/confirmation page (e.g. "Congrats, you successfully applied") or as another form page to fill in.
   - Success → stop the process and close the headless browser.
   - Another form → repeat from step 4.

## Sign-up Handling

Some application flows require creating an account before you can apply. When that's detected:

1. Try to find a contact email in the CV; if none is found, ask the user for one.
2. Check the local credentials log book (via a tool call) for an existing entry for this site.
   - **Found** → notify the user that stored credentials for this site are being reused, and proceed with those.
   - **Not found** → tell the user a new account is about to be created on this site with that email, generate a secure password, communicate it to the user, sign up, and store the resulting `(website, username, password)` triplet in the log book via a tool call.
3. The log book file itself is never read or written directly by the LLM — only through dedicated tool calls.
   (This is a security boundary to keep credentials out of the model's direct file access; the "minor detail" of hardening it further can come after the rough end-to-end system works.)

## Stuck Handling

When the agent hits something it can't resolve on its own (CAPTCHA, an ambiguous field, a validation error it can't figure out how to fix):

1. Pause the session and ask the user for guidance in the CLI.
2. If the issue still isn't resolved, abort the application attempt, report the failure and the reason, and suggest the user complete the application themselves — opening a normal (non-headless, visible) browser at the same URL for them.

## Success Detection

Purely LLM judgment: after each page load, the LLM reads the page content and informally decides whether it reads as an application-success/confirmation page or as another form step. No hardcoded pattern matching. Then it notifies the user of the succesful application and to check his email for followups from HR.

## Technical Constraints

- Python 3.14, managed with `uv`.
- Standard-library-only by default. Any third-party dependency requires explicit user approval before being added to `project/pyproject.toml`; if the user declines a proposed dependency, an alternative approach must be suggested instead.
- **Approved exception — browser automation**: Python's stdlib has no way to drive a headless browser. The user approved adding **Playwright** as a dependency for this. If Playwright turns out not to work for this use case, fall back to **Selenium** (known to work for the user) rather than reaching for a from-scratch CDP/WebSocket client.
- The MCP server itself does not need this exception — it's plain JSON-RPC over stdio and can be implemented with stdlib alone (`json`, `sys.stdin`/`sys.stdout`).

## Iteration: CLI Entrypoint & MCP Server Wiring (2026-07-18)

**Problem this closes:** every individual piece (credentials store, application log, browser session tools,
retry policy, the `/apply` command prompt, the JSON-RPC transport/registry/dispatch framework) was built and
unit-tested in isolation, but nothing ever assembles them into a runnable process, and nothing tells Claude Code
where to find it. Specifically: (a) there is no entry point — no `__main__`, no console script; (b) feature 4's
browser tools (`open_session`, `close_session`, `fill_field`, `fill_credential_field`, `advance_page`) were never
registered against a `ToolRegistry` at all, unlike the credentials and applog tools, which each got a
`register_*_tools(registry, ...)` wiring function — the browser tools also don't yet match the registry's
expected handler shape (`arguments: dict -> dict`); and (c) nothing documents how Claude Code discovers/launches
the server (no MCP registration mechanism, no `claude mcp add` instructions). Running `/apply` today would fail immediately —
the slash command exists, but there is no server on the other end of it.

**What this iteration adds:**

- **`peddler-mcp`** — a new console script that wires every feature's tools into one shared `ToolRegistry`
  (adding the missing `register_browser_tools(registry)` equivalent, with adapter handlers matching the
  `arguments: dict -> dict` shape the registry expects) and serves them over the existing stdio JSON-RPC
  `Transport`/`Server` framework (feature 1) — the actual MCP server process, never run directly by the user,
  only spawned by Claude Code once registered.
- **`peddler`** — a new console script, the user-facing launcher, built on `argparse`:
  - `--dir` (default: current working directory) — the folder to launch Claude Code in. This is the user's own
    job-search workspace, not Peddler's own source checkout.
  - `--credentials` (default: `~/.peddler/credentials.json`) — overrides the credentials log book path.
  - `--applog` (default: `~/.peddler/applications.log`) — overrides the application log path, for symmetry with
    `--credentials`. Both currently live as hardcoded constants (`DEFAULT_CREDENTIALS_PATH`,
    `DEFAULT_APPLOG_PATH`); this iteration makes them overridable, plumbed through to the `peddler-mcp` subprocess
    via environment variables passed through the MCP registration.
  - Everything after a `--` separator is forwarded verbatim to `claude` itself (e.g. `peddler -- --model opus`).
  - Behavior: checks Playwright's browser binaries are installed (fails fast with a fix-it message if not);
    ensures `peddler` is registered as an MCP server via `claude mcp add ... --scope local` (idempotent — treats
    "already registered" as success, never writes a `.mcp.json` file into the user's own `--dir`) with the
    resolved credentials/applog paths as env vars; then `exec`s (replaces the current process with) `claude` in
    `<dir>`.
- **Lifecycle, by design, not by extra code:** Claude Code itself spawns the MCP server as its own child process
  once registered, and kills it when Claude Code exits — this is how MCP over stdio already works, not something
  `peddler` has to implement. Because `peddler` `exec`s `claude` (replacing itself rather than spawning and
  babysitting a subprocess), closing the terminal sends the normal OS signal down the process chain
  (terminal → `claude` → its MCP-server child) and everything shuts down together for free, using ordinary
  process/signal semantics — no manual process-pairing/tracking code needed. `peddler` opens no new terminal
  window; it takes over the shell it was invoked from.

## Out of Scope (for now)

- CAPTCHA solving.
- Anti-bot / stealth evasion beyond what's needed for normal, honest form-filling.
- Hardening the credentials log book beyond "LLM never touches it directly via tool-mediated access" (e.g. encryption at rest) — noted as a future improvement, not a v1 requirement.
- Harness to write a valid cover letter when required (with no hallucinations, no exaggerations to match the JD, etc.)
- Cross-process file locking for the credentials log book / application log when multiple `peddler` sessions run
  simultaneously against the same default paths — mitigated for now by convention (one application at a time, or
  distinct `--credentials`/`--applog` paths per simultaneous session), not code.
