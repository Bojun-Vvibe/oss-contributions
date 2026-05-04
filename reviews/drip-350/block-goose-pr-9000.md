# block/goose#9000 — feat(acp): replace raw config and secret methods

- **URL**: https://github.com/block/goose/pull/9000
- **Head SHA**: `79f11672aca3`
- **Diffstat**: +1381 / -649 (37 files)
- **Verdict**: `needs-discussion`

## Summary

Removes the generic `_goose/config/{read,upsert,remove}` and `_goose/secret/{check,upsert,remove}` ACP methods and replaces them with typed, allowlisted endpoints: `_goose/preferences/{read,save,remove}`, default provider/model reads, provider configuration saves, extension config writes, and dictation secret management. Regenerates the ACP schema and SDK client; updates both `ui/goose2` (React) and `ui/text` (terminal) callers.

## Findings

### API surface
- `crates/goose/acp-meta.json` (+20 / -20) and `crates/goose/acp-schema.json` (+132 / -104) — full ACP method swap. This is a **hard breaking change** for any out-of-tree ACP client (third-party UIs, integrations, scripts). The PR body's verification step 1 explicitly says callers should confirm the old methods "are no longer exposed" — so deprecation/coexistence is not on offer. There is no compatibility shim. This needs an explicit maintainer call on:
  1. SemVer impact — does goose's ACP surface have a stability promise?
  2. Whether to ship a one-release window where both old and new methods coexist (old emitting a deprecation warning), instead of a hard cut.
- `crates/goose/src/acp/server/secrets.rs` (-32) is fully deleted. Anyone holding a third-party ACP client that calls `_goose/secret/upsert` directly will break the moment they update.

### Implementation
- `crates/goose/src/acp/server/config.rs` (+135 / -16) — implements the allowlisted preference layer. Allowlisting is the right call security-wise; the question is whether the allowlist is centralized (single source of truth) or scattered. Reviewer should confirm there's one canonical `SUPPORTED_PREFERENCES` constant rather than per-call enumerations.
- `crates/goose/src/acp/server/dictation.rs` (+47 / -1) — adds typed dictation secret save/delete with provider-support validation and post-write cache invalidation. Good — invalidation is the kind of thing that's easy to forget and the PR body explicitly cites the `acp_secret_cache_invalidation_test` update.
- `crates/goose/src/config/base.rs` (+14 / -0) — adds a batch config write helper. Good for atomicity of multi-key preference saves; reviewer should confirm it actually wraps writes in a single guarded read/write cycle (the PR body claims this) and isn't just a thin loop.

### Test coverage
- `crates/goose/tests/acp_custom_requests_test.rs` (+256 / -20) — substantial new coverage for typed preferences, defaults, validation failures, and the migration off the removed generic methods. Healthy.
- `ui/goose2/src/features/chat/hooks/__tests__/useVoiceInputPreferences.test.ts` (+133 / -17) and `useAutoCompactPreferences.test.ts` (+66 / -15) — solid frontend coverage for the new API shape.
- `ui/goose2/src/features/settings/ui/__tests__/VoiceInputSettings.test.tsx` (+98 / -0) — net-new file. Good.

### Generated code
- `ui/sdk/src/generated/{client.gen.ts,types.gen.ts,zod.gen.ts,index.ts}` — regenerated. Reviewer should confirm generation is reproducible from the updated `acp-schema.json` (i.e. re-running the codegen command produces an identical diff). Drift between hand-edited generated files and schema is a long-tail debugging tax.

### Scope
- Touching 37 files in one PR — Rust server, schema, generated SDK, React UI, terminal UI, i18n strings — makes bisecting a breakage hard. If a maintainer wanted to split this, the natural seam is: (1) add new typed methods alongside old ones, (2) migrate UIs, (3) remove old methods. As written it's all-or-nothing.

## Recommendation

`needs-discussion`. The implementation looks careful and well-tested, but the hard removal of `_goose/config/*` and `_goose/secret/*` is a breaking ACP-surface change that deserves an explicit maintainer decision on deprecation policy and possibly a phased rollout. Once that's settled, the PR is in good shape.
