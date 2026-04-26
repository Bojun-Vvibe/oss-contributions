---
pr: 24532
repo: sst/opencode
sha: b79d5ee037fa15bf6d0a9935285d1b6bf0b00393
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24532 — feat(tui): show background task status in sidebar and fix 0ms duration label

- **Head SHA**: `b79d5ee037fa15bf6d0a9935285d1b6bf0b00393`
- **Size**: +85 / −1 across 3 files

## Summary
New sidebar plugin `feature-plugins/sidebar/tasks.tsx` (registered at order 150, between Context=100 and MCP/LSP/Todo/Files) plus a `routes/session/index.tsx:2015-2016` label fix that swaps `Locale.duration(0)` ("0ms") for the literal "In Progress" while a child task session is still busy.

## Specific findings
- `feature-plugins/sidebar/tasks.tsx:14-43` — the `runningTasks` memo correctly handles both lifecycle states: `state.status === "running"` (brief, while the tool itself is executing) and `state.status === "completed"` with `metadata.sessionId` still in `busy` status (the common steady-state because the task tool returns quickly while real work continues in the child session). This is the right model.
- `feature-plugins/sidebar/tasks.tsx:36-37` — `props.api.state.session.status(childSessionId)` is called inside the memo. Verify this getter creates a reactive dependency on the child session's status; if it just snapshots, the sidebar won't re-render when the child finishes and the entry will be stuck. Worth a manual check on the API contract.
- `feature-plugins/sidebar/tasks.tsx:24,38` — `input.subagent_type ?? "General"` then `type.charAt(0).toUpperCase() + type.slice(1)`. If `subagent_type` is already capitalized (e.g. `"Explore"`), this produces `"Explore"`; if it's `"explore"`, you get `"Explore"`. Fine. But if it's an empty string (not undefined), you get `""` and the row renders as `"⟳  — desc"`. Cheap to guard with `type || "General"`.
- `feature-plugins/sidebar/tasks.tsx:66` — `<text … wrapMode="none">` will hard-truncate long descriptions. Acceptable for a sidebar but worth a comment or a `…` truncation indicator.
- `routes/session/index.tsx:2014-2016` — guard reads `const d = duration()` then conditions on `d > 0`. Question: can a *completed* task tool legitimately have `duration === 0` because the child session finished in <1ms? If so this label permanently mis-reports "In Progress". Probably negligible in practice but worth a `Math.max(1, d)` floor or an explicit "completed && childBusy" check matching the sidebar's logic.
- `plugin/internal.ts:8,26` — registration is alphabetic-position-aware; placement after `SidebarFiles` and before `SidebarFooter` is consistent with the sidebar order numbering. No concern.
- No test file. PR body acknowledges "no existing tests for TUI sidebar plugins". That's accurate but means the `running`-vs-`completed-but-busy` lifecycle is verified only by manual click-through. A small unit around `runningTasks` (mock `messages` + `state.session.status`) would protect against future plugin-API changes.

## Risk
Low. Pure additive sidebar surface; worst case is a stale entry in the sidebar or a missed render.

## Verdict
**merge-after-nits** — confirm `state.session.status` reactivity, harden the empty-string `subagent_type` case, and consider the sub-1ms duration corner.
