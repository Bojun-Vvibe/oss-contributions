# QwenLM/qwen-code PR #3633 — revert(cli): undo OPENAI_MODEL precedence change in modelProviders lookup

- **Repo:** QwenLM/qwen-code
- **PR:** [#3633](https://github.com/QwenLM/qwen-code/pull/3633)
- **Closes:** #3567
- **Files:** 2 files, +0/−283 (net deletion)

## Context

This is a revert PR. An earlier change at `modelConfigUtils.ts:100-108`
added a `USE_OPENAI`-specific precedence chain
(`argv.model || env.OPENAI_MODEL || env.QWEN_MODEL || settings.model.name`)
when looking up a `ProviderModelConfig` by id within the
`modelProviders[USE_OPENAI]` array. That broke the `/model` interactive
selection flow because users who had `OPENAI_MODEL` exported in their
shell would have their interactive selection silently overridden on
config resolution. PR #3645 (drip-80) was the proper test-matrix-first
re-fix; this PR is the immediate revert that restores the old behavior.

## What the diff actually does

**`modelConfigUtils.ts:100-108`** restores the pre-regression code:
```diff
-      const requestedModel =
-        authType === AuthType.USE_OPENAI
-          ? argv.model ||
-            env['OPENAI_MODEL'] ||
-            env['QWEN_MODEL'] ||
-            settings.model?.name
-          : argv.model || settings.model?.name;
+      // Try to find by requested model (from CLI or settings)
+      const requestedModel = argv.model || settings.model?.name;
```

**`modelConfigUtils.test.ts`** (−274 lines) deletes four tests that
locked the reverted behavior:
- `'should find modelProvider from OPENAI_MODEL when argv.model is not provided'`
- `'should find modelProvider from QWEN_MODEL when OPENAI_MODEL is not provided'`
- `'should prefer OPENAI_MODEL over QWEN_MODEL and settings.model.name for USE_OPENAI provider lookup'`
- `'should ignore OPENAI_MODEL for non-USE_OPENAI provider lookup'`
- `'should respect precedence: argv.model > OPENAI_MODEL > QWEN_MODEL > settings.model.name'`

All five test deletions are mechanical mirrors of the production-code
revert.

## Strengths

- **Minimum-surface revert.** The test-side deletions are exactly
  the tests that locked the reverted behavior, no collateral test
  removal.
- **Restores `/model` interactive selection** — the bug being
  reverted is user-facing (the interactive picker was being
  shadowed by `$OPENAI_MODEL` from the user's shell), so reverting
  fast unblocks every user with `OPENAI_MODEL` exported.
- **Coordinates with #3645** — that PR (drip-80) is the proper
  re-fix that adds the precedence back via a test-matrix-first
  approach where `settings.model.name` is checked against
  `modelProviders[selectedAuthType]` *before* falling back to
  `OPENAI_MODEL`. This revert sets the baseline cleanly so #3645
  has a known starting point.
- **Comment-anchored** — the new code at `modelConfigUtils.ts:103`
  includes a brief comment `// Try to find by requested model
  (from CLI or settings)` that documents the deliberately-narrow
  scope.

## Risks / nits

1. **No CHANGELOG entry** — users who built workflows around the
   `OPENAI_MODEL` env-var precedence (introduced and now reverted
   in the same week) need to know. A one-line note in
   `CHANGELOG.md` saying "Reverted #3567's `OPENAI_MODEL`
   precedence change; use `--model` or `settings.model.name`
   instead. See #3645 for the proper fix" would close the loop.
2. **`/model` interactive flow regression test missing.** This
   revert deletes the unit tests but doesn't add an integration
   test that asserts "user runs `/model x`, then a turn executes,
   model x is what's used regardless of `OPENAI_MODEL`". Without
   that test, the original bug (interactive selection silently
   shadowed) can re-regress on the next refactor.
3. **No `git revert` provenance.** The PR description should call
   out which commit SHA is being reverted so reviewers can check
   the revert is faithful. Right now the title says
   `revert(cli): undo …` but there's no `Reverts: <sha>` trailer.
4. **`#3567` tracking issue should be re-opened.** Reverting the
   fix doesn't fix the underlying issue — it just removes a bad
   fix. Whatever motivated #3567 (presumably "I want
   `OPENAI_MODEL` to drive provider lookup") still wants
   addressing, which is what #3645 does. Make sure #3567 is
   re-opened or linked from #3645.
5. **`Settings Model` provider id collision** — the deleted tests
   were using `id: 'settings-model'` to assert that
   `settings.model.name` was the lookup key. The reverted code
   keeps that behavior, but with the broader fix in #3645
   landing, the test fixture shape changes again. Make sure the
   merge order is `#3633 (this) → #3645` so the tests #3645
   adds match the production code at that point.

## Verdict

`merge-as-is`

Clean revert with proportional test deletion. The bug being
reverted (interactive `/model` selection silently shadowed by
`$OPENAI_MODEL`) is bad enough to justify reverting first and
landing the proper fix in #3645 second. Two small follow-ups
should be added by the maintainer before tagging the next release:
a `CHANGELOG.md` line explaining the round-trip, and a
`/model` end-to-end test in #3645 to lock the contract long-term.
Not blocking on this PR.
