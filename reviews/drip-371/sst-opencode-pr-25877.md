# sst/opencode PR #25877 — fix: let provider model hooks see config providers

- URL: https://github.com/sst/opencode/pull/25877
- Head SHA: `d6e922633d72057637aa11839e2db3fa3d55049b`
- Author: alceops (Alce)
- Closes: #25630
- Files: `packages/opencode/src/provider/provider.ts` (+27/-27, pure block move)

## Assessment

This is the smallest possible fix for a real ordering bug: plugin `provider.models()` hooks were running before `cfg.provider` was merged into `database`, so any hook targeting a user-defined config provider hit the `if (!provider) continue` early-out at the deleted `provider.ts:1148-1149` and silently no-op'd. The fix at `provider.ts:1236-1262` simply moves the hook loop from before the config-merge block to after it, while staying before the env/auth/custom-provider loaders that come further down. That ordering is the correct invariant: hooks need the full *catalog + config* provider set as input, but should run before runtime auth/env mutation overlays them.

The diff is a literal block move with zero logic changes (verified line-by-line: the hook iteration body, `Effect.promise` wrapping, `pluginAuth = yield* auth.get(providerID)`, and the `Object.fromEntries` model normalization are byte-identical). That makes the fix easy to reason about and trivially revertible. The author flagged that they couldn't run the test suite because `bun` isn't installed in their WSL runtime — that's a real risk, but `git diff --check` passing plus the surgical block-move nature limits the blast radius.

One concern: the hook now runs *after* `database[providerID] = parsed` for env/auth-derived providers (the loop at `provider.ts:1262+`), but *before* the env-overlay loop at `provider.ts:1264+`. If any plugin hook expected to see env-merged provider objects (e.g., to inspect runtime auth state via `provider.options`), this move could break that. The `pluginAuth = yield* auth.get(providerID)` pattern suggests hooks were already explicitly fetching auth separately, so this is probably fine, but a maintainer who knows the plugin contract should confirm no in-tree plugin relies on the prior post-env state.

The closes-issue reference (#25630) and the focused nature of the change (one-block move, no API surface change, no new flags) make this a low-risk merge once a maintainer verifies the test suite and the plugin contract intent.

## Verdict

`merge-after-nits`
