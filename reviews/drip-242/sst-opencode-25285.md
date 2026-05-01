# Review: sst/opencode #25285 — fix(tui): retain current model and agent across session switches

- URL: https://github.com/sst/opencode/pull/25285
- Head SHA: `f1abed5c502209d5d80c478e1ddd5a8bb3f05aee`
- Base: `dev`
- Size: 1 file, +19/-13
- Closes: #4930

## Summary of intended change

Inside `Prompt`, a `createEffect` watching `props.sessionID` was rewriting the
global `local.agent` / `local.model` / `local.model.variant` stores from the
last user message of every newly-loaded session. Result: switching sessions via
`Ctrl+X l` clobbered the user's currently-selected agent/model with whatever
the picked session was historically running on. The fix flips this restoration
into a **one-shot**: it still fires on the first session load (so startup
defaults still come from the most-recent session), but every subsequent
session switch leaves the active agent/model untouched.

## Review

The diff is small and entirely contained in
`packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:241-272`. The
mechanism is two flags both closed over by the effect:

- `syncedSessionID` — already present pre-PR, prevents re-running the body
  for the same session id.
- `initialized` (new at `:246`) — `let` outside the effect, flipped to
  `true` exactly once at `:255` after the first qualifying session has been
  observed.

The control-flow rewrite at `:248-256` collapses the prior nested
`if (sessionID !== syncedSessionID) { if (!sessionID || !msg) return; ... }`
into early-returns, then the new `if (initialized) return;` gate. One subtle
but correct detail: `syncedSessionID = sessionID` at `:253` runs **before**
the `initialized` check at `:255-256`. That means once a session has been
observed (even if it's the first one), the id is recorded as "synced" so a
later effect re-trigger for the same id short-circuits at `:248` regardless
of whether `initialized` was already `true`. Without that ordering the
first-session path would still work but later toggles back to the first
session could redundantly re-enter the body.

### Edge cases worth pinning

1. **First session has no last-user message** (`msg` falsy at `:249`): the
   effect early-returns at `:249` *before* setting `syncedSessionID` or
   flipping `initialized`. Good — that means a subsequent prop change to a
   different session that *does* have a `msg` correctly counts as the "first
   real" initialization. But it also means: if the user starts on an empty
   session, switches to a populated one, the populated one's agent/model
   *will* override the user's current selection. That is arguably the
   intended definition of "first session that loads", but worth a one-line
   comment near `:249-250` calling it out so future readers don't misread
   `initialized` as "first effect-fire".

2. **`createEffect` is per-component instance**. If the `Prompt` component
   ever unmounts and remounts (e.g. workspace switch or HMR in dev), the
   `let initialized = false` resets and the next session load will again
   re-sync. Probably fine — that's the same shape as a fresh process start —
   but worth confirming whether `Prompt` is stable across the
   `Ctrl+X l`/`Ctrl+X w` flows being fixed.

3. **`args.agent` interaction** at `:262`: preserved unchanged; if the user
   started codex with `--agent X`, the *first* session load also won't
   override the agent. Model still gets set. That asymmetry was pre-existing
   so out of scope, but a follow-up to also gate `local.model.set(msg.model)`
   on a `--model` CLI flag would close the symmetry.

## Verdict

**merge-after-nits**

Wants:
- One-line comment near `:249-250` clarifying that `initialized` flips on
  the first session that actually has a `msg`, not on the first effect fire,
  so future readers don't misread the gate.
- Optional: a tiny test asserting that two `setProps({ sessionID })` calls
  in sequence only push to `local.agent`/`local.model` once; given the file
  has no existing test harness, a manual repro note in the PR body covering
  the `Ctrl+X l` flow would be acceptable.

The fix itself is correct and minimal.
