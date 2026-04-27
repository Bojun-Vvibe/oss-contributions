# QwenLM/qwen-code#3657 — feat(vscode): add tab dot indicator and notification system

- **Author**: dreamWB
- **Size**: +766 / -46 across the vscode-ide-companion package
- **Issue**: #3106

## Summary

Adds a three-layer notification system to the VS Code extension so users get visibility into long-running tasks and permission prompts when the Qwen Code panel isn't focused:

1. **Tab dot indicator** — orange (task complete) / blue (needs user input) badge on the tab icon, gated by `qwen-code.dotIndicator` (default `true`).
2. **VS Code information notification** — `window.showInformationMessage(...)` with a "Show" button that focuses the panel.
3. **Platform sound** — best-effort `child_process.exec`/`execFile` to system audio players.

Plus a refactor consolidating duplicate `onDidReceiveMessage` handlers into a shared method.

## Diff inspection

`packages/vscode-ide-companion/package.json:235-251` adds the two new contributions:
- `qwen-code.dotIndicator: boolean` (order 3, default `true`)
- `qwen-code.notifications: boolean` (order 4, default `true`)

The 280+ line test addition at `WebViewProvider.test.ts:1264+` is the substantive review surface:
- New `vi.hoisted()` refs (`endTurnCallbackRef`, `streamChunkCallbackRef`, `permissionRequestCallbackRef`, `askUserQuestionCallbackRef`, `mockShowInformationMessage`, `mockWindowState`) cleanly model the async-callback shape.
- The `vi.mock('child_process', ...)` block at `:299-318` neutralizes audio-player exec calls — necessary since the real exec would shell out to `afplay`/`paplay` during unit tests.
- `beforeEach` resets the four callback refs + `mockWindowState.focused = true` + `mockShowInformationMessage.mockReset()` — disciplined fixture hygiene.
- `vi.useFakeTimers()` in the new describe block lets the 20s threshold test fire deterministically.

Two binary asset additions (`assets/icon-blue.png`, `assets/icon-orange.png`) are unreviewable from text diff but are necessary tab-icon overlays.

## Strengths

- Two boolean settings (default `true`) give users opt-out without an extra "off" code path.
- `idleNotificationSent` / `attentionNotified` guards (mentioned in PR body) prevent duplicate notifications on rapid stream chunk updates — the right shape, even if I can't audit them from the truncated diff.
- Test scaffolding is well-engineered: explicit hoisted refs > implicit module-state mocking.
- Refactor of duplicate `onDidReceiveMessage` handlers is the right cleanup to land alongside the feature.
- 20s threshold for "long task" matches CLI notification semantics — consistency across surfaces.

## Concerns

1. **Linux notification sound depends on `canberra-gtk-play` or `paplay`** (PR body acknowledges). The `child_process.exec` call should swallow ENOENT silently — if it doesn't, every Linux user without those binaries will see a console error per task completion. The mock at `test:299-318` doesn't exercise the not-installed path; needs a `mockedExec.mockImplementationOnce((_, cb) => cb(new Error('ENOENT')))` test to lock in the silent-fail contract.
2. **`window.state.focused` is the only focus signal mocked.** VS Code also exposes `webviewView.visible` and `webviewView.onDidChangeVisibility` — the dot indicator should clear on any visibility transition, not just window-focus. Without the actual notification dispatch code in the diff window I can't tell if both are checked.
3. **Notification guards are stateful (`idleNotificationSent`, `attentionNotified`).** State that needs explicit reset (on view-focus? on session change? on user dismiss?) is a duplicate-suppression bug magnet. Worth a state-machine diagram in the PR body or at minimum a comment block at the field declarations explaining all reset conditions.
4. **Two binary PNGs land without dimension/byte-size confirmation.** VS Code tab icons should be 16×16 SVG-rendered or 32×32 PNG; the file should also be < 4KB to avoid bloating the .vsix. A `file assets/icon-*.png` line in the PR body would settle this.
5. **`qwen-code.dotIndicator` and `qwen-code.notifications` are independent toggles** — that's correct, but the description for `notifications` says "with sound" without exposing a separate sound toggle. Some users want dot+notification but no sound (open-plan offices). Worth a `qwen-code.notificationSound` boolean as a follow-up.
6. **No i18n.** "Show" button text and information-message bodies are hardcoded English. The companion extension targets a global Qwen userbase — the message strings should at minimum be extracted into a `messages.ts` so localization can land later.

## Verdict

**merge-after-nits** — the architecture is sound and test coverage is genuinely good for a VS Code feature, but the Linux ENOENT handling, notification-guard reset semantics, and i18n string extraction should be addressed (or at minimum filed as follow-ups) before merge. The visibility-vs-focus question needs answering too.
