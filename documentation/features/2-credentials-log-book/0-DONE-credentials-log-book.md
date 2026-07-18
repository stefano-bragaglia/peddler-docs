# 2. DONE - Credentials Log Book

## Summary

The tool-mediated local store for per-site sign-up credentials. Exposes `read_credentials(site)` and
`write_credentials(site, username, password)` as the *only* way anything touches the underlying log book file —
the LLM never reads or writes it directly, keeping generated passwords out of the model's direct file access and
out of the CLI conversation transcript. Also owns secure password generation (stdlib `secrets`) for new sign-ups.

## Requirements Covered

- Requirement 3 — the `read_credentials(site)` / `write_credentials(site, username, password)` tools.
- Requirement 9 (partial) — the credential lookup/storage half of Sign-up Handling (the decision to sign up vs.
  reuse, and the actual sign-up form interaction, belong to feature 5).
- Requirement 11 — the credentials log book file itself, and its tool-only access boundary.
- Non-functional: Security — cryptographically secure password generation; passwords never echoed into the CLI
  conversation output (`Requirements.md → Non-Functional → Security`).

## Dependencies

- `1-mcp-server-framework` — tools are registered against that server.

## Stories

See the numbered story files alongside this one in `documentation/features/2-credentials-log-book/`.
