# PR #3682 — fix(core,cli): stop stripping reasoning on model switch / history load

- **Repo**: QwenLM/qwen-code
- **PR**: #3682
- **Head SHA**: `7635f8c1b9d8e5275681023ad2c48d7a2dd9c5cd`
- **Author**: fyc09 (Fu Yuchen)
- **Size**: +7 / -10 across 4 files (related: #3579, #3304, #3619)
- **Verdict**: **needs-discussion**

## Summary

Backs out two `stripThoughtsFromHistory()` calls that were
previously added as event-side fixes for #3304: one inside the
`/load_history` slash command path
(`slashCommandProcessor.ts:649`) and one inside `Config.switchModel`
(`config.ts:1426`). The PR also flips the corresponding regression
tests to assert *preservation* rather than stripping, and rewrites
the `config.ts` comment from "strip thinking blocks because
strict OpenAI-compatible providers reject reasoning_content with
422" to "preserve full history because some OpenAI-compatible
reasoning models (DeepSeek) require reasoning_content to be
preserved across turns."

## Specific changes

- `packages/cli/src/ui/hooks/slashCommandProcessor.ts:649` —
  removes the single line
  `config?.getGeminiClient()?.stripThoughtsFromHistory()` from
  the `'load_history'` slash-command branch. Net effect: replaying
  a saved session now keeps thought parts intact.
- `packages/cli/src/ui/hooks/slashCommandProcessor.test.ts:504,533`
  — renames the test from
  `'should strip thoughts when handling "load_history" action'` to
  `'should preserve thoughts when handling "load_history" action'`
  and flips the assertion from
  `expect(mockClient.stripThoughtsFromHistory).toHaveBeenCalledWith()`
  to `.not.toHaveBeenCalled()`.
- `packages/core/src/config/config.ts:1423-1428` — drops the
  unconditional `this.geminiClient.stripThoughtsFromHistory()`
  call inside `switchModel(...)` and replaces the original
  comment block with the new rationale (preserve for DeepSeek
  and other reasoning-content-required providers).
- `packages/core/src/config/config.test.ts:533, 569` — renames
  `'should strip thoughts from history on model switch (#3304)'`
  to `'should preserve thoughts from history on model switch'`
  and flips `stripSpy` assertion to `.not.toHaveBeenCalled()`.

## Why this is `needs-discussion`

The PR is a **policy reversal**, not a bug fix. The original
strip-on-event change at `config.ts:1426` was added explicitly
to fix #3304 — strict OpenAI-compatible providers returning 422
when `reasoning_content` from a previous model leaks into the
request payload. Removing it without addressing the original
failure mode means #3304-class providers will now fail again.
The PR body acknowledges this in the Risk section
("Strict provider wire-format compatibility is not handled by
history mutation anymore") and points at "provider-specific
request-boundary sanitization work is not part of this PR" —
which means **this PR is half of a fix and the other half is
unscoped**.

The core trade-off:

- **Keep stripping (status quo)**: DeepSeek / reasoning-content-required
  providers degrade because their multi-turn quality drops when
  prior reasoning is dropped from history. That's the bug the
  PR fixes.
- **Stop stripping (this PR)**: strict OpenAI-compat providers
  regress to the original #3304 422 failure unless a *separate*
  request-boundary sanitization layer is added before the
  request hits the wire.

The right shape is to **defer history mutation entirely and do
the filtering at the request-build boundary**, keyed off the
target provider's tolerance for `reasoning_content`. That way
history is the source of truth, and each provider gets the
shape it can accept. The PR description says that work "is not
part of this PR" — but without it, this PR substitutes one
class of regression for another.

## Specific concerns

1. **#3304 is closed but not solved.** Linking it as "Related"
   while removing the fix is a regression-by-omission. Either
   #3304 needs to be reopened with an explicit "this is now
   addressed at the provider request boundary in PR #XXXX" or
   this PR needs to land *with* that boundary fix.
2. **No matrix test for the `(provider, has_reasoning_content)`
   product.** The flipped assertions only check that the strip
   *helper* isn't called; they don't check that
   strict-provider request bodies don't contain
   `reasoning_content`. Without the request-boundary
   sanitization, "the helper isn't called" is the wrong
   invariant.
3. **The test rename pattern is concerning.** When a test named
   `should strip thoughts ... (#3304)` becomes `should preserve
   thoughts ...`, the issue number is being orphaned. Either
   the test should be renamed to point at the *new* governing
   issue (#3579, #3619) or the original issue tag should
   remain with a comment explaining the change in policy.
4. **Two call sites, one decision.** `/load_history` and
   `switchModel` are removed in lockstep. They're conceptually
   the same — "we're rebinding model context, sanitize history
   first" — but they could have different right answers.
   Loading a *saved* session containing thoughts is arguably
   safer to preserve (the thoughts came from the same model
   that wrote them); switching the *active* model mid-session
   is where the cross-model contamination originally surfaced.
   The PR treats them identically without justifying that.

## Risks

- **Regression of #3304** for any user on a strict
  OpenAI-compatible endpoint that doesn't accept
  `reasoning_content` (the original bug).
- **Silent payload bloat** for any provider that accepts but
  doesn't *use* `reasoning_content` — the field is kept across
  turns and accumulates, increasing token cost.
- **Test renames lock in a behavior contract that the PR has
  not justified is the new contract.** If the policy needs to
  flip back (or be made provider-conditional), the tests now
  actively work against that.

## Verdict

`needs-discussion` — the underlying observation (DeepSeek
requires reasoning preservation) is real, but the fix shape is
wrong: history shouldn't be mutated to make wire-format
compatible. The maintainer-decision reference to #3579 should
be quoted directly in the PR body so reviewers can see the
explicit "strip-on-event was identified as the wrong fix
direction" guidance, *and* a follow-up PR (or this one,
expanded) needs to land the request-boundary sanitization
before this can merge without re-opening #3304.

## What I learned

"History mutation as wire-format compatibility shim" is a
recurring antipattern in agent CLIs that talk to multiple
providers — it conflates the user-visible conversation state
with the provider-visible request payload, and the resulting
fix-then-revert ping-pong (#3304 strip → DeepSeek breaks → this
PR un-strip → #3304 returns) is the canonical signature. The
durable shape is a per-provider request-shape filter applied at
serialization time, with history kept canonical. The harder
question — "which fields does each provider tolerate / require?"
— becomes a static config table, not a dynamic mutation policy.
