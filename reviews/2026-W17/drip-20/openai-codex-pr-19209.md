---
pr_number: 19209
repo: openai/codex
head_sha: 8d84ad8b3131303d375b854aa100efed56614d12
verdict: merge-as-is
date: 2026-04-25
---

# openai/codex#19209 — emit usage-limit-prompt analytics from TUI

**What changed.** +126 / −1. Five touchpoints: (1) `tui/src/app_event.rs:200` adds a new `AppEvent::TrackUsageLimitBanner { action, credit_type }` variant; (2) `tui/src/app/event_dispatch.rs:482` routes that event to a new helper; (3) `tui/src/app/background_requests.rs:62` adds `track_usage_limit_banner` (spawns a task that fires `ClientRequest::TrackUsageLimitBanner` with a `usage-limit-banner-{uuid}` request id); (4) `tui/src/chatwidget.rs:8092` and 8124 wire the actual emissions — `Shown` fires when `start_workspace_member_credits_depleted_prompt` opens the popup, `CtaClicked` fires when the `y` selection action runs, and the existing `SendAddCreditsNudgeEmail` is left untouched; (5) `chatwidget/tests/status_and_layout.rs:605–660` extends the two pre-existing prompt tests (`workspace_member_credits_depleted_prompts_and_sends_credits` and `workspace_member_usage_limit_prompts_and_sends_usage_limit`) with `next_track_usage_limit_banner_event` assertions for the `Shown` then `CtaClicked` ordering and the correct `AddCreditsNudgeCreditType`.

**Why it matters.** Without `Shown`, you can't compute denominator for CTA conversion — only the `cta_clicked` count gets observed, leaving "show rate × click-through" indistinguishable. With both events in place plus the existing email-send signal, the funnel has shown → cta_clicked → email_sent. Member-facing wording-only feature; no sandbox/permission/secrets surface touched.

**Concerns.**
1. **`Shown` is emitted *after* the `selection_view.show(...)` call (chatwidget.rs:8124).** If a panic between those lines escapes (it shouldn't — `selection_view.show` is infallible per the surrounding code), the popup renders but no `Shown` event fires. Cosmetic — but consider emitting *before* `show` for guaranteed denominator coverage even on weird interleavings.
2. **`CtaClicked` is enqueued via the `SelectionAction` closure that *also* enqueues `SendAddCreditsNudgeEmail`** (line 8092). They go on the same channel in that order, so the analytics event always lands strictly before the email-send event. Good — but a 1-line comment naming that ordering invariant would help future editors not split the closure.
3. **`request_id: RequestId::String(format!("usage-limit-banner-{}", Uuid::new_v4()))`** is unique per call (good). The handler ignores the response body (`let _: TrackUsageLimitBannerResponse = ...`) — confirm the response type is genuinely unit-shaped. If the protocol returns an ack with status, swallowing it is fine; if it returns something that could surface a server-side validation error, log it.
4. **`track_usage_limit_banner` errors are `tracing::warn!`-only.** Right call for a non-critical analytics ping — but verify it doesn't cascade: `wrap_err("account/usageLimitBanner/track failed in TUI")` could include the inner serialized request which might contain `credit_type` enum names. Those are presumably PII-free (`Credits` / `UsageLimit`), but worth confirming.
5. **No event for the `No`/dismiss path.** The body explicitly says "selecting `No` only dismisses the prompt." That's a deliberate product choice, but downstream the `Shown - CtaClicked` delta is the *only* signal of dismissals. If a future dashboard needs explicit dismissal tracking, add `TrackUsageLimitBannerAction::Dismissed` now to avoid a second wire-protocol bump.

Tight, well-tested, low-risk. Ship it.
