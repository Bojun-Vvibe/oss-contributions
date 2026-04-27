# sst/opencode#24532 — feat(tui): show background task status in sidebar and fix 0ms duration label

- **Head**: `f0b13ace8eef37c1e11a42c1184808f4adee0a0d`
- **Size**: +85/-1 across 3 files
- **Verdict**: `merge-after-nits`

## Context

Two changes bundled by what they have in common — making background `task`-tool work visible in the TUI:

1. A new sidebar plugin `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/tasks.tsx` (+81) registered into `INTERNAL_TUI_PLUGINS` at `plugin/internal.ts:25` between `SidebarFiles` and `SidebarFooter`.
2. A one-line label fix at `routes/session/index.tsx:2015-2016` swapping the misleading `0ms` duration on completed-but-still-running task tool parts for `"In Progress"`.

## Design analysis

The clever bit is the **dual-state recognition** in `runningTasks` at `tasks.tsx:11-43`. The `task` tool body returns a child `sessionId` almost instantly, so the tool part itself is `running` only briefly, then flips to `completed` while the *child session* keeps working. The plugin handles both:

```ts
if (tool.state.status === "running") { … push }                 // brief window
…
if (tool.state.status === "completed") {
  const childSessionId = (tool.state.metadata as { sessionId?: string })?.sessionId
  if (!childSessionId) continue
  const status = props.api.state.session.status(childSessionId)
  if (!status || status.type !== "busy") continue
  …push                                                         // long window
}
```

That's the right model — a single status check on the parent tool would have a sub-second visible window. The label fix at `routes/session/index.tsx:2015-2016` mirrors the same insight from the rendering side:

```ts
const d = duration()
content.push(`└ ${tools().length} toolcalls · ${d > 0 ? Locale.duration(d) : "In Progress"}`)
```

— previously the line unconditionally called `Locale.duration(0)` which formats as `0ms`, semantically wrong for tasks whose child session hasn't reached `endedAt` yet. PR description's before/after screenshots are consistent with the diff.

## Nits

1. **Duplicated formatting block** — the `type.charAt(0).toUpperCase() + type.slice(1)` + `input.description ?? "Task"` shape is identical in both branches at `tasks.tsx:21-26` and `:36-41`. A small helper (`formatTaskRow(input)`) would dedupe and let both branches share future fields (e.g. start timestamp, child session ID for click-through).
2. **No regression test** — PR body explicitly opts out citing "no existing tests for sidebar plugins". That's accurate as a status check but the dual-state predicate is non-obvious enough to deserve at least a single Vitest exercising both branches against a fake `props.api.state.session.status` mock; the cost is low and the brief-running-window branch is exactly the kind of code that breaks silently when someone refactors `tool.state.status` enum names.
3. **Plugin slot order** — `order: 150` at `tasks.tsx:64` lands between `SidebarFiles`/`SidebarTodo` and `SidebarFooter` numerically, but the human-meaningful ordering across sidebar plugins isn't documented anywhere I can find. Worth a 1-line `// 150 = above footer, below file/todo lists` comment.

## What I learned

The "tool part completes but the work it kicked off is still running" mismatch is a recurring source of UX bugs in agent TUIs (we've seen variants in qwen-code's session picker and crush's todo pills). The pattern of using the tool's emitted child-session-ID as the durable handle, then querying live session status off it, is a clean separation: the tool result is immutable history, the session status is the live signal. The duration-label fix is the smallest possible expression of the same idea — when `endedAt` isn't set, don't pretend the answer is `0ms`.
