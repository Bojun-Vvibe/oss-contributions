# sst/opencode #25898 — fix(tui): list root sessions in session picker

- **PR:** sst/opencode#25898
- **Head SHA:** `1bbe5a78f00f00c93c7b07371fb4a47ebd7664c0`
- **Files:** 3 changed (+14 / -6)

## What changed

- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx:37` — the search query path now passes `roots: true` to `sdk.client.session.list({...})` so the picker's text-search results include root sessions (sessions that have no parent), not just children of the active session.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:129` — the unfiltered list call drops the implicit "last 30 days" `start: Date.now() - 30 * 24 * 60 * 60 * 1000` window in favour of an explicit `roots: true, limit: 1000`. This is a behavioural shift: the picker now goes broader-but-shallower (top 1000 root sessions, no time bound) instead of narrower-but-deeper (any session created in the last 30 days, no limit beyond the API default).
- Test updates at `packages/opencode/test/cli/cmd/tui/sync.test.tsx:133-151` lock in both branches of the new contract: with `session_directory_filter_enabled=true` the request includes `path=packages/opencode`, no `scope`, `roots=true`, `limit=1000`, and crucially **no `start` param**; with the directory filter disabled it sends `scope=project`, no `path`, plus the same `roots`/`limit`/no-`start` set.

## Risks / notes

- The 30-day window deletion is the load-bearing change here, not the `roots: true` flag. For users with large session histories this is now bounded purely by `limit: 1000`; users with >1000 root sessions will silently lose access to anything past that cap and have no time-based fallback. Worth confirming the server applies a sensible default ordering (most-recent-first) so the cap bites at "old" sessions rather than at random.
- The two callsites both gain `roots: true` but only one (`sync.tsx`) drops `start`. Asymmetric — the `dialog-session-list` query path inherits `...input.filter` last, which on the search path doesn't supply `start`, so behaviour is consistent in practice; just visually awkward.
- No mention of pagination. If the new `limit: 1000` represents a server-side hard cap rather than a request-side preference, users with bigger histories need follow-up.

## Verdict

**merge-after-nits** — fix is correct and well-tested but the silent removal of the 30-day fallback in favour of an unbounded-by-time, capped-at-1000 list deserves a release note and ideally a follow-up that exposes the cap to the user (e.g. "showing 1000 most recent — type to search for older").
