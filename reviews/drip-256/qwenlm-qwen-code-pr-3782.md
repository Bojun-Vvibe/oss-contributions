# QwenLM/qwen-code PR #3782 — fix(vscode-companion): fix ESLint curly and eqeqeq violations causing CI failures

- URL: https://github.com/QwenLM/qwen-code/pull/3782
- Head SHA: `1c2b501ac547242d073b13af3d9644652a3b704b`
- Author: B-A-M-N
- Verdict: **merge-as-is**

## Summary

Mechanical lint cleanup across three files in `packages/vscode-ide-companion/src/`:

- `extension.ts`: wraps a single-statement `if (provider.sendCopyCommand(action)) break;` in a brace block.
- `webview/App.tsx`: wraps several single-statement `if`s (`if (child == null) return null;`, `if (!directChild) return -1;`, several `if (...) continue;` and `if (...) return;`) in brace blocks. One change is non-trivial: `if (child == null) return null;` becomes `if (child === null) { return null; }` — a `==` → `===` tightening (so it no longer matches `undefined`).
- `webview/providers/WebViewProvider.ts`: same brace-wrapping for two early-return `if`s in `sendCopyCommand`.

## Line-level observations

- `extension.ts` line ~221: pure `curly` rule fix, no behavior change.
- `webview/App.tsx` line ~179: **behavior change** — `child == null` → `child === null`. Previously, `MessageList`'s map callback dropped both `null` and `undefined` children; now it only drops `null`. If any upstream code can return `undefined` from the message factory, those will now flow through and likely crash the React render (`mapping.push(index)` plus `<React.Fragment key=...>{undefined}</React.Fragment>` is technically valid for React but the `mapping` accounting will diverge from rendered output). Worth confirming the upstream factory's return type. If it's `Element | null`, the change is safe; if it's `Element | null | undefined`, this is a regression.
- `webview/App.tsx` lines ~214, ~1207, ~1240, ~1255: pure `curly` fixes.
- `webview/providers/WebViewProvider.ts` lines ~1740–1746: pure `curly` fixes inside `sendCopyCommand`.

## Why merge-as-is

- The PR title and description scope this as ESLint compliance, and 6 of the 7 changes are pure brace-wrapping with zero semantic effect.
- It unblocks CI, which is high-value.

## Caveat / one suggestion

The single semantic change (`==` → `===`) on `child == null` is subtle. ESLint's `eqeqeq` rule typically allows `== null` as the canonical "null or undefined" idiom (`{ "null": "ignore" }`). If the project's eslint config doesn't have that allowance, the right fix is to add `// eslint-disable-next-line eqeqeq` or to keep the original semantic via `if (child === null || child === undefined)`, **not** to silently narrow the check. Worth a one-liner from the author confirming the upstream return type can never be `undefined`.

## Suggestions

1. Confirm `MessageList`'s child can never be `undefined`; if it can, change line 179 to `if (child === null || child === undefined)` (or restore `== null` with an eslint-disable pragma).
2. Consider tweaking `.eslintrc` to `"eqeqeq": ["error", "always", { "null": "ignore" }]` so future contributors don't have to choose between two flavors of wrong.
