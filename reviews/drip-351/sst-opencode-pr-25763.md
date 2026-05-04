# sst/opencode PR #25763

- **Title**: fix(provider): surface OpenAI nested error.message in parseAPICallError
- **Author**: anandgupta42
- **Head SHA**: `dce8aa4c265b9da558c1905c8cbe4eb6bbad3890`
- **Diff**: +118 / -5 (2 files)

## Summary

Fixes a long-standing UX bug where OpenAI 4xx errors that ship `{error: {message, type, code}}` were dumped to the user as raw JSON instead of the human-readable `error.message`. The original chain `body.message || body.error || body.error?.message` short-circuited on the truthy `body.error` object, then failed the `typeof errMsg === "string"` guard, falling through to the generic body dump.

## File-by-file

### `packages/opencode/src/provider/error.ts` (lines 5-27)

Replaces the `||`-chain with explicit per-level `typeof === "string"` guards in priority order: `body.error.message` → `body.message` → `body.error` (string). This is the right structural fix — nested-message-first matches what OpenAI/Anthropic actually return, and the per-level typeof guard prevents a non-string at any one level from poisoning the chain. Comment at line 14-17 correctly notes alignment with `parseStreamError`.

Nit: the dropped `if (errMsg && typeof errMsg === "string")` had a defensive double-check; the new `if (errMsg)` is fine because every assignment branch is already string-guarded, but a one-liner asserting the union type stays `string | undefined` would future-proof against a sloppy edit.

### `packages/opencode/test/provider/error.test.ts` (lines 35-139, new file)

Four tests cover the regression matrix correctly:
- **L56-84**: the canonical OpenAI bug — verifies the structured body is no longer dumped (`not.toContain('"error":')`, `not.toContain("{")`).
- **L86-105**: defensive case where `body.error.message` is a non-string (array) — verifies `body.message` still wins.
- **L107-123**: fallback to `body.message` when `body.error` lacks a message.
- **L125-139**: plain `{error: "string"}` shape used by some providers.

Test scaffolding (`makeAPICallError`, L40-53) is minimal and reusable. Coverage is exactly the right shape for a typeof-guard regression.

## Risks

- The reordering changes priority: previously a top-level `body.message` would be tried before `body.error.message`. New order picks nested first. For OpenAI-shaped bodies this is strictly better, but if any provider returns *both* a top-level `message` and a stale nested `error.message`, behavior changes silently. Test at L107-123 only covers the case where nested is absent. Worth a one-liner in the PR description.

## Verdict

**merge-after-nits**
