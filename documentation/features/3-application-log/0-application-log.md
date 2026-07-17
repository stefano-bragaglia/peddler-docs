# 3. Application Log

## Summary

A separate, persistent local log of every `/apply` attempt — URL, timestamp, and outcome (success / aborted /
stuck-unresolved). Distinct from the credentials log book: this is a plain history/audit trail with no
security-sensitive contents, giving the user visibility into what they've applied to and laying the groundwork
for future duplicate-application checks.

## Requirements Covered

- Requirement 12 — the persistent application log and its recorded fields.

## Dependencies

- `1-mcp-server-framework` — the recording action is exposed as a tool (or invoked via the framework's tool
  dispatch) so the orchestration logic in feature 5 can call it the same way it calls other tools.

## Stories

See the numbered story files alongside this one in `documentation/features/3-application-log/`.
