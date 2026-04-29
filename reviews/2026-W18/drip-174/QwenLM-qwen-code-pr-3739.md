# QwenLM/qwen-code #3739 — Add background agent resume and continuation

- **PR:** https://github.com/QwenLM/qwen-code/pull/3739
- **Title:** `Add background agent resume and continuation`
- **Author:** doudouOUC (jinye)
- **Head SHA:** d0f49f56e4923ab57af79466dfec6b9a212c1fc6
- **Files changed:** 24 (notably new `background-agent-resume.ts` +974, new test file +643, `agent-transcript.ts` +131/−2, `tools/agent/agent.ts` +53/−1, `tools/send-message.ts` +27/−4, `tools/task-stop.ts` +22/−2, plus dialog/pill/AppContainer/config wiring), **+2271 / −33**
- **Verdict:** `merge-after-nits`

## What it does

Adds first-class persisted resume/continuation for background
subagents in interactive CLI sessions. Three substantive pieces:

### 1. `BackgroundAgentResumeService` (`background-agent-resume.ts` +974)

Discovers paused background agents from the
`subagents/<sessionId>/` sidecar directory and surfaces them as a new
`paused` status entry in the existing background-task registry. On
`AppContainer` mount (`AppContainer.tsx:362-377`), calls
`config.loadPausedBackgroundAgents(config.getSessionId())` and emits a
transient `MessageType.INFO` notice through `historyManager.addItem`
when any are recovered. The service exposes `resumeBackgroundAgent` /
`abandonBackgroundAgent` actions wired through to the dialog
(`BackgroundTasksDialog.tsx:492-498, 521-527`) on `r` and `x` keys.

### 2. Transcript-first fork resume (`agent-transcript.ts:+131/−2`)

This is the load-bearing design choice. Forked agents now write two
new system records into the transcript on bootstrap:
- `system/agent_bootstrap` — the fork bootstrap history
- `system/agent_launch_prompt` — the original launch task prompt

On resume, the fork worker context is reconstructed *from the
transcript* rather than rebuilt from current parent prompt/tool
state. The PR body explicitly notes "legacy fork transcripts without
bootstrap records remain visible as paused and abandonable, but are
blocked from unsafe resume" — that's the right migration shape (don't
silently apply a new schema to old data).

### 3. Lifecycle plumbing across the existing tool surface

`send-message.ts` (+27/−4) and `task-stop.ts` (+22/−2) gain
paused-status branches; `subagent-manager.ts` (+22/−5) extends the
lifecycle state machine with `paused`; `agent.ts` (+53/−1) re-runs
`SubagentStart` hooks on resume and coalesces concurrent resume
attempts (per PR body); `runtime/agent-headless.ts` (+11/−4) handles
the resumed-fork transcript-first init.

### Downstream-consumer audit (called out by author)

PR body claims a focused review of three consumers that read the
transcript schema:
- **session export path** (`exportCommand.test.ts`) — only replays
  `slash_command` system records, ignores other system subtypes
  safely.
- **ACP history replay** (`HistoryReplayer.test.ts`) — skips unknown
  system subtypes, so `agent_bootstrap`/`agent_launch_prompt` don't
  leak into replayed chat.
- **insight/session analytics** (`DataProcessor.test.ts`) — only
  counts `user`/`assistant`/`tool_result`/`slash_command`, so new
  records don't skew metrics.

That audit is the right shape for a transcript-schema change. It's
the difference between "extending the schema" and "shipping a wire
break."

## Why it's directionally correct

- The feature solves a real continuity problem (interrupted
  subagents currently disappear; user has no way to recover them).
- The transcript-first fork resume design is the correct architectural
  choice — reconstructing fork worker context from the parent's
  *current* state is exactly what causes "resumed fork has different
  worker constraints than original" bugs.
- `paused` as a registry status (not a separate dialog) extends the
  existing UX pattern rather than fragmenting it. The dialog's
  `STATUS_VERBS` map at `BackgroundTasksDialog.tsx:48-54` and the
  `terminalStatusPresentation` icon/color map at lines 65-77 absorb
  the new state cleanly.
- `r` and `x` keys for resume/abandon at
  `BackgroundTasksDialog.tsx:495-498, 524-527` mirror the existing `x`
  for cancel-running, with the `selectedEntry?.status === 'paused'`
  guards at lines 542-543 and 547-548 hiding the affordance when the
  selected entry is not paused. Tests at
  `BackgroundTasksDialog.test.tsx:208-227` pin the keypress→action
  wiring.
