# anomalyco/opencode PR #24271 — Set active server before navigation and use replace navigation to avoid extra history entries

- **URL:** https://github.com/anomalyco/opencode/pull/24271
- **Head SHA:** `831bf6b2ac88a82032a9f8d359e8f59228ebefb6`
- **Files touched:** 2 (`packages/app/src/components/dialog-select-server.tsx`, `packages/app/src/components/status-popover-body.tsx`)
- **Verdict:** `merge-as-is`

## Summary

Fixes a server-switching race in the desktop app's SolidJS UI.
Previously both server-selection flows called `navigate("/")` first
and then deferred `server.setActive(...)` via `queueMicrotask`. The
next route could mount before the active-server signal had updated,
producing transient `Session not found` errors on the new server's
first render. Fix reverses the order and uses `replace: true` so a
server switch isn't a new history entry.

## Specific references

- `packages/app/src/components/dialog-select-server.tsx:354-361` —
  the `if (persist && conn.type === "http")` branch becomes
  `navigate("/", { replace: true })`; the fall-through case swaps
  `navigate("/"); queueMicrotask(() => server.setActive(...))` for
  `server.setActive(...); navigate("/", { replace: true })`.
- `packages/app/src/components/status-popover-body.tsx:292-296` — same
  reorder applied to the popover's per-server click handler. Same
  shape, same fix.

## Reasoning

Correct diagnosis. With SolidJS signals, `queueMicrotask` deferring
the `setActive` call after `navigate` means the route renders against
the previous server's state, and the fix of "set state, then
navigate" is the canonical pattern. Using `replace: true` is also the
right call — server switches are not navigation events users want in
their back-stack.

Diff is ~5 lines per file, both call sites are updated symmetrically,
no new imports needed, and the logic in the `persist && http` branch
is preserved (it still calls `server.add(conn)` first). Low-risk and
self-contained. Merge as-is.
