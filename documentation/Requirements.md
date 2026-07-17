# Requirements

## Functional Requirements

1. The system provides a Claude CLI slash command `/apply <jd.md> <url>` that starts one application attempt for
   one role at one URL.
2. The system loads and parses the user's canonical CV (Markdown) and the job description at `<jd.md>` (Markdown)
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
   - **Another form step** — the system repeats from requirement 5 for the new page.
9. **Sign-up Handling**: when a login/sign-up page is detected:
   - The system looks for a contact email in the CV; if none is found, it asks the user for one.
   - It calls `read_credentials(site)`. If an entry exists, it notifies the user that stored credentials are being
     reused and logs in with them.
   - If no entry exists, it notifies the user that a new account is about to be created with that email, generates
     a secure password (stdlib `secrets` module), communicates the password to the user, performs the sign-up,
     and calls `write_credentials(site, username, password)` to store the new entry.
10. **Stuck Handling**: when the system encounters something it cannot resolve on its own (CAPTCHA, an ambiguous
    field, an unresolvable validation error), it pauses and asks the user for guidance in the CLI. If the user's
    guidance doesn't resolve it, the system aborts the application, reports the failure and its reason, calls
    `close_session()`, and suggests the user complete the application themselves by opening a normal (visible,
    non-headless) browser at the same URL.
11. The credentials log book is a local file, read and written exclusively through the `read_credentials` /
    `write_credentials` tools — never directly by the LLM.

## Non-Functional Requirements

- **Language/runtime**: Python 3.14, managed with `uv`.
- **Dependencies**: standard-library-only by default. Any new third-party dependency requires explicit user
  approval before being added to `project/pyproject.toml`; if declined, an alternative must be proposed instead.
  Approved exception: **Playwright** for headless browser automation, with **Selenium** as the agreed fallback if
  Playwright doesn't work out. The MCP server itself is implemented with stdlib alone (JSON-RPC over stdio).
- **Security**: the LLM never has direct file access to the credentials log book — only the two dedicated tools
  do. Generated passwords use a cryptographically secure source (stdlib `secrets`). Encrypting the log book at
  rest is explicitly deferred (not a v1 requirement — see `Description.md → Out of Scope`).
- **Reliability**: browser automation must surface page/tool errors back to the LLM rather than crashing the
  process outright, so the LLM can retry or hand off to Stuck Handling.
- **Scope boundaries** (carried over from `Description.md`): no CAPTCHA solving, no anti-bot/stealth evasion
  beyond normal honest form-filling, no cover-letter generation.
- **Concurrency**: v1 supports one active application session (one open browser) at a time per `/apply`
  invocation; concurrent `/apply` runs are out of scope for now.

## User Interaction Model

The user interacts entirely through Claude CLI, in a single conversational session per application attempt.

```
> /apply ~/jobs/acme-backend-swe/jd.md https://acme.example.com/careers/apply/1234

Peddler: Reading job description and your CV...
Peddler: Opening https://acme.example.com/careers/apply/1234 ...
Peddler: This looks like a sign-up page. No stored credentials found for acme.example.com.
         I'll create an account using <email from CV>. Generated password: ****************
         (also saved to the credentials log book)
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

## Questions: Requirements

1. *Q: Where does the canonical CV Markdown file live — a fixed, configured-once path (e.g. `~/.peddler/cv.md`),
   or is it passed explicitly per invocation (similar to `jd.md`)?*
2. *Q: Should Peddler keep a persistent application log (site, timestamp, outcome: success/aborted) separate from
   the credentials log book, so a user can see application history / avoid re-applying to the same URL?*

## Questions: Interaction

1. *Q: During Stuck Handling, does the browser stay headless while the system asks the user for guidance (user
   can only respond with text, e.g. "try MM/YYYY format"), or does it switch to a visible window at that point so
   the user can see/interact with the actual page directly — which would be necessary for something like solving
   a CAPTCHA themselves?*
2. *Q: Is it acceptable for a generated sign-up password to be printed in plaintext in the CLI conversation
   output, or should it instead be written only to the credentials log book and never echoed into the chat
   transcript?*

## Questions: Edge Cases

1. *Q: What should happen if the browser/page crashes, times out, or the network drops mid-application — treat
   it as an automatic retry, or immediately hand off to Stuck Handling (pause and ask the user)?*
2. *Q: Should there be a hard cap on the number of form pages/steps the system will traverse before giving up
   (to guard against an infinite loop if success detection never triggers), and if so, roughly how many steps?*
