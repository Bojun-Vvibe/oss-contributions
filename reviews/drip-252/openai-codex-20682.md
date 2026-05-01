# openai/codex #20682 — feat(app-server): always return limited thread history

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20682
- **HEAD SHA:** `8bce448c942e292b7e3e0e8b889cf7b7844a8fbd`
- **Author:** owenlin0
- **Verdict:** `merge-after-nits`

## What the diff does

Decouples *what gets persisted* from *what gets returned* in the
`thread/read` and `thread/resume` response surface. Previously the
`persistExtendedHistory: true` flag did double duty — it both wrote
extra `EventMsg` variants to the rollout file AND those richer
items were echoed back in `thread.turns` on read/resume/fork. The
"echoed back" half made response payloads grow unboundedly with
every WebSearch / ImageGeneration / DynamicToolCall in the session.

Three coordinated changes in `codex-rs/`:

1. Doc / contract clarifications at
   `app-server-protocol/src/protocol/v2.rs:3611-3870` (`ThreadStartParams`,
   `ThreadResumeParams`, `ThreadForkParams`) and `app-server/README.md:307,403`
   make the new contract explicit: persist-extended is now a
   storage-only flag; read/resume responses always project to the
   Limited surface.
2. New helpers at `codex_message_processor.rs:8593-8642` —
   `populate_thread_turns_from_limited_history`,
   `build_limited_turns_from_rollout_items` (delegates to
   `ThreadHistoryBuilder` filtered by
   `is_persisted_rollout_item(item, EventPersistenceMode::Limited)`),
   `limited_active_turn`, and `thread_item_maps_to_limited_rollout`
   (a tagged-arm whitelist: User/Hook/Agent/Plan/Reasoning/
   Compaction/Review-mode-enter+exit are in; FileChange and
   ImageGeneration are in only when not in-progress; Command/Mcp/
   Dynamic/CollabAgent/WebSearch/ImageView are out).
3. Three call-site swaps at `:4046, :4107, :4933` and `:8418-8425`
   route `thread/read`, `thread/resume`, `thread/fork`, and the
   pending-resume continuation through the new limited helpers; the
   pending-resume path also drops its `Result`/error-handling shape
   since the limited path is infallible.

Locking test: `limited_turn_projection_filters_extended_rollout_items`
at `:10208+` constructs a 5-event rollout (TurnStarted, UserMessage,
WebSearchEnd, ImageGenerationEnd "completed", TurnComplete) and
asserts the projection keeps the user message + image generation but
drops the web search.

## Why the change is right

The previous double-duty contract was a foot-gun shaped exactly like
the one #20677 (drip-251) closed for the MCP-tool-call lifecycle:
"store one thing, return a different thing" was conflated into a
single boolean. Splitting them puts persistence and projection at
two independent typed gates so future surface changes (add a new
EventMsg, expose more in storage, expose less in response) only
touch one of the two.

The whitelist at `thread_item_maps_to_limited_rollout` is the load-
bearing detail: it's a tagged-arm exhaustive match over `ThreadItem`
so adding a new variant forces a compile error here, preventing the
"silently leaks into the response payload" failure mode. The
"in-progress" carve-outs for `FileChange` and `ImageGeneration`
(`!matches!(status, PatchApplyStatus::InProgress)`,
`!status.is_empty()`) are the right shape — partial state shouldn't
leak across resume because clients can't render half-state safely.

The `pending-resume` simplification at `:8418-8425` (drop `Result`
since limited projection is infallible) is honest collateral
cleanup, not feature creep.

## Nits (non-blocking)

1. **Inconsistent helper retention.** The old
   `populate_thread_turns_from_history` at `:8593-8592` is preserved
   but no longer called. If it has no other consumer it should be
   deleted in this PR (dead code) or marked with a comment explaining
   why it stays (e.g., "kept for an experimental
   `experimentalReturnFullHistory` flag we plan to add"). Leaving
   an obviously-similar helper next to its limited sibling invites
   future drift where someone fixes a bug in one and not the other.

2. **`build_turns_from_rollout_items` callers still exist.** The diff
   swaps three call sites to `build_limited_turns_from_rollout_items`
   but the unprefixed `build_turns_from_rollout_items` is still
   present (the merge_turn_history_with_active_turn / Reasoning
   / Plan paths reference it). Worth confirming via `grep -n
   "build_turns_from_rollout_items" codex-rs/app-server/src/`
   that every remaining caller is intentionally returning the
   richer projection (e.g., for an internal consumer that *does*
   need the full set), and that none is a `thread/read` / `resume`
   / `fork` path that was missed.

3. **Test coverage of `limited_active_turn`** is missing. The new
   `limited_turn_projection_filters_extended_rollout_items` test
   at `:10208+` exercises the rollout-item path but not the
   active-turn path; a parallel test that constructs a `Turn`
   with a `ThreadItem::WebSearch` and asserts it's filtered out
   would lock the active-turn arm. The two paths use different
   filters (`is_persisted_rollout_item(EventPersistenceMode::Limited)`
   vs `thread_item_maps_to_limited_rollout`) and should each be
   pinned.

4. **README / doc churn** at `:307, 403` is correct but the
   `persistExtendedHistory` flag's prior documented behavior
   ("persist a richer subset of ThreadItems for non-lossy history
   when calling `thread/read`...") becomes a behavior-change for
   any client that *was* relying on the richer return payload.
   Worth a CHANGELOG / release-notes entry calling this out so
   downstream clients know to switch to a different mechanism
   (persist-extended for storage + ad-hoc rollout-file reads for
   audit) rather than discovering it as a regression.

5. **`thread_item_maps_to_limited_rollout` semantics for
   `Reasoning`** — keeping `Reasoning { .. } => true` is the
   correct call (reasoning is intentional persisted state, the
   user wants it back on resume), but a one-line code comment on
   the `true` arms would future-proof against someone narrowing
   the whitelist later thinking they're being conservative.

## Verdict rationale

Right architectural decoupling, right shape (tagged-arm whitelist
that fails-closed at compile time), load-bearing locking test in
place. Dead-code helper retention, missing active-turn projection
test, and the behavior-change CHANGELOG note are the real review
items; the design is solid.

`merge-after-nits`
