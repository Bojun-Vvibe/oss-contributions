# QwenLM/qwen-code PR #3661 — feat(vscode): add tab dot indicator and notification system (#3106)

- URL: https://github.com/QwenLM/qwen-code/pull/3661
- Head SHA: `09549c0d50f12042d8be278338cfaa20e6dc17a7`
- Diff: +766 / -46 across 5 files
- Author: external contributor (dreamWB)

## Context / Problem

The VS Code extension previously had no out-of-tab signaling: a long-running
task could complete or a permission/`askUserQuestion` could fire while the
user was in another editor, with no visual or audio cue. The CLI already had
`useAttentionNotifications`; this PR ports that intent to the IDE companion.

## Design — three layers

1. **Tab dot indicator** (`assets/icon-blue.png`, `assets/icon-orange.png`,
   wired via the existing icon-swap path) — orange = task completed, blue =
   action needed. Fires only when `panel.active === false` (tab not focused).
   Blue takes priority over orange and is never downgraded; cleared on
   user-returns-to-tab.
2. **VS Code notification bubble + "Show" button** — gated by `!(window
   focused AND panel visible)` plus the 20-second floor for the
   task-completed flavor (action-needed fires immediately). "Show" focuses
   the Qwen Code panel. The truth-table in the PR body covers all four
   combinations of (window focused, panel visible) and matches what the
   guards `idleNotificationSent`/`attentionNotified` enforce in code.
3. **Platform sound** — macOS `afplay`, Windows `SystemSounds.Asterisk`,
   Linux `canberra-gtk-play` / `paplay` fallback chain.

Two new boolean settings at `package.json:235-247` —
`qwen-code.dotIndicator` and `qwen-code.notifications`, both default `true`,
documented with the actual trigger semantics in `markdownDescription`.
Backward-compatible defaults are reasonable for a feature most users will
want; opt-out is one toggle.

Test scaffolding at `WebViewProvider.test.ts:1-90` adds four hoisted
callback refs (`endTurnCallbackRef`, `streamChunkCallbackRef`,
`permissionRequestCallbackRef`, `askUserQuestionCallbackRef`),
`mockShowInformationMessage`, and `mockWindowState: { focused: true }` to
make focus state deterministic. The hoisted-ref pattern is the right shape
for testing the `onDidReceiveMessage` consolidation refactor — without it
the four trigger paths can't be exercised independently in vitest's
module-mock model.

## Risks / nits

1. **Linux sound path is exec'd but not mocked**: `child_process` is
   stubbed in the test but the *specific* `canberra-gtk-play → paplay`
   fallback isn't exercised. ENOENT on both binaries is the realistic
   failure (e.g. on a headless Codespace) and the current branch will
   silently swallow it. Add a "ENOENT-on-both → no throw" pin.
2. **Focus-vs-visibility coverage gap**: the truth-table has four cells
   but I don't see test pins for the `webviewView.visible === false` arm
   independently from `window.state.focused === false`. The two flags are
   conceptually different (panel can be visible but tab unfocused, or
   focused-window-but-collapsed-panel) and the bug they prevent only
   reproduces in the off-diagonal cells.
3. **Notification-guard reset semantics undocumented**: when do
   `idleNotificationSent` and `attentionNotified` reset? The PR body
   asserts "guards prevent duplicate notifications" but the reset
   trigger (next user message? new turn? panel-focus event?) needs a
   one-paragraph state-machine comment in the source, otherwise the
   next maintainer will re-derive it from grep.
4. **Hardcoded English strings**: the "Show" button label and the
   notification body text aren't routed through the `i18n` extraction
   path. Other VS Code surfaces in this package presumably are; the
   eight-locale parity check should run before merge.
5. **20-second floor is hardcoded**: should it be a third setting
   (`qwen-code.notifications.minTaskDurationMs`)? Power users on slow
   networks might want 5s; users on fast local models might want 60s.
   Not a block — defaults are sane.

## Verdict

`merge-after-nits` — the three-layer design is well-thought-out, the
truth-table maps cleanly to the guards, the hoisted-callback test
scaffolding is disciplined, and the settings defaults are
backward-compatible. Linux ENOENT pin, focus-vs-visibility split, the
guard-reset comment, and i18n extraction are all addressable in a follow-up
commit on the same PR.

## What I learned

For an IDE notification feature, the truth-table over (window focused,
panel visible) is the contract — write it in the PR body *and* the source,
because the next person to "fix the spam" will re-derive it incorrectly
otherwise. The 20s asymmetry (task-completed gated, action-needed
immediate) is the kind of UX subtlety that disappears from documentation
the moment someone "simplifies" the gate predicate.
