---
pr: https://github.com/sst/opencode/pull/24574
sha: bea31ab28087ebdc6242dc1ab5a70bc7df7570c3
diff: +22/-1
state: MERGED
---

## Summary

Splits the previously-shared empty-content normalization branch in `normalizeMessages()` so that the Anthropic and Bedrock SDKs each get their own copy of the filter, in preparation for the two providers diverging on what counts as "empty" or "valid" content. Functionally a no-op for the SHA itself — the two cloned blocks are byte-for-byte identical — but it removes the `||` coupling that would otherwise make any future Bedrock-specific tweak silently change Anthropic behavior (or vice versa).

## Specific observations

- `packages/opencode/src/provider/transform.ts:55` — the predicate flips from `model.api.npm === "@ai-sdk/anthropic" || model.api.npm === "@ai-sdk/amazon-bedrock"` to two consecutive `if` blocks (Anthropic at `:55`, Bedrock at `:75` per the diff). Both blocks run for their respective providers; there is no shared `if/else if` so a future provider that legitimately matches both names would get filtered twice — not a real concern given the npm package names are disjoint, but worth a one-line comment.
- The duplicated body is identical down to the `(msg): msg is ModelMessage => msg !== undefined && msg.content !== ""` type predicate. No tests added or modified — acceptable given the diff is mechanical and existing coverage of the combined branch must already exercise both providers, but a one-line snapshot pinning "Anthropic and Bedrock both filter empty array parts" would lock the behavioral parity that this refactor is explicitly preserving.
- The PR title prefix `ignore:` (rather than `refactor:` or `chore:`) suggests the author is intentionally signaling this should not appear in the changelog — matches the no-functional-change reality.

## Verdict

`merge-as-is` — surgical decoupling refactor with no behavior change, low blast radius (single function, two SDK-specific branches), and zero risk to the empty-content normalization invariant that the original combined branch was enforcing.
