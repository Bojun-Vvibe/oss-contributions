# QwenLM/qwen-code#3865 — feat(base): persist channel sessions across restarts

- PR: https://github.com/QwenLM/qwen-code/pull/3865
- Head SHA: `77d73d0b84cad0b0379c9d04e2f333cb40dd7741`
- Size: +23/-11
- Verdict: **merge-after-nits** (man)

## Shape

Three-file surgical change that flips channel-session persistence from
"clear-on-shutdown" to "persist-and-restore-on-startup":

1. `packages/channels/base/src/AcpBridge.ts:135-140` — `loadSession()` now
   forwards the existing `sessionId` straight back to the caller instead of
   round-tripping through `response.sessionId` from the ACP server. Subtle
   correctness change: lets the caller use a stable, externally-issued session
   identifier instead of whatever the server allocates on resume.
2. `packages/channels/base/src/SessionRouter.ts:199-205` —
   `clearAll()` no longer `unlinkSync(this.persistPath)` on shutdown. The
   persist file is preserved so the next process start can call
   `restoreSessions()` and pick up where the previous run left off. The
   import of `unlinkSync` is removed at `:1`.
3. `packages/cli/src/commands/channel/start.ts:218-226` and `:377-385` — both
   `startSingle` and `startAll` now invoke `await router.restoreSessions()`
   after `registerToolCallDispatch` and before `channel.connect()`, with a
   `[Channel] Sessions restored: N[, failed: M]` stdout line. The placement
   is correct (router must exist; channel must not yet be live).

## Notable observations

- `loadSession` returning the input `sessionId` instead of `response.sessionId`
  at `AcpBridge.ts:140` is a subtle behavior change worth flagging in the PR
  body. If the server's ACP `loadSession` impl ever needed to issue a
  *different* session ID (e.g., on collision/migration), this PR silently
  drops that ability. For the persistence use case it's correct, but a
  comment like `// session ID is caller-issued for persistence; ignore
  any server-side renaming` would document the deliberate choice.
- `clearAll()` docstring at `SessionRouter.ts:199-200` correctly states the
  new contract: *"Clear in-memory state on shutdown. Persist file is kept so
  sessions can be restored on next start via restoreSessions"*. Good — the
  rename of behavior deserves the rename of doc.
- The `[Channel] Sessions restored: N` log at `start.ts:221` is plain stdout
  (not structured); fine for a CLI tool but consider whether the channel-
  startup banner in `start.ts` already uses a more structured logger that
  this should plug into.

## Concerns / nits

- No test added. The two contracts that need pinning are: (a) restart cycle
  preserves an active session (run → register session → SIGTERM → run →
  assert session still routable), and (b) corrupted persist file is recovered
  gracefully (failed > 0, restored = 0, no crash).
- `restoreResult.failed > 0` triggers a log message but not a non-zero exit
  code or any user-visible warning. If 100 sessions failed to restore, the
  user just sees `Sessions restored: 0, failed: 100` and proceeds. Consider
  promoting `failed > restored * 0.5` to a warning-level log with hint to
  inspect the persist file.
- The missing `unlinkSync` import at `SessionRouter.ts:1` is correct, but the
  `existsSync` import is preserved — verify no other call site in the file
  was using `unlinkSync` (the visible diff shows only the one use removed).
- Loop ordering in `start.ts:218-226` puts `restoreSessions()` *before*
  `channel.connect()`. If `connect()` fails, the in-memory router state is
  populated but the channel never came up — does subsequent retry/reconnect
  see the restored state cleanly, or does it double-restore?
