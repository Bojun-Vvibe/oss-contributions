# openai/codex #20837 — Add hook auto review

- URL: https://github.com/openai/codex/pull/20837
- Head SHA: `b2e53180ebf9b0f949c9c45891c0fa065db578e8`
- Scope: +2121 / -51 across 60 files

## Summary

Extends `auto_review` mode to also pre-review newly loaded hooks before
`SessionStart` hooks run. Adds a durable `Dangerous` hook trust status,
new metadata (`reviewed_by`, `dangerous_hash`, `dangerous_reason`), two
new notifications (`hook/autoReview/started`, `hook/autoReview/completed`),
and a `reviewingHooks` thread active flag so clients can render the
"startup paused for hook review" state.

## What I checked

- **Protocol surface** (`app-server-protocol/schema/json/v2/HookAutoReviewStartedNotification.json`,
  `HookAutoReviewCompletedNotification.json`, `HookAutoReviewDangerousHook.json`)
  — schemas are concise and use `uint32` for counts (good — JSON safe-integer
  range). `HookMetadata.ts` and `HookTrustStatus.ts` both bumped by 1 line,
  matching the new field/variant additions. `ThreadActiveFlag.ts` adds the
  `reviewingHooks` variant.
- **Core review pipeline** (`codex-rs/core/src/hook_auto_review.rs`, +276) —
  new module containing the locked-down review pass. Runs before
  `SessionStart`. Verdicts persist via `dangerous_hash` so re-running the
  same hook source short-circuits.
- **Guardian glue** (`core/src/guardian/hook_review.rs`, +284) — review
  invocation lives next to the existing guardian module. Reasonable place.
- **Hook engine** (`hooks/src/engine/discovery.rs:39+`) — discovery now flags
  new/modified hooks for review. The `unmanaged_hook_trust_status_tracks_stored_hash`
  test (referenced in the PR description) covers the hash-based change
  detection.
- **TUI surface** (`tui/src/chatwidget.rs:84+`, `tui/src/bottom_pane/hooks_browser_view.rs:60+`)
  — `Reviewing hooks` status, the `[!] Auto-review marked X hook dangerous`
  warning, and `/hooks` showing reviewer + danger reason. Snapshot
  `hooks_browser_dangerous_handler.snap` confirms the rendered state.
- **Test matrix** — 9 distinct cargo test invocations listed in PR description.
  Notable: `hook_auto_review_persists_verdicts_before_session_start_hooks`
  proves verdicts land before any hook code can run, which is the security
  property that matters.

## Concerns / nits

1. **Reviewer trust is implicit.** The reviewer-as-LLM model is reviewing
   shell scripts that the same agent infrastructure could end up running.
   PR description shows the reviewer marking "reads SSH keys and posts them
   to a remote host" as dangerous, which is exactly what we want — but
   prompt-injection in the hook source could try to trick the reviewer.
   `hook_auto_review.rs` should probably include a brief note on the threat
   model: what is the reviewer's prompt isolation strategy? (The `guardian`
   directory naming hints at this; a docstring at the top of
   `hook_auto_review.rs` explaining "reviewer never executes the hook,
   only reads its source as text" would be reassuring.)
2. **Failed reviewer ⇒ blocked.** If the reviewer LLM call fails (network,
   rate limit), the hook stays blocked. Good default. But there's no escape
   hatch documented — a user with no network may be stuck unable to load
   any new hooks. Worth a brief docs note on the manual `t to trust`
   bypass shown in the PR's `/hooks` mock.
3. **Schema breakage check.** Adding a variant to the `HookTrustStatus` enum
   (`Dangerous`) is a breaking change for any client that exhaustively matches
   on the variant set. Schema bumps usually want a version note. The
   `v2.schemas.json` bump reflects this is in v2 already, so probably fine.
4. **`dangerous_hash` collision risk.** Confirm the hash is content-addressed
   over the full hook source plus any environment that affects execution
   semantics (cwd, shell, etc.). Otherwise two hooks with identical source
   but different shells could share a verdict.
5. PR is 2121 lines across 60 files, with 14 schema files alone. The schema
   churn is mechanical and necessary; reviewers should focus on
   `hook_auto_review.rs`, `guardian/hook_review.rs`, and the engine
   discovery changes.

## Risk

Medium-high. Security-relevant feature touching the trust boundary for
arbitrary shell hooks. Test coverage is strong (9 cargo test scopes).
The reviewer-LLM-judges-shell-script approach is a real architectural
choice — getting it wrong opens an injection vector. Worth a security
team eyeball before merge.

## Verdict

`needs-discussion` — primarily on threat model: prompt isolation for the
reviewer, escape hatch when reviewer is unavailable, and `dangerous_hash`
input scope. The implementation looks careful; the architectural choice
deserves an explicit sign-off.
