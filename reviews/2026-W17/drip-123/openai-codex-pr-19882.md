# openai/codex#19882 — Add /hooks browser for lifecycle hooks

- **PR**: https://github.com/openai/codex/pull/19882
- **Author**: abhinav-oai (Abhinav)
- **Head SHA**: `7656221d30ec942212a856e96b75cc1d98c6ce46`
- **Verdict**: `merge-after-nits`

## Context

Codex's hooks engine (`codex-rs/hooks/`) ships a discovery + execution path for user-defined lifecycle hooks (`pre_tool_use`, `post_tool_use`, etc.) sourced from config layers and plugin marketplaces. The TUI has had no first-class browser for these hooks: users had to read their config files to know what was wired up, and there was no way to toggle a hook off without editing TOML by hand. PR #19859 (drip-121) added a hooks panel inside the *plugin* details view; this PR is the broader companion — a top-level `/hooks` browser that lists all discovered hooks across all sources (user config, project config, plugin) and exposes per-hook enable/disable + a "managed" indicator that distinguishes plugin-shipped hooks from user-authored ones.

## What it changes

The diff splits cleanly into three layers. **Protocol** (`app-server-protocol/src/protocol/v2.rs:4490-4493` + the four mirrored JSON schemas + the regenerated TS type at `schema/typescript/v2/HookMetadata.ts`): adds an `is_managed: bool` field to `HookMetadata` and includes it in the `required` array. **Engine** (`hooks/src/engine/mod.rs:74-77` + `discovery.rs:453-456`): adds the same `is_managed` field to `HookListEntry` and populates it from `source.is_managed` during discovery. The `mod_tests.rs:118-130` addition exercises `crate::list_hooks(...)` against a config layer and asserts the returned `is_managed` survives, plus a parallel addition at `:511-525` exercises plugin-source hooks and asserts `plugin_id` matches `"demo-plugin@test-marketplace"`. **TUI** (`tui/src/app/background_requests.rs:92-100` + `:212-225` + `app.rs:88-92`): two new background request handlers — `fetch_hooks_list` issues `ClientRequest::HooksList` with the current cwd via a UUID-stamped request ID, dispatches the result via `AppEvent::HooksLoaded { cwd, result }`; `set_hook_enabled` issues `write_hook_enabled` and dispatches `AppEvent::HookEnabledSet { result }`. The `hooks_list.rs:65-68` integration test pins `assert!(!hook.is_managed)` for the user-config-sourced fixture. A separate touch at `protocol/src/protocol.rs:1606-1609` adds `EnumIter` to `HookEventName`'s derive list (used by the new browser to enumerate event-name groups for display).

## Design analysis

Three things are right. First, the `is_managed` flag is sourced from `source.is_managed` at the discovery layer (`discovery.rs:455`) rather than being inferred at the TUI layer from the source enum — this is the correct location because `is_managed` is a property of *how the hook entered the engine*, not *how the UI wants to display it*, and UIs in other surfaces (CLI `hooks list`, JSON output) get the right answer for free. Second, the protocol-side `is_managed` lands in the `required` array in all three schema mirrors (`codex_app_server_protocol.schemas.json:9724`, `v2.schemas.json:6354`, `v2/HooksListResponse.json:107`), which is the strict version-bump signal — every external `hooks/list` consumer will fail their schema validation on a missing `isManaged`, which is the right behavior for a non-optional new field on an existing response shape. Third, the unit test addition at `mod_tests.rs:122-130` calls `crate::list_hooks` (the public API) rather than reaching into engine internals, which means the test will catch any future regression where the field is correctly populated in `HookListEntry` but lost on the way to the API surface.

## Concerns / nits

1. **Schema breakage callout missing** — the three JSON-schema mirrors all add `isManaged` to the `required` array, which is a breaking change for any external `hooks/list` consumer that hasn't yet been updated. The PR description should spell this out explicitly with a `BREAKING:` prefix (the codex repo's convention for protocol-level breaks). This is the same nit I raised on #19859 last drip and the same fix applies — the reviewer should ask for the description bump before merging.

2. **`EnumIter` derive on `HookEventName`** at `protocol/src/protocol.rs:1610` adds a `strum_macros` dependency on a public protocol type. That's fine for now but means any consumer that derives a `match` on `HookEventName` and reads it via `serde` will silently grow new variants when the enum grows — which is the *intent* (the new browser uses it to enumerate display groups), but it's worth a comment at the derive site naming the load-bearing consumer (the TUI `/hooks` browser group enumeration) so a future refactor that wants to move `HookEventName` doesn't accidentally drop the derive.

3. **`ClientRequest::HooksList` is dispatched per `cwd`** at `background_requests.rs:103` with a fresh UUID-stamped request ID per call. That's correct for ordering safety, but the `AppEvent::HooksLoaded { cwd, result }` payload at `:99` only carries the cwd, not the request ID — meaning if the user navigates rapidly between projects, two in-flight requests for the same `cwd` will both deliver and the UI will accept whichever arrives last. That's probably fine in practice (the result is the same), but if a future `HooksList` learns to take per-request filter args, this needs to grow request-ID-based de-duplication.

4. **`write_hook_enabled` swallows the OK value** at `:218-220` (`.map(|_| ())`) — fine for the toggle UX (the UI just needs to know "did the write land"), but if a future state-machine wants to reconcile the persisted state against the optimistic UI state, this discards the source of truth. Worth a comment.

## Risks

Medium-low. The protocol break is real but expected (this is what `is_managed: bool` `required` does). The TUI background-request layer is well-isolated from the engine's `discovery.rs` change and the test coverage at the engine layer (`mod_tests.rs:122-130` and `:511-525`) pins both the user-source and plugin-source paths, so the wire-up is type-safe end-to-end. The `EnumIter` derive on `HookEventName` doesn't change the wire format (it's a Rust-side trait derive only, not a serde attribute change).

## Suggestions

Bump the PR description with a `BREAKING:` callout for the new required field, add a one-line comment at the `EnumIter` derive naming the TUI consumer, and (optional) thread a request-ID through `AppEvent::HooksLoaded` for symmetry with the rest of the background-request surface. None of these block — all three are small.

## What I learned

The "promote a discovery-time property all the way out to the API surface, even if the UI is the only current consumer" pattern is the right one — it pays off the *next* time someone needs the same property in a different surface (CLI, JSON, log), at the cost of a one-time `required` field bump on the existing API. Worth the cost when the field is genuinely a property of the underlying entity, not a presentation choice.
