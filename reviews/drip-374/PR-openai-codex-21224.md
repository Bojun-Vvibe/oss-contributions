# openai/codex #21224 — Add workspace announcement polling client

- URL: https://github.com/openai/codex/pull/21224
- Head SHA: `809185ddde3c86e7c2fbefa4bcfffa4b73a99647` (`809185d`)
- Author: @xli-oai
- Diff: +183 / −0 across 6 files
- Verdict: **merge-after-nits**

## What this changes

This is the PR-1 plumbing for an upcoming workspace-announcement
feature: the data-plane glue plus a background poller that reads but
does not act. Concretely:

- `backend-client/src/types.rs` (+37) introduces a
  `CodexWorkspaceMessagesResponse` shape and an `announcements()`
  helper iterator. Per the PR body, only `created_at` is modeled and
  it's modeled as `String` — no `chrono` dep, no `updated_at` —
  deliberately conservative so the schema can grow without forcing
  type churn here.
- `backend-client/src/client.rs` (+49) adds
  `Client::list_workspace_messages()` (path resolved via a new
  `workspace_messages_url()` helper that respects `path_style`) plus
  a small `lib.rs` export bump.
- `features/src/lib.rs` (+8) adds a new `Feature::WorkspaceMessages`
  variant.
- `app-server/src/lib.rs` (+15) wires the poller in: the spawn is
  gated on `config.features.enabled(Feature::WorkspaceMessages)` at
  line ~697, the spawned `JoinHandle` is awaited cooperatively after
  `transport_shutdown_token.cancel()` at line ~1043, and the new
  module is declared at line 97.
- `app-server/src/workspace_messages.rs` (+71, new file) — the
  poller itself.

## Why the poller shape is correct

`spawn_announcement_poller` constructs a
`tokio::time::interval(ANNOUNCEMENT_POLL_INTERVAL)` (15 minutes,
const at line 12) and explicitly sets
`MissedTickBehavior::Delay` at line 23 — important because the
default (`Burst`) would fire a flurry of catch-up polls if the
process was suspended/laptop-slept past multiple intervals, hammering
the backend on resume. `Delay` correctly says "if we missed several
ticks, just resume from now and don't try to make up the gap."

The tokio-select inside the loop (lines 25-30) races
`shutdown_token.cancelled()` against `interval.tick()`, so shutdown
is responsive at any point during the 15-minute wait without needing
a separate timeout. `interval.tick()` fires immediately on first
poll (tokio's documented behavior) which matches the PR body's
"poll immediately and then every 15 minutes" claim — no need for an
initial direct call before the loop.

`poll_announcements` (lines 36-55) wraps the fetch in a
`timeout(ANNOUNCEMENT_FETCH_TIMEOUT, ...)` (5 seconds, const at line
13). The three-arm match handles success, fetch-error, and timeout
distinctly with `tracing::debug!` at each branch — the explicit
choice of `debug!` (not `warn!` or `error!`) is consistent with the
PR's "fail-open on auth/network/timeouts" stated policy: a failed
poll should never escalate to operator-visible noise because the
feature is non-essential and the next poll is ≤15 minutes away.

`fetch_announcement_count` (lines 58-71) is the pre-flight gate that
keeps this from spinning against unauthenticated installs:
`auth_manager.auth().await` returning `None` → `Ok(0)`,
`auth.uses_codex_backend() == false` → `Ok(0)`. Only when both gates
pass does it construct `BackendClient::from_auth(...)` and call
`list_workspace_messages()`, then return
`messages.announcements().count()`. The result count is logged at
`debug!` and otherwise discarded — explicit per the PR body: "do not
ack or render yet."

## Concerns

(a) The shutdown semantics in `app-server/src/lib.rs` are sequenced
correctly (cancel token → await poller → loop over transport accept
handles), but the `let _ = handle.await;` discards any panic from
the poller task. That's fine for an opt-in feature in early rollout,
but if `Feature::WorkspaceMessages` ever flips to default-on it would
be worth either logging the JoinError or using `.abort()` + a
shorter join timeout so a wedged poller can't extend shutdown
indefinitely.

(b) `ANNOUNCEMENT_FETCH_TIMEOUT = 5s` is reasonable for the happy
path against a healthy backend, but combined with the `interval.tick()`
firing immediately, a slow first poll on app-server cold-start could
overlap with the user's first thread/start request. Not a correctness
issue, just resource overhead at startup; consider a small initial
delay (`interval.reset_after(Duration::from_secs(30))`) so the poller
yields the cold-start window to user-facing requests.

(c) The poll discards the announcement *content* (only `count()` is
captured into the debug log). That's fine for this PR's stated scope
("do not ack or render yet"), but the debug log line should include
either an announcement-IDs hash or the raw count of *new* (vs.
already-seen) announcements once a future PR adds local
de-duplication state — otherwise the log will be uniformly "saw N
announcements" forever and won't help operators verify the
upcoming render path is actually delivering only-new entries. Worth
a `// TODO(workspace-messages): emit deltas, not totals` breadcrumb.

(d) `list_workspace_messages()` returns
`Result<CodexWorkspaceMessagesResponse, RequestError>`. The poller
discards the `RequestError` enum into `debug!(?err, ...)`. That's
fine for transient failures, but a 401/403 here likely means the
auth state has gone stale in a way that *will* be visible elsewhere
in the app-server — wiring an explicit "auth-error during poll →
nudge auth_manager to refresh" hook is out of scope for this PR but
worth noting for a follow-up.

PR is well-scoped, conservative on dependencies (no `chrono`),
responsive to shutdown, and gated behind a feature flag. Ships once
the four nits above are either addressed or explicitly deferred.
