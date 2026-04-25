# anomalyco/opencode#24383 — fix: move session roots filter from client-side to SQL layer

- **Repo:** anomalyco/opencode
- **Author:** heimoshuiyu (黑墨水鱼)
- **Head SHA:** `16fa43f1` (16fa43f1cea3066803689b91b6487c114e322a98)
- **Size:** +3 / -4

## Summary
The TUI was paginating session lists with `limit: 30` and then filtering out
child sessions client-side via `.filter((x) => x.parentID === undefined)`.
With many sub-sessions, the user-visible list could be far smaller than 30,
or empty. PR pushes the filter to the SQL layer via the existing
`roots: true` parameter on `sdk.client.session.list`.

## Specific references
- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx:37` @ `16fa43f1` — search call now `{ search: query, limit: 30, roots: true }`.
- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx:115` @ `16fa43f1` — removes the post-fetch `.filter((x) => x.parentID === undefined)`. Correct: the filter is now redundant.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:365` @ `16fa43f1` — initial sync changed to `{ start: start, roots: true }`.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:485` @ `16fa43f1` — refresh path also now `{ start, roots: true }`. Both paths consistent.

## Observations
1. **Server-side `roots` semantics need confirmation**: the PR assumes `roots: true` filters to `parentID IS NULL` at the SQL layer. If the server impl actually filters to "sessions with no children" or similar, the behavior changes subtly. A pointer to the server route in the PR description (or a quick test) would lock this in.
2. **Search-result drop**: by removing the client-side filter from search results too, callers who used to receive sub-sessions in search hits no longer do. Search by keyword used to potentially match a child session's title — now those won't surface in the dialog. Whether this is desirable depends on UX intent. Probably right (the dialog only knows how to render roots), but worth confirming.
3. **No test added**: behavioral change in two distinct call sites with no test. A snapshot test of `DialogSessionList` rendering against a fixture with parent + child sessions would prevent silent regression if the server's `roots` flag changes meaning.

## Verdict
**merge-after-nits** — confirm `roots` server semantics in the description and add a fixture test; otherwise a clean, correct fix.
