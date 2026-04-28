# PR #8887 — refactor: consolidate provider credentials into the acp backend

- **Repo:** block/goose
- **Link:** https://github.com/block/goose/pull/8887
- **Author:** kalvinnchau (Kalvin C)
- **State:** OPEN
- **Head SHA:** `5595c0e3294e75e48621df50902ede701929640d`
- **Files:** 35 changed (+2795/-1007)

## Context

goose2 was running two parallel credential systems: a Tauri-side service (`ui/goose2/src-tauri/src/services/goose_config.rs`, 452 lines, deleted) that read the keyring directly from the desktop binary, and the core-side config in `crates/goose/src/config/base.rs`. That meant *both* binaries had to hold OS keychain entitlements — duplicate credential handling, duplicate keychain prompts, and two places to keep in sync when adding providers. This PR collapses that into ACP-only: provider config lives in core, the desktop UI calls four new ACP methods.

## What changed

Four new ACP methods on the server (`crates/goose/src/acp/server.rs` +544/-101):
- `_goose/providers/config/read`
- `_goose/providers/config/status`
- `_goose/providers/config/save`
- `_goose/providers/config/delete`

Schema/meta files updated (`acp-meta.json` +20, `acp-schema.json` +232) and an SDK shim (`crates/goose-sdk/src/custom_requests.rs` +90, `ui/sdk/src/generated/*` +209). Tauri-side: deletion of `commands/credentials.rs` (-50), `services/goose_config.rs` (-452), `services/provider_defs.rs` (-150), and a small unwiring in `lib.rs` (+1/-8). Frontend pivots to the new SDK calls (`useCredentials.ts` +183/-32, new `inventorySync.ts` +115, `ModelProviderRow.tsx` +46/-10) with three new test suites (~500 lines combined).

Core changes worth flagging:
- `config/base.rs` (+104/-57) gains batched secret writes/deletes — important for keychain perf because each individual write triggers an OS prompt on locked keychains.
- `providers/inventory/mod.rs` (+158/-42) hardens the refresh job: each refresh now captures the credential identity at *plan* time, results are stored under that captured identity, and stale-credential refreshes are rejected before the model fetch runs. This is the right shape for a refresh job that races against credential mutation — without it you can write fresh model lists into the wrong provider's slot when credentials are rotated mid-refresh.
- A 343-line integration test `crates/goose/tests/acp_secret_cache_invalidation_test.rs` exercises the cache-invalidation contract end-to-end.

## Design analysis

The architecture is correct: the long-term direction for desktop wrappers is "thin shell over ACP," and credentials are exactly the kind of cross-cutting state that should not be split between the shell and the engine. The deletion-heavy diff (1007 lines deleted; whole files removed) is the strongest signal of correctness — duplicate code is going away, not just being shuffled.

The refresh-identity-capture pattern in `inventory/mod.rs` is the most subtle and most important piece. The bug it prevents is: provider A has creds, refresh job fetches A's models, user swaps A to credential B, the in-flight job completes and writes A's model list into B's slot. The fix — capture identity at plan time, reject if mutated, drop result on mismatch — is the standard "compare-and-swap on an identity token" pattern. The acp_secret_cache_invalidation_test should specifically cover this race. If it doesn't, that's a gap.

The batched secret writes in `base.rs` are pragmatic — saving five fields for a provider config (api_key, base_url, org_id, etc.) shouldn't pop the keychain prompt five times. Worth confirming the batch is atomic: if write 3 of 5 fails, do the first two persist or get rolled back? Half-saved provider configs are a worse user experience than no save at all, because the next read returns a partially valid config and behaviour gets weird.

## Risks

1. **Cross-binary keychain entitlement migration.** Removing the Tauri-side keychain access means existing installations may have credentials persisted under the *Tauri* keychain identity that the core binary can't read. The PR doesn't mention a migration step. If the core process and the Tauri process previously wrote to different keychain service names, every existing user gets a "please re-enter your API keys" experience on upgrade. Worth a one-shot migration on first launch, or at minimum a release note.

2. **SDK regen drift.** Generated SDK files (`ui/sdk/src/generated/*`) are checked in. If a contributor regenerates with a different generator version they'll see a noisy diff. The `acp-schema.json` +232 change should drive the SDK regen — confirm CI verifies they're in sync.

3. **Localisation gaps.** `en/settings.json` +3/-2 and `es/settings.json` +2/-2 update only two locales. Other locales still in the repo (likely zh, ja, etc. given the project's reach) will fall back to English for the new strings. Either include them, or add a CI check for translation parity.

4. **Test scope.** 35 files, 4 new ACP methods, race-condition fix in inventory refresh, batched secret API, full Tauri-side deletion. The test plan covers the new ACP method via the cache-invalidation test, the React component via Vitest, and `pnpm typecheck` + `cargo fmt`. Missing: an end-to-end test that walks the actual goose2 binary through "set credential → refresh → mutate → refresh again" to validate the desktop integration, not just the units. The PR says the desktop test is `just tauri-check`, which is mostly a build sanity check, not a behavioural test.

## Verdict

**Verdict:** request-changes

Architecture is right and the deletion-heavy shape is encouraging. Blocking asks: (1) document or implement the keychain credential migration for existing installs, (2) confirm batched secret writes are atomic on partial failure, (3) ensure the cache-invalidation integration test specifically covers the credential-rotation-mid-refresh race that the inventory hardening is designed to fix. With those addressed, this is a high-quality refactor.

---

*Reviewed by drip-155.*
