# sst/opencode #24290 — fix(session): skip tool calls during summary instead of throwing

- **PR:** https://github.com/sst/opencode/pull/24290
- **Head SHA:** `25cdb8f9e06f36057521a138e3538126e57f6771`
- **Base:** `dev`
- **Files changed:** ~12 (terminal panel UX + PTY `source` field + processor summary-guard relaxation + tool registry wiring)

## Summary of the diff

Two unrelated themes are bundled:

1. **Summary-guard softening** in `packages/opencode/src/session/processor.ts:258-291` — `tool-input-start` and `tool-call` no longer `throw` when `ctx.assistantMessage.summary` is true; they `slog.warn` and `return` instead.
2. **Agent-vs-user PTY tagging + auto-open panel UX** — adds a `source: "user" | "agent"` field to the PTY schema (`packages/opencode/src/pty/index.ts:62-77,215-217`), forwards it through `Pty.create`, listens for `pty.created` in the app context (`packages/app/src/context/terminal.tsx:185-199`), and reworks `terminal-panel.tsx:65-105` so the panel auto-opens when an agent-created PTY appears and auto-closes when the last terminal disappears.

## Line-level call-outs

- `packages/opencode/src/session/processor.ts:260-261` and `:289-290` — the swap from `throw` to `slog.warn(...) ; return` is the right call (an in-flight summarization should not nuke the whole turn just because the model emitted a stray tool call), but this silently drops the tool call without surfacing it to the user. Consider adding a single user-visible breadcrumb part (e.g. a synthetic `info` part) so people don't wonder why the model "ignored" the tool. At minimum, include the `value.id` / `toolCallId` in the warn payload so the dropped call is traceable in logs — right now only `toolName` is logged, which makes correlation in multi-tool turns annoying.
- `packages/opencode/src/pty/index.ts:62-66` — `source` is `Schema.optional(...)` on `Info`, but `:217` always materialises `source: input.source ?? "user"`. Once layer-created PTYs always carry a value, the optionality on the wire schema is purely for backward-compat of existing serialized state; worth a one-line comment so a future cleanup doesn't tighten the schema and break replay.
- `packages/app/src/context/terminal.tsx:188-198` — the new `pty.created` listener does `findIndex` then `setStore("all", store.all.length, ...)`. If two `pty.created` events race (e.g. SSE replay after reconnect), the dedup window is fine, but `numberFromTitle(info.title) ?? 0` will collide on `0` for every untitled agent PTY. Worth deriving from `store.all.length + 1` to keep titles unique in the sidebar.
- `packages/app/src/pages/session/terminal-panel.tsx:69-80` — the new "auto-open on agent PTY" effect fires on `terminal.all().length` change, then reads `all[all.length - 1]` to identify the new entry. This is fragile under concurrent PTY creates (the latest entry may not be the just-arrived one if the array got two adds in the same tick). Tracking the previous id-set and diffing would be more correct, even if the practical race is rare.
- `packages/app/src/pages/session/terminal-panel.tsx:82-91` — moving the `focus(id)` effect above the close-handling effect changes ordering relative to before. Verify there's no regression where focus runs before the panel finishes its open transition (Solid's effect order is creation order).
- `packages/opencode/src/tool/registry.ts:35,90,191-200` — adding `Pty.Service` as a layer requirement and importing `TerminalTool` widens the registry's dependency graph. Make sure every alternative `layer` (tests, headless runs) provides a `Pty.Service`, otherwise tool registry construction fails at startup with a confusing "Service not found" rather than a feature-flag.
- `bun.lock` + `packages/opencode/package.json:45` — pinning `@babel/types` to `7.29.0` as a `devDependency` while `@babel/core` stays at `7.28.4` is fine for now but they share an internal contract; if a later `@babel/core` bump lands, double-check `@babel/types` moves with it.

## Verdict

**merge-after-nits**

## Rationale

The core change (no longer throwing during summary) is a real reliability win — current behaviour aborts the entire turn on a benign race. The terminal-panel UX polish is well-scoped. Two nits worth addressing before merge: (a) log `toolCallId` alongside `toolName` in the dropped-call warn so it's grep-able, and (b) the "newest PTY = last array entry" assumption in `terminal-panel.tsx` should diff id-sets instead. The bundled scope (summary-fix + UX rework + new tool registration) is large for one PR; reviewers may ask for a split, but the changes are at least cleanly separated by file.
