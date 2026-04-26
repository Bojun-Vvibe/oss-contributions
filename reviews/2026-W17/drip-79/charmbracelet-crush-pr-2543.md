# charmbracelet/crush PR #2543 — Fix: avoid unsupported terminal capability probe

- **PR:** https://github.com/charmbracelet/crush/pull/2543
- **Author:** owldev127
- **Head SHA:** `99948d5ebbfd7b90c747f9d55c5cf7736fd34f87`
- **Files:** 2 (+30 / -4)
- **Verdict:** `merge-after-nits`

## What it does

Rewrites the boolean logic in `shouldQueryCapabilities` so that crush stops probing terminals that don't support the `XTVERSION` escape sequence. Previously, the function would probe any terminal where `TERM_PROGRAM` was unset *and* `SSH_TTY` was unset — which incorrectly included plain `xterm-256color` sessions on Linux consoles (no `TERM_PROGRAM`), causing the terminal to receive an unrecognized OSC sequence that some terminals echo back as garbage.

## Specific reads

- `internal/ui/common/capabilities.go:127-137` — old logic was a single boolean expression that combined four conditions with `||`:
  ```go
  return (!okTermProg && !okSSHTTY) ||
      (!strings.Contains(termProg, osVendorTypeApple) && !okSSHTTY) ||
      xstrings.ContainsAnyOf(termType, kittyTerminals...)
  ```
  The first two clauses essentially said "no `TERM_PROGRAM` and not under SSH" → probe; this caught generic `xterm` users.
  New logic is explicit and short-circuits cleanly:
  ```go
  if okSSHTTY {
      return false   // never probe over SSH
  }
  if xstrings.ContainsAnyOf(termType, kittyTerminals...) {
      return true    // always probe Kitty-family
  }
  return okTermProg  // probe only when we know what terminal program we're in
  ```
  This is strictly narrower than before. The Apple-vendor early return at line 127 is preserved (Apple Terminal already handled separately above).
- `internal/ui/common/capabilities_test.go:1-23` — new test file with one parallel test confirming `TERM=xterm-256color` no longer triggers the probe. Minimal but exactly the regression case.

## Risk surface

Low.

- The change is more conservative (fewer probes) so the worst case is "user with an undetected XTVERSION-capable terminal stops getting features that depend on the probe response" — degrades to the no-probe baseline, doesn't break anything.
- No SSH probing is preserved (good — SSH multiplexing makes capability probes flaky).
- Kitty-family detection is preserved verbatim via `xstrings.ContainsAnyOf(termType, kittyTerminals...)`.
- The `okTermProg` final branch keeps probing for known-good terminal programs (iTerm2, WezTerm, etc. that set `TERM_PROGRAM`).

## Nits (not blocking)

1. **Test coverage is one case.** Worth adding three more table-driven cases: (a) `TERM=xterm-kitty` → `true`, (b) `SSH_TTY=/dev/pts/0` → `false`, (c) `TERM_PROGRAM=iTerm.app` → `true`. With the rewrite, each branch is testable in 3 lines. The current single test only exercises one fall-through path.
2. The Apple early-return at line 127 (preserved) checks `termProg` for `osVendorTypeApple` — but `termProg` could be empty there since `okTermProg` isn't tested. `strings.Contains("", "Apple")` returns `false` so this is safe today, just noisy. Worth a `okTermProg && strings.Contains(...)` for clarity. Pre-existing, not introduced by this PR.
3. PR title prefix is `Fix:` instead of the conventional `fix(ui):` used by this repo (the immediately-adjacent open PRs all use the conventional prefix). Cosmetic but easy to align.

Verdict: merge after the table-driven test expansion (3 more cases). The logic is correct, narrower than before, and the rewrite reads cleanly.
