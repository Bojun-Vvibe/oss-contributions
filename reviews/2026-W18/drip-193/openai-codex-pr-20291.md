# openai/codex #20291 — app-server: remove dead api version handling from bespoke events

- **URL:** https://github.com/openai/codex/pull/20291
- **Head SHA:** `224ead8cfc77719ccb95cf3fa03bb8974883980a`
- **Merge SHA:** `4e677d62da09c6b3243972b9191ee93c6c5e6527`
- **Files:** `codex-rs/app-server/src/bespoke_event_handling.rs` (large net-removal: -~750 lines across the `apply_bespoke_event_handling` function, sub-handlers `complete_command_execution_item`, `maybe_emit_raw_response_item_completed`, `handle_turn_diff`, `handle_thread_rollback_failed`, `handle_error`; multiple `mod tests` blocks shrink), `codex-rs/app-server/src/codex_message_processor.rs` (-~14 lines; ~14 single-line removals scattered through `CodexMessageProcessor` impl methods at `:2832, 4204, 4365, 4544, 4941, 6904, 7190, 7302-7325, 7347, 7363, 7376, 7420, 7461, 7477, 7486`)
- **Verdict:** `merge-after-nits`

## What changed

Pure dead-code purge: removes the legacy "API version" branching that lived inside `apply_bespoke_event_handling` and its callees in `bespoke_event_handling.rs`, plus the corresponding pass-through plumbing scattered through `CodexMessageProcessor` in `codex_message_processor.rs`. The diff is heavily lopsided toward removal — the line count map (`grep -E "^@@" /tmp/d_20291.diff | head`) shows nine large hunks in `bespoke_event_handling.rs` (`@@ -239,435 +228,377`, `@@ -687,406 +618,314`, `@@ -2415,105 +2190,6` = pure deletion of `handle_error`, `@@ -4928,35 +4595,6` = deleted test) and roughly two-dozen one-line removals in `codex_message_processor.rs` that prune the now-unused parameter pass-throughs.

The shape is "this branch is now unreachable, the dispatch table no longer routes to it, so delete the branch and its plumbing top-down." The diff's `+0/−~750` ratio is the strongest signal that this is the right kind of refactor: nothing new is being introduced, established behavior at the *current* API version is preserved, and the support burden for the legacy version goes to zero.

## Why it's right

- **Removing dead branches eliminates a class of behavioral drift.** When two API versions co-exist behind `match version { ... }` arms, every behavioral fix has to be applied to both arms, and bugs creep in when only one arm gets the fix. Once the legacy arm is provably unreachable (no client speaks it anymore), keeping it around is a maintenance tax with no upside — fixes for the live arm silently regress the dead arm, the dead arm bit-rots, and a future "let's resurrect v1 for compatibility" attempt finds 18 months of accumulated drift.
- **The deletion of `handle_error` at `:2415-2519` (`-105/+6`) is a strong signal of consolidation.** A 100+-line dedicated error handler that survives only as a 6-line stub is the canonical "this whole code path was version-specific scaffolding" pattern. The remaining 6 lines are presumably the version-agnostic delegation; the removed bulk is the per-version error-shape translation that the live API contract no longer needs.
- **`codex_message_processor.rs` removals are all single-line** — the pass-through parameter (likely `api_version`) is being threaded out of every callsite uniformly. That's the right shape: one identifier removed everywhere, no semantic change at any individual call. The 14+ touch sites at `:2832, 4204, 4365, 4544, 4941, 6904, 7190, 7302-7325, 7347, 7363, 7376, 7420, 7461, 7477, 7486` are mechanically the same edit.
- **Test files at `bespoke_event_handling.rs:3287-2963, 4425-4100, 4908-4576, 4928-4595, 4976-4614` shrink uniformly with the production code.** Tests for unreachable code paths get deleted alongside them — the alternative (keeping the tests but skipping them) is worse because it implies the dead code is "still tested" while the test never actually runs.

## Nits (non-blocking)

1. **No CHANGELOG / migration note in the diff slice.** A `-750-line` removal that retires a legacy API version *should* have a corresponding entry naming the version that's being dropped and the cutover date. If clients that haven't upgraded yet still exist in the wild, they'll get a runtime "unsupported version" error with no documentation pointing them at the upgrade path. Worth adding a one-paragraph note in the README/CHANGELOG even if all known clients have already migrated — it makes the support story crisp for anyone hitting the error months from now.

2. **The diff doesn't show whether the wire protocol's version-negotiation handshake also removes the legacy version from the advertised set.** If the server still *advertises* support for the legacy version but no longer *implements* it, that's a worse failure mode than a clean rejection at handshake. Verify that `app-server-protocol`'s version-list constant has the corresponding removal in the same release (not in this PR's diff slice; should be a peer/follow-up).

3. **Removed test `at :4928-4595` (`-29 lines`) is gone with no replacement.** That was presumably a "behavior should be X under legacy version" test — fine to delete since the legacy branch is gone, but worth confirming there's a sibling "behavior should be X under current version" test that already covered the same observable behavior. If the deleted test was the *only* coverage of a specific observable, the deletion creates a coverage gap that won't show up in line-coverage metrics (since the production code is also gone) but will show up the next time someone refactors the surviving handler.

4. **The diff slice doesn't include the `mod.rs` / `lib.rs` exports.** A `pub use` of any of the removed types (e.g. a legacy-version-specific event variant) would silently leak into the crate's public surface and cause downstream `unused_imports` warnings or worse. Worth running `cargo doc --no-deps` on the resulting tree to confirm nothing the crate exports references the removed paths.

5. **`apply_bespoke_event_handling` is still ~1700 lines after the cut.** The legacy-version removal is the right first step, but the function's surface area suggests another round of "split the dispatch by event type into per-event modules" would now be substantially easier — most of the version-orthogonal scaffolding that was holding it together as one big match is now gone. Worth a follow-up issue to track.
