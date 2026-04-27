# openai/codex PR #19769 — Add proactive 90%-usage upgrade prompt to TUI

- **PR**: https://github.com/openai/codex/pull/19769
- **Author**: @jchu-oai
- **Head SHA**: `9aa3206b`
- **Size**: +347 / −0
- **Files**: `codex-rs/tui/src/app/event_dispatch.rs`, `codex-rs/tui/src/app_event.rs`, `codex-rs/tui/src/chatwidget.rs`, `codex-rs/tui/src/chatwidget/snapshots/codex_tui__chatwidget__tests__proactive_usage_prompt.snap`, `codex-rs/tui/src/chatwidget/tests/helpers.rs`, plus the new test in `status_and_layout.rs`

## Summary

Adds a one-shot upsell prompt that fires when a rate-limit snapshot reports any window crossing 90% used (and < 100%) for ChatGPT-account-backed users. The CTA branches by plan: Plus → "Upgrade now" → Pro upgrade URL; Pro/ProLite → "Buy credits" → usage settings URL; workspace owners (`WorkspaceOwnerCreditsDepleted` | `WorkspaceOwnerUsageLimitReached`) → "Review usage options" → usage settings URL. Workspace *members* are intentionally excluded (no direct purchase prompt). Selecting "yes" opens the URL via `AppEvent::OpenUrlInBrowser` and posts a follow-up info message ("Finish in your browser, then return here to continue.") via the new `AppEvent::AddInfoMessage`. The prompt is one-shot per session via `ProactiveUsagePromptState::{Idle, Pending, Shown}` — once Shown, never re-fired.

The new prompt also *suppresses* the existing `RateLimitSwitchPromptState` model-switch nudge for the same threshold crossing — the high-usage gating at `chatwidget.rs:3034` adds `&& !proactive_usage_prompt_eligible` so users don't get both prompts back-to-back. The PR is explicitly draft and the description notes UTM and copy may be tuned.

## Verdict: `needs-discussion`

The mechanics are clean and the snapshot test pins the expected popup shape. But this is a product/policy change in addition to a code change, and the policy decisions deserve explicit discussion before code review picks them apart.

## Specific references

- `codex-rs/tui/src/chatwidget.rs:483-486` — three new constants:
  ```rust
  const PROACTIVE_USAGE_PROMPT_THRESHOLD: f64 = 90.0;
  const PROACTIVE_USAGE_PROMPT_USAGE_URL: &str = "https://chatgpt.com/codex/settings/usage?utm_source=codex_cli&utm_medium=cli&utm_campaign=usage_limit_prompt&utm_content=threshold_90";
  const PROACTIVE_USAGE_PROMPT_PRO_URL: &str = "https://chatgpt.com/explore/pro?utm_source=codex_cli&utm_medium=cli&utm_campaign=usage_limit_prompt&utm_content=threshold_90";
  ```
  The 90.0 threshold has no relationship to the existing `RATE_LIMIT_WARNING_THRESHOLDS: [f64; 3] = [75.0, 90.0, 95.0]` array (line 481) other than coincidentally sharing the middle value. If product later moves the upsell threshold to 85, the warnings array still fires at 90 — these should either share a constant or have a comment at each site explaining why they're separate.
- `codex-rs/tui/src/chatwidget.rs:556-571` — the threshold predicate is correctly OR'd across `primary` and `secondary` windows:
  ```rust
  fn rate_limit_snapshot_crosses_proactive_prompt_threshold(snapshot: &RateLimitSnapshot) -> bool {
      snapshot.primary.as_ref().is_some_and(rate_limit_window_crosses_proactive_prompt_threshold)
          || snapshot.secondary.as_ref().is_some_and(rate_limit_window_crosses_proactive_prompt_threshold)
  }

  fn rate_limit_window_crosses_proactive_prompt_threshold(window: &RateLimitWindow) -> bool {
      window.used_percent >= PROACTIVE_USAGE_PROMPT_THRESHOLD && window.used_percent < 100.0
  }
  ```
  The `< 100.0` upper bound is the right shape — at 100% the user is already at the limit and a different code path (`RateLimitReachedType` handling) takes over. Without this bound the prompt would fire *redundantly* alongside the 100% blocker.
- `codex-rs/tui/src/chatwidget.rs:3025-3038` — gating in the snapshot handler. `proactive_usage_prompt_eligible` is computed once and used twice: first to set `ProactiveUsagePromptState::Pending`, then to *suppress* the existing `high_usage` model-switch nudge:
  ```rust
  if high_usage
      && !proactive_usage_prompt_eligible
      && !has_workspace_credits
      && !self.rate_limit_switch_prompt_hidden()
      && self.current_model() != NUDGE_MODEL_SLUG
  ```
  The mutual-exclusion is correct (no double-prompt) but introduces a *new* implicit precedence: at exactly 90%, ChatGPT-account users see the upsell prompt and never the model-switch nudge, even if their model is gpt-5 and the cheaper alternative would actually solve their problem. A user who would benefit from "switch to gpt-5.4-mini" gets "spend more money" instead.
