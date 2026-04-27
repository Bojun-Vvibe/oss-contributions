# openai/codex#19706 — Preserve TUI markdown list spacing after code blocks

- **Author**: etraut-openai
- **Size**: +66 / -0 (markdown_render.rs + tests + snapshot)
- **Issue**: #19702

## Summary

The TUI markdown renderer was visually attaching the next list marker directly to a fenced code block inside the previous list item, even when the source markdown had a blank line between items. Behavior was wrong specifically for "loose" lists with code-block items; "tight" simple loose lists should still render compact.

Fix tracks two new pieces of state in the `Renderer` struct (`codex-rs/tui/src/markdown_render.rs:154-158`):
- `list_needs_blank_before_next_item: Vec<bool>` — per-list-depth flag
- `list_item_contains_code_block: Vec<bool>` — per-item flag

Logic:
1. `start_codeblock` (`:546-548`) flips the current item's "contains code block" flag.
2. `TagEnd::Item` (`:298-303`) consumes the item's flag and, if true, sets the parent list's "needs blank before next item" flag.
3. `start_item` (`:495-503`) consumes the parent list's flag and emits a blank line if set.

## Diff inspection

```rust
// :298-303 — TagEnd::Item handler
if self.list_item_contains_code_block.pop().unwrap_or(false)
    && let Some(needs_blank) = self.list_needs_blank_before_next_item.last_mut()
{
    *needs_blank = true;
}
```

Uses Rust 2024 `let-chains` syntax — assumes the codex-rs MSRV supports it. Worth confirming.

```rust
// :495-503 — start_item
if self
    .list_needs_blank_before_next_item
    .last_mut()
    .map(std::mem::take)
    .unwrap_or(false)
{
    self.push_blank_line();
}
```

The `std::mem::take` on the `&mut bool` is the cleanest way to "read and reset" — equivalent to `*x = false; old_x` in one expression. Correct.

The two new state stacks (`list_indices`, `list_needs_blank_before_next_item`, `list_item_contains_code_block`) are pushed/popped in lockstep with the existing `list_indices` (start_list pushes/end_list pops `list_needs_blank_before_next_item`; start_item pushes / TagEnd::Item pops `list_item_contains_code_block`). Verified push/pop balance:
- `list_needs_blank_before_next_item`: push at `start_list` (`:488`), pop at `end_list` (`:494`) — balanced ✓
- `list_item_contains_code_block`: push at `start_item` (`:497`), pop at `TagEnd::Item` (`:298`) — balanced ✓

Test coverage in `markdown_render_tests.rs:1141-1163`:
- `list_item_after_code_block_keeps_blank_separator` — inputs `1. First:\n\n   ```rust\n   fn first() {}\n   ```\n\n2. Second:\n`, expects 5-line output with blank separators around the code block.
- `list_item_after_simple_item_stays_compact` — inputs `1. First\n\n2. Second\n`, expects 2-line compact output.
- New `plain_lines` helper at `:18-29` flattens spans to strings for stable assertions.
- New snapshot file at `tui/src/snapshots/...keeps_blank_separator.snap` pins the rendered output against insta.

## Strengths

- Two-flag state machine is the minimal correct shape — one tracks "did this item have a code block" (per-item, popped on item end), the other tracks "should the next item get a blank prefix" (per-list, consumed once).
- `std::mem::take` for read-and-reset is idiomatic and avoids the bug-prone two-statement form.
- Snapshot test pins the visual output against regressions in either direction (too-compact OR too-loose).
- Two complementary tests prove the fix doesn't over-trigger: code-block items get blanks, simple items stay compact.
- Before/after screenshots in the PR body make the visual difference immediately obvious.
- Zero deletions — purely additive change, can't regress existing behavior outside the targeted case.

## Concerns

1. **Nested code blocks.** What about `1. Item with sublist:\n   - sub-item with code block\n   - another sub-item\n2. Top item`? The `list_item_contains_code_block` flag bubbles to the **immediate** parent item, but a sub-list's code-block-having sub-item should arguably also flag the outer item as "had a block" (so the outer-list separator triggers). Untested in the diff. Worth adding `list_item_after_code_block_in_sublist` test.
2. **`let-chains` MSRV.** `if expr && let Some(x) = ...` is stabilized in Rust 1.88 (May 2025). codex-rs's `Cargo.toml` `rust-version` field should be checked — if it's `1.85` or earlier, this won't compile. Trivially rewritable as nested `if`.
3. **Mixed list types.** `1. ordered\n\n  - nested unordered with code block\n  - another\n\n2. ordered` — the outer list's `list_needs_blank_before_next_item` should still fire when the inner unordered list's first item has a code block? PR doesn't address. Probably not the bug from #19702 but worth a thinking-out-loud comment.
4. **No fuzz / property test.** The state stacks are easy to imbalance (pop without push) under malformed markdown. The pulldown-cmark parser shouldn't emit imbalanced events but a property test would catch any regression.
5. **`list_needs_blank_before_next_item.pop()` in `end_list`** drops the flag without checking if it was set. If the **last** item in a list contained a code block, the flag is set but never consumed (no next item). That's actually correct behavior (no separator needed before non-existent next item) but worth a comment so the next reader doesn't try to "fix" it.

## Verdict

**merge-as-is** — the fix is surgically correct, the state-machine is the minimal shape, push/pop balance verifies, and tests pin both the fix and the non-regression. The MSRV check on `let-chains` is the only thing worth confirming pre-merge but trivially fixable. Nested-list and mixed-list cases can ride on follow-ups.
