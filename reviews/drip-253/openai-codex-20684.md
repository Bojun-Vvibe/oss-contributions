# openai/codex #20684 — tui: hook trust review flow

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20684
- **HEAD SHA:** `faba78ece2509b5ebbf4333fe5d068ad1de90071`
- **Author:** abhinav-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Wires a "hook trust" review surface into the TUI so newly-installed or
modified hooks (untrusted/modified `HookTrustStatus`) cannot run silently
and the user is nudged to review them. Three pieces:

1. `app/background_requests.rs:88-129` — new `refresh_startup_hooks`
   spawns a tokio task at app startup that calls `fetch_hooks_list`,
   filters for `cwd == app.config.cwd`, counts hooks whose
   `trust_status` is `Untrusted | Modified`, and emits a
   `WarningEvent` history cell via
   `startup_prompts::hooks_needing_review_warning(count)`. Called from
   `app.rs:929` right after `refresh_startup_skills`.
2. New `TrustHook { key, trusted_hash, enable }` AppEvent
   (`app_event.rs:752-756`) plus dispatch arm
   (`event_dispatch.rs:1695-1701`) plus `App::trust_hook`
   (`background_requests.rs:367-384`) that fires
   `write_hook_trust(...)` (`:868-908`) — a `config/batchWrite`
   request against `hooks.state` keyed by hook stable key, persisting
   `{enabled, trusted_hash}` (or just `trusted_hash` when disabling).
   `HookTrusted { result }` arm (`event_dispatch.rs:1726-1730`)
   surfaces failures via `add_error_message`.
3. `bottom_pane/hooks_browser_view.rs:71-180` — auto-selects the first
   row whose `needs_review > 0` so the user lands on the actionable
   entry, splits the row counts into `active` (runnable) and
   `needs_review` (untrusted/modified), and at `:182` early-returns
   from the toggle path when `hook_needs_review(hook)` so users can't
   accidentally enable an untrusted hook with the same keybind that
   toggles trusted ones.

Plus a snapshot test
`hooks_needing_review_startup_warning_snapshot` at `app/tests.rs:266-275`
locking the rendered warning string at width 80.

## Why it's right

The trust gate is at the right place (`hook_needs_review` short-circuits
the *toggle* path in `hooks_browser_view.rs:181-183`), which means the
review-then-trust flow is the *only* way an untrusted hook becomes
runnable — there's no "enable" backdoor that skips the trust check.
That's the load-bearing detail; without it the toggle key would still
silently flip `enabled: true` even on `Untrusted` rows.

Persisting both `enabled` and `trusted_hash` together when enable=true
(but only `trusted_hash` when enable=false) at `:870-887` is the right
shape — disabling a previously-trusted hook should preserve the trust
record so re-enabling later doesn't re-prompt for review of an
unchanged definition. The `MergeStrategy::Upsert` at `:898` is correct:
other hook keys' state must not be clobbered.

`startup_prompts::hooks_needing_review_warning` correctly returns
`None` for count==0 (no spam on clean state), singularizes for count==1,
and pluralizes otherwise — all three branches are exercised by the
snapshot test (count==2 path).

The `cwd == app.config.cwd` filter at `:108` matters because
`fetch_hooks_list` returns entries across all cached project cwds; only
the current project's hook population is relevant for the startup
warning.

## Nits

1. **`refresh_startup_hooks` swallows the error path** at `:96-98` with
   `tracing::warn!` only — no warning cell, no retry, no
   user-visible signal that "the trust state could not be loaded so
   you can't tell if there's something to review". Consider emitting
   a one-time `new_warning_event("could not load hook trust state;
   open /hooks to review manually")` so the user has a non-trace
   signal.

2. **Snapshot test only covers the count==2 path.** The `0 → None` and
   `1 → singular` arms in `hooks_needing_review_warning` at
   `app/startup_prompts.rs:80-86` are not directly exercised. The
   `count==0` case is the load-bearing "do not spam clean projects"
   one — a `pure` unit test
   `assert_eq!(hooks_needing_review_warning(0), None)` plus a
   singular-form test would lock the contract without needing
   another snapshot baseline.

3. **No test for the toggle-blocked-on-needs-review behavior** at
   `hooks_browser_view.rs:181-183`. The new early-return is the
   most security-relevant part of the diff (it's what makes "trust
   first, then enable" the only path), but no test pins it. A
   minimal unit test constructing a `HooksBrowserView` with one
   `Untrusted` hook and asserting the toggle action is a no-op
   would lock the contract — without it, a future cleanup that
   reorders branches in the toggle handler could quietly let
   untrusted hooks be enabled.

4. **`HookTrusted { result }` only surfaces failures** at
   `event_dispatch.rs:1726-1730`. On success there's no
   acknowledgment cell, so the user pressing the trust key sees the
   row state change but no confirmation that the persistence
   round-trip completed. The sibling `HookEnabledSet` arm at
   `:1707-1722` *does* refresh the underlying list on success — the
   trust arm should mirror that pattern (refresh hooks list so the
   `trust_status` flips from `Untrusted/Modified` to `Trusted` in
   the visible row immediately, rather than waiting for the next
   pull).

5. **`refresh_startup_hooks` runs unconditionally at startup**
   (`app.rs:929`). For users on projects with no hooks configured
   the `fetch_hooks_list` request is a guaranteed empty round-trip
   to the app server. Worth a fast-path skip if
   `self.config.hooks.is_empty()` (analogous patterns exist
   elsewhere in `app.rs`).

## Verdict rationale

Right architectural shape: the trust gate is at the toggle path so
untrusted hooks can't be enabled without going through review, the
startup warning uses a singular/plural/none branched message, and the
batchWrite persistence preserves trust records across enable/disable
cycles. Snapshot-only test coverage on the rendered warning misses
the `0`/`1` arms and the most security-relevant new behavior (toggle
blocked on `needs_review`); the success path of `HookTrusted` should
refresh the list to match the sibling `HookEnabledSet` pattern.

`merge-after-nits`
