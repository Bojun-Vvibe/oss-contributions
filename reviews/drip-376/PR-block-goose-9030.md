# block/goose #9030 — feat: move goose2 provider catalog behind ACP layer

- Author: kalvinnchau
- Head SHA: `57344ca4c79a5944f3a5b2bd28a82bebe590cdf6`

## Files changed (top)

- `crates/goose/src/providers/catalog.rs` (+861 / -1) — backend canonical catalog
- `ui/goose2/src/features/providers/providerCatalog.ts` (+90 / -39) — UI shim becomes thin
- `ui/goose2/src/features/providers/providerCatalogEntries.ts` (-481) — deleted UI duplication
- `ui/goose2/src/features/providers/stores/providerCatalogStore.ts` (+99 / -0, new) — zustand store
- `crates/goose-sdk/src/custom_requests.rs` (+83 / -2) — DTO + rename
- `crates/goose/acp-schema.json` (+183 / -2) — schema regen
- Plus broad UI hook/component sweep migrating `getCatalogEntry()` → `getCatalogEntryFromEntries(catalogEntries, …)`

## Observations

- `crates/goose-sdk/src/custom_requests.rs:430` — renames `ProviderCatalogEntryDto` → `ProviderTemplateCatalogEntryDto`. **Breaking change** for any external SDK consumer that imports the old name. The rename is correct semantically (this struct is specifically for *template*-style providers, distinct from the new setup catalog), but downstream impact is real. The accompanying mass-rename across UI files (`customProviders.ts:18`, `customProviderTypes.ts:8/56`, `useCustomProviders.ts:28/49`) suggests the maintainers intentionally accept the breakage. Worth calling out as a SemVer-major change in release notes.
- `crates/goose/src/providers/catalog.rs:113-180` — new types `ProviderSetupCategory`, `ProviderSetupMethod`, `ProviderSetupGroup`, `ProviderSetupField`, `ProviderSetupCapabilities`, `ProviderSetupCatalogEntry` form a clean canonical model. The `#[serde(rename_all = "snake_case")]` on the enums correctly matches the JSON wire format and the `#[serde(default, skip_serializing_if = "Option::is_none")]` discipline on optional fields keeps the wire payload lean.
- `ui/goose2/src/features/providers/stores/providerCatalogStore.ts:39-47` — `GOOSE_PROVIDER_CATALOG_ENTRY` hardcoded fallback at the *store* level is the right call: ensures the agent picker always has at least one usable entry even if the ACP catalog list call fails or hasn't returned yet. The `withGooseFallback` helper at `:14-21` makes the merge logic explicit.
- `ui/goose2/src/features/providers/stores/providerCatalogStore.ts:56-60` — `loadPromise` module-level singleton dedupes concurrent `load()` calls during startup. Good pattern for "fetch once on app boot" semantics.
- `ui/goose2/src/features/chat/hooks/useResolvedAgentModelPicker.ts:62-79` — refactor wraps the previously-unmemoized derivations in `useMemo` with `[catalogEntries, selectedProvider]` deps. Necessary because the catalog now comes from a zustand store rather than a static module import; without memoization, every render would re-walk the full catalog. Correct.
- `ui/goose2/src/features/providers/hooks/useProviderInventory.ts:24-26` — `if (!catalogLoaded) { return true; }` fallback for `isConfiguredGooseModelProvider` correctly handles the startup race where inventory arrives before catalog. Without this, every configured provider would be filtered out for ~100ms on cold start. Good defensive design.
- `crates/goose/tests/acp_custom_provider_methods_test.rs:73-115` — integration test asserts presence of {goose, anthropic, openai, claude-acp, codex-acp, vsc-redacted-acp, amp-acp, cursor-agent, pi-acp} and absence of deprecated {codex, claude_code, gemini_cli}. This is exactly the right shape of test for a catalog-migration PR — pins both the inclusions and the deletions.
- `ui/goose2/src/features/providers/lib/providerKey.ts:1-8` — `normalizeProviderKey` extracted into its own file with a slight regex improvement (`/[^\p{L}\p{N}]+/gu` Unicode-aware vs. the prior `/[-_\s]+/`). The new regex correctly handles non-ASCII provider names (e.g., a hypothetical `mistral-é`) where the old one would have left punctuation. Subtle behavior improvement.
- `ui/goose2/src/features/providers/distroProviderConstraints.test.ts:12/22/32` — test data updated from `tier: "promoted"` to `group: "default"`, reflecting the field rename in the new DTO. Mechanical but worth noting: any external code reading the old `tier` field is broken.
- The PR is genuinely large (+1500/-600 across ~30 files). The split between backend (~+1100) and frontend (~+400) is clean, and no single file exceeds reasonable review surface. The mechanical rename sweep dominates the line count.

## Verdict

`merge-after-nits`

Architecturally sound consolidation moving the source-of-truth provider catalog from frontend-duplicated TS to backend-canonical Rust, with the UI becoming a thin async-loaded reflection. The fallback discipline (`GOOSE_PROVIDER_CATALOG_ENTRY` hardcode, `catalogLoaded` race guard) is solid. Two nits: (1) the `ProviderCatalogEntryDto` → `ProviderTemplateCatalogEntryDto` SDK rename is a breaking change deserving an explicit release-notes callout, and (2) the `tier` → `group` field rename will silently break any external code reading the old field — also worth a release-note mention. Neither is a code defect; both are user-facing migration concerns.
