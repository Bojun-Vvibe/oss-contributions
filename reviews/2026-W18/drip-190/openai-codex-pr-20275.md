# openai/codex #20275 — fix: show correct Bedrock runtime endpoint in /status

- **URL:** https://github.com/openai/codex/pull/20275
- **Head SHA:** `ddcefe604ecace3d22694a3f30fe9c3ccfcf8446`
- **Files:** 14 files (+150/-17) — `codex-rs/model-provider/src/amazon_bedrock/{auth,mantle,mod}.rs`, `provider.rs`, `tui/src/{app,chatwidget,status/card,status/tests}.rs` + 5 test fixtures + `Cargo.{toml,lock}`
- **Verdict:** `merge-after-nits`

## What changed

`/status` was rendering the *configured* Bedrock base URL, which for Amazon Bedrock providers is a placeholder (`bedrock-mantle.us-east-1.api.aws/...`) that gets *replaced* at request time with a region-resolved URL based on `AWS_REGION` env / profile / SDK chain. Result: status displayed `us-east-1` to a user actually hitting `eu-west-1`. This PR plumbs the runtime-resolved URL into `/status`:

1. **Trait surface.** `provider.rs:107-112` adds a default `runtime_base_url(&self) -> Result<Option<String>>` returning `self.info().base_url.clone()`. Generic providers get correct behavior for free; only providers whose request-time URL diverges from the configured one need to override.
2. **Bedrock override.** `amazon_bedrock/mod.rs:88-91` overrides to call new `runtime_base_url(&self.aws)` from `mantle.rs`. `mantle.rs:51-65` adds `runtime_base_url` and a private `resolve_region` (moved from `auth.rs` — old `resolve_region` deleted at `auth.rs:58-64`). `BedrockAuthMethod` and `resolve_auth_method` are made `pub(super)` so `mantle` can re-call them.
3. **TUI plumbing.** `app.rs:603-610` adds `resolve_runtime_model_provider_base_url` that constructs a fresh `ModelProvider` (no auth manager) and calls `runtime_base_url`, with a `tracing::warn!` on failure so a transient AWS SDK error doesn't poison `/status`. The resolved value is cached in `App::run` at `:797-799` and threaded through `ChatWidgetInit { runtime_model_provider_base_url: Option<String> }` (init field at `chatwidget.rs:609`, stored at `:826`, exposed via accessor at `:10379-10382`) into the `/status` render path (`status/card.rs:166,188,234`).
4. **`format_model_provider` fix.** `status/card.rs:706-714` swaps `provider.base_url.as_deref().and_then(sanitize_base_url)` for `runtime_base_url.and_then(sanitize_base_url)`. The `is_default_openai` check still uses `base_url.is_none()` (the *runtime* one), preserving the "don't show provider chip for default OpenAI" UX.
5. **Test.** `status/tests.rs:243-292` adds `status_model_provider_uses_bedrock_runtime_base_url`: configures provider with `us-east-1` base URL, passes runtime URL `bedrock-mantle.eu-west-1.api.aws/openai/v1`, asserts the rendered status contains the runtime URL and *not* the configured one. All other existing test call sites get an explicit `/*runtime_model_provider_base_url*/ None` parameter.

## Why it's right

- Diagnosis is correct: Bedrock's auth-time region resolution (`AWS_REGION` / SDK chain) is what determines the actual endpoint, and that resolution can't be done synchronously at config-load time — it needs an async SDK call. The previous code displayed config-time, the fix displays auth-time.
- Trait default `Ok(self.info().base_url.clone())` is the right shape — non-Bedrock providers don't pay for the change, and the override is opt-in.
- The test contract is correct: assert the runtime URL is rendered AND assert the configured URL is *not* rendered. That second assertion is what locks the contract against a future regression that double-renders both.
- `resolve_region` move from `auth.rs` to `mantle.rs` is the right boundary — region resolution is a mantle concern (mantle base-url generation), not an auth concern. Auth still owns `resolve_auth_method` / `BedrockAuthMethod` and exports them at `pub(super)` precisely for this consumer.
- Failure mode at `app.rs:608` is warn-and-fall-back-to-`None`, which means "show no provider chip" rather than "crash startup" if AWS SDK is misconfigured at boot. Operator-friendly.

## Nits / not blockers but worth a follow-up

- **Auth-manager == None at app.rs:604.** `create_model_provider(provider.clone(), /*auth_manager*/ None)` constructs a fresh provider just to call `runtime_base_url`, but Bedrock's `resolve_auth_method` calls into `bearer_token_from_env()` and the AWS SDK chain — both side-effecting. If the bearer token rotates between status-card init time and request time, the runtime URL shown could be stale. Worth a comment naming "captured at startup, not refreshed on /status invocation" so a future change to refresh-on-demand is obvious.
- **The fresh-`ModelProvider` allocation per `/status` render is wasteful.** The runtime URL is cached on `ChatWidget`, but the `replace_chat_widget_reseeds_collab_agent_metadata_for_replay` path at `app/tests.rs:4632-4635` shows the value is propagated by *reading* the existing chat widget's accessor — that's fine for replay, but the original `App::run` resolves it once at startup and never refreshes. If someone later wants `/status` to reflect a runtime region switch (e.g. AWS SDK region change mid-session), the architecture forces a chat-widget-replace. A cheap `runtime_base_url_provider: Arc<dyn Fn() -> ...>` injection would make that future change trivial; the current design wedges in a static value.
- **Test thread-through churn.** Five test files (`app/tests.rs:411`, `chatwidget/tests/{helpers,plan_mode,popups_and_settings,status_and_layout}.rs`) all get `runtime_model_provider_base_url: None` added. That's fine, but a `Default` impl on `ChatWidgetInit` (or a builder) would absorb future field adds without N more diff lines per change. Not blocking this PR, just a structural follow-up.
- **`#[allow(clippy::too_many_arguments)]` on `new_status_output_with_rate_limits_handle` at `card.rs:185`.** Adding the 17th positional argument is a clippy-silencer red flag. Worth a struct-args follow-up the next time this signature touches.

## Risk

Low. The only behavioral change visible to users is `/status` showing a different (correct) Bedrock URL. The trait default returns the configured URL, so non-Bedrock providers are byte-identical. Failure path is warn+`None`, not panic.
