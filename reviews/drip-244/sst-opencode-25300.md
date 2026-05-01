# PR #25300 — feat(tui): opt-in `confirm_exit` setting requires double Ctrl+C to quit

- Repo: sst/opencode
- Head: `2974d6d60395ae9785f233455b6386c804065e5d`
- URL: https://github.com/sst/opencode/pull/25300
- Verdict: **merge-after-nits**

## What lands

Implements #15932 as opt-in: new `confirm_exit: z.boolean().optional()` field on
`TuiOptions` at `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts:27-32`,
and a 13-line guard inside the existing `app_exit` keybind branch at
`packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:1112-1127` that
shows a 2 s "Press again to exit" toast on the first press and only calls
`exit()` on a second press inside `EXIT_CONFIRM_WINDOW_MS`. Default `false`
preserves the current single-press behavior.

Scope is correctly tight: the guard sits inside the existing
`if (store.prompt.input === "")` arm so dialog-cancel and sub-session jump-back
paths that also call `app_exit` keep their semantics.

## Nits (the load-bearing one)

`lastExitAttempt` is declared with `let lastExitAttempt = 0` at component-body
top-level on line 110, *not* inside a `createSignal` / `useRef`-equivalent.
Solid re-runs the component body on prop changes, so any re-render of
`Prompt` (sessionID change, args change, etc.) would reset
`lastExitAttempt = 0` and break the 2 s window — the user would have to press
twice within the same render cycle. This needs to be a `let lastExitAttempt = 0`
declared *outside* the component or held in a ref-like primitive
(`createSignal` with `untrack`, or stash on the closure of the keybind handler
with `useStore`-style state). Verify by re-rendering Prompt between two
Ctrl+Cs — current implementation will silently drop the first attempt.

## Other small things

- `EXIT_CONFIRM_WINDOW_MS = 2000` is reused for both the timeout window *and*
  the toast `duration`. Reasonable coupling but worth a one-line comment so a
  future tweak doesn't accidentally desync them.
- The toast variant is `"info"`; the existing pattern for "you almost did
  something destructive" is closer to `"warning"`. Minor.
- No unit test added covering the two-press path. A small test in
  `test/config/tui.test.ts` (or a sibling) wrapping the prompt with a fake
  keybind would lock the contract.

## Why merge-after-nits not merge-as-is

The `let lastExitAttempt` placement is a real correctness bug under Solid's
re-render model, not a style nit. Author also notes one pre-existing tui.test
failure that's unrelated, fine.