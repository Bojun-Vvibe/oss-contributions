---
pr_number: 24174
repo: sst/opencode
head_sha: 80aeb78b38a05939af1508edbc0a4acb85b5b9c0
verdict: request-changes
date: 2026-04-25
---

# sst/opencode#24174 — background subagent support (`task(background=true)` + `task_status`)

**What changed.** 12 files / +809 / −61. New tool `packages/opencode/src/tool/task_status.ts` (+ `.txt` description) for polling/awaiting a background subagent. `packages/opencode/src/tool/task.ts` adds `background: Schema.optional(Schema.Boolean)` parameter and a `backgroundOutput(sessionID)` immediate-return path (line ~50). `packages/opencode/src/session/prompt.ts` (line 117) extends `TaskPromptOps` with `loop(input)` and `fork(effect: Effect.Effect<void, never, never>)`. Registry (`tool/registry.ts`) wires `taskstatus` and adds `SessionStatus.Service` to the layer. TUI route (`session/index.tsx` line 2003) appends `(background)` suffix to the task card title.

**Why it matters.** Subagents previously blocked the parent loop until completion. Long-running research/scout tasks (cf. the recently-merged scout-agent PR #24149) want fire-and-forget semantics with a separate poll path.

**Concerns.**
1. **`fork(effect: Effect.Effect<void, never, never>)` is overly narrow as a public op.** The signature requires `never` in both error and requirement channels, which means callers must `.pipe(Effect.catchAll(...))` and provide all dependencies before handing the effect to `fork`. That's correct in Effect-land but pushes a sharp edge to every future caller. Consider widening to `Effect.Effect<void, unknown, never>` and absorbing errors at the prompt-layer boundary with a structured log.
2. **`task_id` schema tightened to `SessionID` from plain `Schema.String`** (task.ts line 28). Good — but this is a wire-breaking change for any model that learned to pass arbitrary strings as `task_id`. Specifically, retrying a stale task_id from a prior session that used the looser schema will now fail Schema decode at the tool boundary instead of falling through to the `Effect.catchCause(() => Effect.succeed(undefined))` on line 110. Either keep the broader input schema and validate inside the handler, or document this as a breaking change in the release notes.
3. **`backgroundMessage` (line ~58) builds the auto-resume notification with `<task_result>` / `<task_error>` tags but `backgroundOutput` (line ~40) uses only `<task_result>`.** Inconsistent — a successful background task that auto-resumes the parent will show `<task_result>` for both the synchronous "started" message and the asynchronous "completed" message. The model has no way to disambiguate "started" vs "completed" except by reading the title line. Add a `state: started|completed|error` line to the synchronous output (currently only the async path emits it, line 64).
4. **`run.fork(cancel(sessionID))` for cancel** (prompt.ts line 116, unchanged) and `run.fork(effect)` for the new background path share the same Fiber pool. No supervision strategy is attached — if a backgrounded subagent throws inside the forked effect after the parent session is closed, the runtime will see an unhandled fiber failure. The `Effect.Effect<void, never, never>` requirement nominally prevents this, but that's only a compile-time guarantee against typed errors, not panics. Add `Effect.tapDefect` at the fork site to log and swallow.
5. **No test for the auto-resume path itself.** `task.test.ts` and `task_status.test.ts` exist, but neither asserts the toast-on-auto-resume behavior described in the summary, which is the user-visible promise of this feature.
6. **`Locale.titlecase(props.input.subagent_type ?? "General")` (TUI line 2005)** unchanged — fine — but the new `(background)` suffix is appended unconditionally as a literal string. Localization layer should own the parenthesized suffix too.

Useful feature, real ergonomics gap on the long-running-subagent path. Don't merge until #1, #3, #4 resolve and at least one E2E test covers auto-resume.
