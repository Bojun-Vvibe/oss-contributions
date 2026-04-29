# openai/codex PR #20123 — [rollout-tracer] Match analysis messages on encrypted id

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20123
- Head SHA: `a383efae1b`
- State: OPEN, +120/-46 across 2 files

## What it does

Loosens the rollout-trace reducer's reasoning-item identity check. Pre-PR: the reducer required that two reasoning-item observations sharing the same `encrypted_content` blob *also* have matching readable text/summary parts, and bailed with `"reasoning encrypted_content was reused with different readable content"` if they diverged. In practice the Responses API returns rich readable reasoning on completion but later request snapshots often replay only the encrypted blob (or a subset of the readable parts), so the strict check was firing on legitimate replays.

This PR:

1. **Drops `ensure_reasoning_consistency`** entirely (deleted at `:497-521`) — the per-item bail that ran on every snapshot/append.
2. **Loosens `reasoning_body_matches`** to compare *only* on the encrypted blob: `left_encoded == right_encoded` (`:623`), removing the readable-text co-equality requirement.
3. **Adds `incoming_readable_reasoning_supersedes_existing`** (`:664-678`): on merge, if the incoming reasoning has a strict-superset of readable parts (existing parts all appear in incoming, incoming has more), upgrade the stored body to the richer one. Otherwise keep existing.
4. **Replaces the bail in `merge_reasoning_body`** (`:632-633`) with the relaxed check + the supersede-on-richer rule.
5. **Renames the bail message** to `"reasoning item merge attempted with different encrypted_content identity"` so it can only fire on the genuine encrypted-id mismatch case (which would be a real bug).

Test surface: replaces `same_encrypted_reasoning_with_different_text_is_reducer_error` (which asserted the bail) with `encrypted_reasoning_upgrades_when_later_sighting_has_more_readable_body` (asserts merge + upgrade).

## Specific reads

- `reducer/conversation.rs:209-212` and `:365-368` — both `ensure_reasoning_consistency` call sites are deleted from the snapshot/append paths, leaving only `ensure_call_id_consistency`. That's the right surgery: the function was the source of the bail, and it duplicated work that `merge_reasoning_body` now handles correctly.
- `reducer/conversation.rs:608-624` — `reasoning_body_matches` now bails out (returning `false`) early if either side lacks the encoded part, then returns `left_encoded == right_encoded`. The pre-PR `&& readable_reasoning_parts_match(left, right)` co-condition is gone. Since `readable_reasoning_parts_match` itself was permissive (returning `true` if either side was empty), the *operational* effect is small — the divergence case it caught is now treated as "both observations match on identity, merge them." Correct given the design intent.
- `reducer/conversation.rs:631-637` — `merge_reasoning_body` now bails only on `!reasoning_body_matches`, and that bail can only happen on encoded-content disagreement. Bail message correctly updated to reflect the narrower condition.
- `reducer/conversation.rs:638-640` — supersede gate now runs `incoming_readable_reasoning_supersedes_existing` instead of the old "incoming has any readable parts and existing has none" rule. The new rule is **stricter** about when to overwrite — it requires incoming to be a strict superset, not just non-empty when existing is empty. That matches the test's intent ("upgrade only when richer, never when conflicting") but it does mean the case "existing has summary, incoming has text-only" no longer overwrites in *either* direction. Pre-PR the test name was `..._with_different_text_is_reducer_error` (bail). Post-PR it neither bails nor upgrades — it silently keeps existing. Worth a test pinning that branch.
- `reducer/conversation.rs:664-678` — `incoming_readable_reasoning_supersedes_existing` implementation:
  ```rust
  let mut incoming_iter = incoming_parts.iter();
  existing_parts.len() < incoming_parts.len()
      && existing_parts.iter().all(|existing| incoming_iter.any(|incoming| incoming == existing))
  ```
  Note the `incoming_iter` is **mutated** by `.any` (it's an iterator, `any` advances it on first match). The combined effect is checking that existing parts appear in incoming **in order** (subsequence check, not subset check). The doc comment says "ordered superset" — code matches comment. But this means `existing = [text1, text2]` and `incoming = [text2, text1, summary]` would *not* trigger the upgrade despite being a strict-set superset, because `text1` in incoming would be skipped past by the time we look for it after consuming `text2`. Worth a test pinning the order-sensitivity, and worth deciding whether order-preservation is actually the intent or a bug masquerading as a feature. Most likely the Responses API streams reasoning parts in deterministic order, so this is fine — but the comment should explicitly say "subsequence" not just "superset".
- `reducer/conversation_tests.rs:443-510` — the new test pins:
  - same encrypted blob across two requests
  - first sighting: text-only readable
  - second sighting: text + summary
  - assert: single conversation item, parts upgraded to `[text, summary, encoded]`
  - assert: `rollout.conversation_items.len() == 2` (user + reasoning, no duplicates)
  Solid, minimal, and pins exactly the loosening that motivated the PR.

## Risk

1. **Conflict-not-error policy shift**. Pre-PR, two reasoning items with the same encrypted blob but conflicting readable text would *fail loud* with a bail. Post-PR they silently keep existing and ignore incoming. If a real provider bug ever emits the same `encrypted_content` for two semantically-different reasoning items, the rollout-trace will silently absorb only one. Worth a `tracing::debug!` (or `warn!` once-per-rollout) at the supersede-decision site logging "incoming reasoning had readable parts that didn't supersede existing — keeping existing." Cheap diagnostic, no UX cost.
2. **Subsequence vs subset semantics** are subtle and the PR doesn't pin order-sensitivity tests. Add a test where `existing = [A]`, `incoming = [B, A]` (incoming has extra at start) and document the expected outcome (with the current code: `existing.iter().all(|e| incoming_iter.any(|i| i == e))` advances `incoming_iter` past `B`, finds `A`, returns true → upgrades; *but* if `existing = [A, B]` and `incoming = [A, C, B]`, `incoming_iter` finds `A`, then for `B` advances past `C` finds `B` → upgrades; if `existing = [B, A]` and `incoming = [A, B]`, finds `B`? No: starts with `B`, `incoming_iter` consumes `A` then `B` → matches; for `A`, `incoming_iter` is exhausted → false). The actual semantics are "in-order traversal of incoming finds all existing in order" which is order-sensitive subsequence. That's defensible but should be explicit.
3. **Loss of the `ensure_*` symmetry** with `ensure_call_id_consistency`. The two functions were called together at both append sites and read as a paired contract. Now the reasoning case goes through `merge_reasoning_body` lazily and the call-id case still bails eagerly. Future readers will wonder why the asymmetry. A doc comment on the deleted function's old location ("// reasoning consistency is now enforced lazily in merge_reasoning_body to allow benign blob-with-partial-readable replays") would help.

## Verdict

`merge-after-nits` — correctly identifies that the strict equality was over-strict and the right loosening is "encrypted blob is identity, readable parts are best-effort merge." The new test pins the happy path. Two doc nits and two test additions (conflict-no-upgrade case, order-sensitivity case) would close the loop.

## What I learned

When you have two layers of identity (cryptographic blob = strong, human-readable text = weak) for the same logical entity, the reducer/merge contract should be: "match on strong identity, treat weak observations as additive-when-compatible, log-but-don't-bail when weak observations conflict." This PR moves toward that policy correctly. The remaining risk is that the *silent-loss* mode (incoming had richer readable parts but in conflicting order) is observable only through eyeballing rollout dumps, which is exactly the kind of bug that hides for months. A debug-log at the decision point is cheap insurance.
