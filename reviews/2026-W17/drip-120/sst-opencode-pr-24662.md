# PR #24662 — fix(tui): preserve Zed context on terminal focus

- **Repo**: sst/opencode
- **PR**: #24662
- **Head SHA**: `64aae7e0`
- **Author**: kitlangton
- **Size**: +31 / -11 across 2 files
- **URL**: https://github.com/sst/opencode/pull/24662
- **Verdict**: **merge-as-is**

## Summary

When the active item in a Zed workspace is a `Terminal` (or any non-`Editor`
kind) the previous query joined `editors` with an `inner join` and required
`buffer_path is not null`, so the row dropped out and `resolveZedSelection`
returned `empty` — clobbering whatever editor context the user previously
had in the prompt. The fix changes the join to `LEFT JOIN`, preserves the
row, classifies it via the new `item_kind` column, and returns
`unavailable` when the active pane item is non-`Editor`. Downstream, the
TUI's editor-context consumer treats `unavailable` as "keep what you had",
which is exactly the desired behavior when the user pops a Zed terminal
into focus and back.

## Specific changes

- `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts:8-9` —
  `ZedEditorRowSchema` gains `item_kind: z.string()` and relaxes
  `editor_id` to `z.number().nullable()`. The new
  `ZedActiveEditorRow` intersection type narrows back to
  `item_kind: "Editor"; editor_id: number` so the contents-query
  signature is statically guaranteed to have a non-null editor id.
- `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts:66-83` —
  SQL switches to `left join editors e ...` and reads `i.workspace_id`
  instead of `e.workspace_id` (correctly: with a left join the editor
  side may be null, but `items.workspace_id` is always present). The
  `where` clause drops the `i.kind = 'Editor' and e.buffer_path is not
  null` filter; classification now happens in TypeScript.
- `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts:96-100` —
  after the score/sort, an `item_kind !== "Editor"` check returns
  `unavailable`, then `isZedActiveEditorRow` (the new type guard at
  `:128-130`) gates the success path. The two states are correctly
  distinct: `unavailable` means "Zed is busy with a non-editor item,
  keep the previous selection"; `empty` means "no rows matched at all".
- `packages/opencode/src/cli/cmd/tui/context/editor-zed.ts:108` —
  `queryZedEditorContents` parameter type tightens from `ZedEditorRow`
  to `ZedActiveEditorRow`, so the function body's reliance on
  `row.editor_id` is now type-checked rather than relying on the
  caller's runtime guard.
- `packages/opencode/test/cli/tui/editor-context.test.ts:75-78` —
  new test `"resolveZedSelection returns unavailable when a Zed
  terminal is active"` constructs a fixture with `itemKind: "Terminal",
  editor: false`, asserts the exact `{ type: "unavailable" }` return.
  Pinning the `unavailable` (not `empty`) discriminant is the right
  contract to lock down because the consumer's behavior diverges
  on it.

## Risks

The SQL change is the only meaningful behavioral shift. With the
inner-join + `i.kind = 'Editor'` filter gone, the query now returns
rows for every active pane item across all panes (including images,
terminal panes, settings panes, etc.), and the TS layer filters down
to the active editor. For a typical Zed workspace this is at most
a handful of rows; the existing `score > 0` workspace-path match
plus `timestamp desc` sort already narrows to one row. No index
concern.

The fixture rewrite at `editor-context.test.ts:18-35` correctly
gates the `editors` and `editor_selections` inserts on
`options.editor !== false` so the new "Terminal" case can model a
real workspace where the editor side is genuinely absent. Existing
tests pass because the default `itemKind` and `editor` are unchanged.

## Verdict rationale

Tight, focused, schema-clean fix to a real "context disappears when
I touch the terminal" UX bug. Type guard makes the
non-null editor-id assumption explicit at the function boundary
where it matters. Test pins the discriminant the consumer branches
on. Merge as-is.