- `codex-rs/tui/src/chatwidget.rs:8019-8060` — `should_show_proactive_usage_prompt` correctly gates on `has_chatgpt_account` (so API-key users don't see CTAs that require a ChatGPT login), the state being `Idle` (one-shot guarantee), and the threshold predicate. Workspace member exclusion is implicit: the `paid_personal_user || workspace_owner` gate at L8062-8077 returns `false` for workspace members because they have no `PlanType::{Plus,Pro,ProLite}` and no owner-`RateLimitReachedType`. Worth a `// workspace members fall through here intentionally — no direct purchase prompt` comment so a future reader doesn't "fix" the gap.
- `codex-rs/tui/src/chatwidget.rs:8083-8096` — `proactive_usage_prompt_content` switches on `self.codex_rate_limit_reached_type` *first* (workspace-owner branch), then on `self.plan_type` (personal branches). The order matters because a workspace-owner user could plausibly also be Plus/Pro on their personal account; the workspace-owner branch winning means the CTA tracks the *active* rate-limit context (workspace) rather than the user's personal plan. Correct, but undocumented.
- snapshot file `proactive_usage_prompt.snap` — pins the exact rendered popup including the default-selected "No" arrow and the standard footer hint. Locks the UX so a future copy tweak in the constants will fail loudly. Good.

## Concerns / discussion points

1. **Mutual-exclusion choice**: At 90% high-usage, ChatGPT-account Plus users will now *only* see the upsell, not the model-switch nudge. Is this the intended product policy? The model-switch nudge is the strictly-cheaper option for the user; replacing it with "buy more" is a revenue-positive policy choice that should be made by product, not as a side effect of an `&& !proactive_usage_prompt_eligible` clause. Worth a comment naming this as intentional, and worth confirming with PM that "show user the cheaper option first, only upsell if they decline" isn't the better policy.
2. **One-shot per session, but session boundaries are fuzzy**: `ProactiveUsagePromptState` lives on `ChatWidget`, which is recreated when the user runs `codex` again. So a user who hits 90%, sees the prompt, clicks "No", finishes their work, then runs `codex` again the next day at 92% will see the prompt *again*. That's probably fine for a usage-limit prompt (the situation hasn't changed), but worth being explicit: the state is "per process" not "per account" or "per N days". A user-side `codex_home/.proactive_prompt_dismissed_until` JSON would be more polite.
3. **No analytics plumbing** (PR notes this explicitly). Without a decline-rate signal, you can't tell if the prompt is converting or just annoying users. Acceptable for a draft PR; should be a separate ticket before merge.
4. **UTM strings are baked into the binary as constants**. Fine for now, but means a UTM tweak requires a release. Not a blocker — just noting that A/B-testing the UTM (which campaigns convert) requires either dynamic UTM injection or accepting that you ship a new binary per UTM change.
5. **`AppEvent::AddInfoMessage` is a brand-new event for this PR** but is plausibly useful elsewhere (any future "background completion" flow that needs to drop a message into transcript without a tool/turn boundary). The event handler at `event_dispatch.rs:305-307` calls `chat_widget.add_info_message(message, /*hint*/ None)` with `None` for the hint — the inline `/*hint*/` comment is a nice tell for future call sites that there's a second arg they're choosing not to populate.
6. **Test coverage gap**: the snapshot test pins the Plus branch's popup. There's no parallel snapshot for the Pro/ProLite or workspace-owner branches, even though they have distinct CTA strings (`Buy credits`, `Review usage options`). Three snap files would lock all three branches.
7. **`PROACTIVE_USAGE_PROMPT_BROWSER_MESSAGE`** ("Finish in your browser, then return here to continue.") assumes the user's default browser opens — but the existing `OpenUrlInBrowser` event has no failure callback, so on a headless terminal the URL will silently never open and the user will see "Finish in your browser" with no browser. Not introduced by this PR, but worth noting that the upsell flow is now exposed to that pre-existing failure mode.

## What I learned

The shape of "one-shot upsell prompt with state machine" is well-trod (Idle → Pending → Shown), and pinning the rendered popup with a snapshot test is the right way to lock UX so non-developer copy changes can't slip through unnoticed. The interesting design question this PR raises isn't technical — it's the *precedence* between two prompts that share a threshold: when both could fire, the choice of which one wins is a product decision, not a code decision. Encoding that choice as `&& !other_eligible` is correct mechanics but invisible policy; calling it out in a comment ("intentionally suppresses the cheaper-model nudge in favor of the upsell at 90%") would make the policy decision auditable in code review.
