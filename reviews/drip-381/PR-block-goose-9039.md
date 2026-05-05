# block/goose PR #9039 — add provider-first onboarding

- URL: https://github.com/block/goose/pull/9039
- Head SHA: `73bbd4f492c66b820cf911e96f795eb5aad3bc4a`
- Size: +3269 / -37 across ~50+ files (Rust ACP backend + goose2 React UI)

## Summary

Large new-feature PR introducing a provider-first first-run onboarding flow for the goose2 desktop UI, backed by typed ACP custom-method contracts on the Rust side. Adds five new Rust modules (`acp/server/onboarding.rs`, extends `acp/server/config.rs`, `acp/server/providers.rs`, `acp/server/custom_dispatch.rs`, ACP schema regen) and a full `features/onboarding/` React feature directory (hooks: `useOnboardingFlow`, `useOnboardingGate`, `useOnboardingImportStep`, `useOnboardingProviderStep`, `useOnboardingReadyStep`; UI: `OnboardingFlow`, `ProviderStep`, `ImportStep`, `ReadyStep`; lib: `importCounts`, `providerDefaults`).

## Specific findings

### Rust / ACP backend

- `crates/goose-sdk/src/custom_requests.rs` — adds typed ACP request/response shapes for `_goose/defaults/save`, `_goose/onboarding/import/scan`, `_goose/onboarding/import/apply`, and provider native-auth methods. Typed contracts (vs free-form JSON) are the right call for an SDK boundary.
- `crates/goose/acp-meta.json` and `crates/goose/acp-schema.json` — auto-regenerated. As long as the regeneration is reproducible from the typed Rust sources, this is safe; a maintainer should confirm the regen command and that the diff matches what `cargo run --bin acp-schema-gen` (or equivalent) emits.
- `crates/goose/src/acp/server/config.rs` — implements `_goose/defaults/save` with provider/model validation before writing Goose defaults. **Concern:** the diff size suggests this is more than a simple write — need to verify the validation rejects unknown providers (vs silently accepting a typo'd provider id and writing it to disk where every subsequent boot will fail to load the model).
- `crates/goose/src/acp/server/onboarding.rs` — backend scan/apply for existing Goose config + Claude Desktop MCP tools. PR description claims "duplicate-safe extension and skill import behavior" — the duplicate-detection key matters: if it's keyed on `name` only, two extensions with the same display name from different sources will collide; if it's keyed on `(source, name)` it's correct. Need to confirm in source.
- `crates/goose/src/acp/server/providers.rs` — provider-owned native authentication through ACP and inventory refresh after auth completes. Native-auth (vs HTTP-redirect-back-to-localhost) is the right model for a desktop app.
- `crates/goose/src/providers/chatgpt_codex.rs` — marks ChatGPT/Codex (note: this is the upstream "vsc-redacted-acp" product per our review-notes redaction policy) configured when the OAuth token cache exists. Coupling onboarding state to token-cache presence is a reasonable proxy for "user has authed."
- `crates/goose/tests/acp_custom_provider_methods_test.rs` — new integration test for the custom-method dispatch surface. Good signal that the new typed contracts have at least round-trip coverage.

### React / goose2 UI

- `ui/goose2/src/features/onboarding/hooks/useOnboardingGate.ts` + `useOnboardingGate.test.tsx` — gate hook with explicit test coverage. Pinning the gate logic (when does the onboarding flow trigger? when does it dismiss?) with tests is the right call since gate bugs are extremely user-hostile (either you re-prompt onboarded users or you skip onboarding for new users).
- `ui/goose2/src/features/onboarding/ui/OnboardingFlow.tsx` — orchestrator. Standard step-machine pattern based on the per-step hooks (`useOnboardingProviderStep`, `useOnboardingImportStep`, `useOnboardingReadyStep`).
- `ui/goose2/src/features/onboarding/ui/ProviderStep.tsx` / `ImportStep.tsx` / `ReadyStep.tsx` — three-step funnel: pick provider → import existing tools → ready to chat.
- `ui/goose2/src/features/providers/api/credentials.ts` + `credentials.test.ts` — shared provider credential plumbing with test coverage. Centralizing credential handling (vs each provider integration rolling its own) is correct for a security-sensitive surface.
- `ui/goose2/src/app/AppShell.tsx` and `useAppStartup.ts` — wire the gate into the app startup. Need to verify the gate is checked *before* any auto-load that would assume a configured provider, otherwise new users see a flash of error states before the onboarding overlay renders.
- `ui/goose2/src/features/onboarding/lib/providerDefaults.ts` and `importCounts.ts` — pure utility modules; should have unit-test coverage commensurate with the gate hook's. Not visible in the file list above the truncation cutoff — worth a check.

## Concerns

1. **Diff size:** +3269 lines is right at the edge of "single PR" territory. Splitting into (a) ACP custom-method contracts, (b) Rust backend handlers, (c) React onboarding feature would each be reviewable in isolation. As one PR, full review requires several hours.
2. **Schema regen reviewability:** `acp-meta.json` and `acp-schema.json` are auto-generated; the diff is hard to review by eye. Reviewer should re-run the regen command locally and confirm zero diff.
3. **Validation surface:** `_goose/defaults/save` validation is the new write path for provider/model defaults. If a malformed default lands on disk (e.g., from a UI bug in `ProviderStep.tsx` that submits an empty model id), every subsequent app launch will hit it. Defense in depth on both the API handler and the UI submit is warranted.
4. **No CHANGELOG entry visible.** First-run onboarding is a major user-facing change for goose2 — definitely needs a changelog and a release-notes screenshot.
5. **Native-auth thread safety:** the provider native-auth flow at `providers.rs` likely spawns a tokio task and waits on a one-shot channel. If the user closes the onboarding window mid-auth, the task needs to be cancellable; if the user retries auth, the previous oneshot needs to be dropped cleanly. Not visible from the file list — needs source-level review.

## Verdict

`needs-discussion` — the feature is clearly valuable and the architecture (typed ACP contracts + per-step hooks + dedicated provider credential module) is sound, but the diff is large enough and touches enough security-relevant surface (credential storage, OAuth token-cache, defaults persistence) that a maintainer should request a split into 3 PRs and confirm the validation/native-auth concerns before merge.
