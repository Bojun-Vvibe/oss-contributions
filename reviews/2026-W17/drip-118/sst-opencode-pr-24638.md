# PR #24638 — fix(tui): propagate permissions from nested subagents and show full subtask tree

- **Repo**: sst/opencode
- **PR**: #24638
- **Head SHA**: `9cd9587b`
- **Author**: michalsalat (Michal Salát)
- **Size**: +114 / -7 across 2 files
- **Verdict**: **merge-after-nits**

## Summary

Fixes two real, user-visible TUI bugs (closes #13715 and #7654) by
replacing the direct-children-only collection in the `permissions`
and `questions` memos with a BFS traversal over all descendant
sessions, and by recursively rendering nested subagent task trees
inline in the `Task` tool view with per-session stats. The bug
shape: when a subagent spawns its own subagent that needs (e.g.)
`bash` permission, the prompt was being collected via
`children().flatMap((x) => sync.data.permission[x.id] ?? [])` —
which only looks one level deep, so deep-nested permission
requests were dropped on the floor and the session hung forever.
The fix mirrors what the web/desktop app already does via
`sessionTreeRequest()`.

## Specific changes

- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:136-156` — new `descendants` memo that builds a `childrenByParent: Map<string, string[]>` index from `sync.data.session` and does an iterative DFS (using a `stack: string[]` and `stack.pop()`) starting from `session()?.parentID ?? session()?.id`, collecting all reachable session IDs into a flat array. Iterative not recursive — good choice because TUI sessions can nest arbitrarily deep and a recursive walk would risk stack overflow on adversarial input.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:160-165` — both the `permissions` and `questions` memos now `descendants().flatMap((id) => sync.data.permission[id] ?? [])` instead of `children().flatMap(...)`. The existing `children()` memo is preserved — it's still needed for the sibling-navigation surface, so the right call was to add a second memo rather than mutate the first.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1989-2046` — three new helpers: `sessionStats(sessionID, sync)` sums `tokens.input + tokens.output + tokens.reasoning` and `cost` over assistant messages; `formatStats({tokens, cost, toolCount, duration})` builds the `45s · 12 tools · 18,432 tk · $0.0891` style line; `taskSubtree(sessionID, sync, depth)` walks the message-part tree, filters to `tool === "task"` parts, and recursively renders status icon + description + stats + indented children. The lazy fetch (`if (!sync.data.message[childSessionID]?.length) void sync.session.sync(childSessionID)`) is the right call — don't block render on data that may not be cached yet, but trigger the fetch so the next render has it.
- `packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx:408-411` — permission prompt now extracts `Object.entries(data)` (with `undefined`/`null`/`""` filter) and renders each `key: value` as a separate text row instead of just `Tool: ${permission}`. This addresses the "from parameter" suggestion in #7654 — users can now see what arguments the subagent is requesting, not just which tool.
- `packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx:429-444` — new `subagentLabel()` derived computed strips a trailing `(@xxx subagent)` suffix from `session().title` and renders it next to the "Permission required" header so the user sees which subagent in the tree is requesting permission, not just "some descendant".

## Risks

- **`descendants()` rebuilds the full child index on every memo invalidation**: the current shape (`for (const s of all) { ... }`) is O(n) over every session in the workspace, and SolidJS will re-run this whenever `sync.data.session` changes — i.e. every time a subagent emits a status update. For a deeply nested session this is fine, but for a workspace with hundreds of historical sessions the per-update O(n) becomes a hot path. A workspace-level memoized index keyed off `sync.data.session.length` would amortize this.
- **`taskSubtree` returns flat lines via `[]string` then `.join("\n")`**: the description says "indented status indicators" and the screenshot shows the right thing, but the implementation glues everything into one big text block rather than rendering each subtask as its own JSX node. That means screen readers, copy-paste, and any future "click subtask to navigate" feature will have to re-parse a flat string. Not a blocker, but worth a comment that this is a temporary rendering shape.
- **`sessionStats` only sums assistant messages, ignoring user/tool tokens**: the `Message` shape has `tokens.input + output + reasoning` for assistant messages, which is the right billable surface. But the per-tool stats (`toolCount`) come from filtering all parts across all messages, which double-counts a tool result that appears in both the assistant message and the tool message. Worth verifying with a session that has many tool calls.
- **`session().title.replace(/\s*\(@\w+\s+subagent\)\s*$/, "")`** in the `subagentLabel` helper hard-codes the format opencode emits today. If the title format ever changes (e.g. localized "subagent" → "子代理") the regex silently returns the unstripped title. A constant naming the format and a unit test pinning the strip would help.
- **No test coverage for `descendants()` traversal**: the PR runs `bun test test/permission/` clean (85/85) but adds no test for the new BFS itself. The bug shape (one-level-deep collection) was a logic bug in the memo; a test that constructs a 3-level session tree and asserts `descendants().length === 7` (or whatever) would pin it directly.

## Verdict

`merge-after-nits` — the bug is real and the diagnosis is correct
(direct-children-only collection in a tree-shaped data structure
is a textbook one-level-traversal bug). The fix mirrors a pattern
already established in the web/desktop app, which is the right
"least surprise" answer. Asks for the maintainer: (1) add a unit
test for `descendants()` against a multi-level session tree;
(2) consider workspace-level memoization for the
`childrenByParent` index if profiling shows the per-update
rebuild is hot; (3) pull the `(@xxx subagent)` strip regex into
a named constant with a test fixture so localization doesn't
silently break the label.

## What I learned

This is a near-perfect example of how cleanly a small data-shape
bug presents in a tree-shaped UI: every level deeper than the
collector reaches becomes an invisible class of stuck-forever
session. The TUI surface had `children()` because that's all the
sibling-navigation column needed, and the permission collector
just inherited that shape without anyone noticing the mismatch
between "one-level data dependency for sibling navigation" and
"all-levels data dependency for permission gating". The right
defense against this class is a typed wrapper around the data
access — e.g. `sessionTree(rootID)` that returns a typed
`SessionNode[]` and forces every consumer to opt in to the
traversal depth they want — rather than two memos that look
similar but mean different things.
