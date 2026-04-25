# litellm #26517 — non-admin MCP self-service (disconnect, approval filter, BYOK admin guard)

- URL: https://github.com/BerriAI/litellm/pull/26517
- Head SHA: `63e7a669c75a04cc61b97cd4df6773f5d84a322b`
- Verdict: **merge-after-nits**

## What it does

Three UI-side gaps closed against backend endpoints that already shipped:

1. Internal-User row gets a red "Disconnect" link in the MCP server list, wired
   to `DELETE /v1/mcp/server/credential` via a new `deleteUserCredential`
   helper.
2. The MCP server list gets an `approval_status` filter so non-admins only see
   `active` (or unscoped) servers — pending/rejected ones stop leaking.
3. The BYOK token form is now gated behind `transportType === TRANSPORT.OPENAPI
   && isAdmin` so internal users can't open the admin-only credential form.

## Diff notes

- Filter logic in the server list:
  ```
  if (isInternalUser) {
    filtered = filtered.filter(
      (server) => !server.approval_status || server.approval_status === "active",
    );
  }
  ```
  Correct shape — preserves servers with no `approval_status` field (legacy
  rows) instead of hiding them, which matches what the backend treats as
  "default = visible".
- BYOK guard reads `transportType === TRANSPORT.OPENAPI && isAdmin`. Good —
  this is the only place the credential form mounts, so non-admins truly
  cannot reach it from the UI.
- `deleteUserCredential` does the right error unwrap
  (`errorData?.detail?.error ?? errorData?.error ?? "..."`) and surfaces
  through `NotificationsManager.fromBackend`.

## Nits

- `model_prices_and_context_window_backup.json` carries unrelated GPT-5.5
  pricing churn in the same PR. Splitting that out would make the security-
  sensitive UI change easier to audit and bisect later. Not blocking, but I'd
  ask for it before merge if the repo enforces small-PR hygiene.
- The disconnect button has no confirm modal in this diff — for a destructive
  credential delete, even a `window.confirm` would be a worthwhile follow-up.
- Worth adding a Vitest case that asserts the row renders **no** Disconnect
  link when `isInternalUser === false` (admins still get "Update"), to lock
  down the role split.

## Why this verdict

The behavior change is correct, scoped, and matches the backend contract.
Two things keep it out of `merge-as-is`: an unrelated pricing-table churn in
the same diff, and the missing destructive-action confirmation. Both are
small enough to land as follow-ups, so `merge-after-nits` rather than
`request-changes`.