- Recovery notice is emitted as transient INFO history
  (`AppContainer.tsx:368-376`), not a modal — non-intrusive,
  discoverable.

## Nits / risks

1. **Single PR, 2271 additions, 24 files, no `[1/N]` framing.** This
   is at the size where reviewers can't realistically trace every
   path in one sitting. The natural slices are visible:
   - (a) `BackgroundAgentResumeService` + sidecar discovery only
     (`background-agent-resume.ts` + tests + `config.ts` wiring)
   - (b) Transcript schema extension (`agent-transcript.ts` +
     test + downstream-consumer audit)
   - (c) Tool lifecycle (`send-message.ts`, `task-stop.ts`,
     `subagent-manager.ts`, `agent.ts`)
   - (d) UI wiring (`AppContainer`, `BackgroundTasksDialog`,
     `BackgroundTasksPill`, `useResumeCommand`)
   Splitting at this point is probably not worth re-doing the work,
   but for the next feature of similar size the author should split
   from the start.
2. **Test scope is huge but uneven.** `background-agent-resume.test.ts`
   is +643 lines, which is great. But the tool-layer tests
   (`send-message.test.ts:+27`, `task-stop.test.ts:+24`) are barely
   adequate — these are the lifecycle paths that get *executed* every
   resume. Consider adding integration cases asserting that
   `send-message` to a paused agent either (a) blocks with a
   recoverable error or (b) buffers until resume — the PR body says
   "extended `send_message` and `task_stop` to handle paused
   background agents" but doesn't specify *which* semantics, and the
   test additions don't unambiguously establish them.
3. **`recovered.length > 0` notice timing.** The notice is added to
   history at `AppContainer.tsx:368-376` immediately after
   `historyManager.loadHistory(historyItems)`. If history loading
   itself emits its own startup messages, the recovery notice may
   land in an unpredictable position relative to them. Worth a
   `// notice intentionally appended after loadHistory so it appears
   below restored content` comment.
4. **Concurrent resume coalescing is mentioned in the PR body but
   not visible in the diff snippets I sampled.** ("resumed agents
   now re-run `SubagentStart` hooks, coalesce concurrent resume
   attempts, and append to the original transcript"). For a
   974-line service file, an inline comment at the coalescing site
   explaining the lock/sentinel/whatever semantics would help future
   maintainers. The race is real: user double-presses `r`, two
   resumes interleave, transcript gets dual-appended.
5. **Pre-existing React `act(...)` warnings** acknowledged in PR
   body. Not introduced by this PR but worth filing as a follow-up
   issue so they don't accumulate.
6. **`buildRecoveredBackgroundAgentsNotice(recovered.length)` is
   called from inside `AppContainer.tsx:373`** — a UI component
   reaches into a service to format a string. Acceptable for v1 but
   worth a future move into a `useResumeCommand` selector or a
   `lib/notices.ts` helper for testability.
7. **No CHANGELOG entry mentioned.** This is a user-facing feature
   ("you can now resume interrupted background agents") and deserves
   one. The recovery-notice text alone changes startup output, which
   downstream automation may screen-scrape.
8. **`paused` status icon is `\u23F8` (⏸)** at
   `BackgroundTasksDialog.tsx:67-72` with `theme.status.warning`
   color. Confirm the warning color is the right semantic — paused
   isn't a warning, it's a recoverable interruption. `theme.status.info`
   or a dedicated `theme.status.paused` would read more accurately.
   Cosmetic, not blocking.
9. **Schema versioning.** New `system/agent_bootstrap` and
   `system/agent_launch_prompt` records are added to the transcript
   format with no explicit schema version bump. The downstream
   audit (export/ACP/insight) is good, but a schema version field
   would make future migrations explicit rather than relying on
   "all current consumers ignore unknown subtypes." File a follow-up.

## Verdict

`merge-after-nits` — substantial, well-scoped feature with correct
architectural choices (transcript-first resume, downstream-consumer
audit, `paused` as registry-extension not new dialog). Concerns are
mostly about review surface area (24 files, 2271 adds, no slicing)
and depth of lifecycle-edge testing rather than design. Address the
coalescing-comment, the `paused`-color review, and the
schema-versioning follow-up; lock the lifecycle semantics with one
or two more `send-message`/`task-stop` integration tests; ship.
