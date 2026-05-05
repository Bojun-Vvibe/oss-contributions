# charmbracelet/crush #2543 — Fix: avoid unsupported terminal capability probe

- **Head SHA:** `99948d5ebbfd7b90c747f9d55c5cf7736fd34f87`
- **Base:** `main`
- **Author:** owldev127
- **Size:** +30 / −4 across 2 files (`internal/ui/common/capabilities.go`, new `internal/ui/common/capabilities_test.go`)
- **Verdict:** `merge-after-nits`

## Summary

Closes #2523. Fixes a real "raw terminal control characters left on
screen after exit" bug on xterm-256color terminals where Crush was
sending a Kitty graphics capability query (XTVERSION /
`\033[>q`-class probe) to terminals that don't support it. The
unsupported-terminal then echoes the unhandled probe bytes back to the
TTY, leaving visible junk after Crush quits — especially noticeable on
Ctrl+C exits where the response-drain window is shorter.

## What's right

- **The bug is real and the diagnosis is correct.** The pre-PR
  predicate at `capabilities.go:130-133` was a 3-clause boolean OR:
  `(!okTermProg && !okSSHTTY) || (!strings.Contains(termProg,
  osVendorTypeApple) && !okSSHTTY) || ContainsAnyOf(termType,
  kittyTerminals...)`. The middle clause is actively wrong: when
  `okTermProg = false` (terminal doesn't set `TERM_PROGRAM`),
  `strings.Contains(termProg, osVendorTypeApple)` evaluates against
  the empty string, which returns `false`, which negated is `true`
  — so any terminal without `TERM_PROGRAM` set passed the second
  clause and got probed regardless of `kittyTerminals` membership.
  Plain xterm-256color in tmux without `TERM_PROGRAM` is exactly that
  case.

- **The new predicate at `capabilities.go:130-137` is structurally
  sound:**
  1. Apple-Terminal short-circuit returned at `:127-129` (existing,
     unchanged).
  2. SSH short-circuit at `:130-132` (`if okSSHTTY { return false }`)
     — never probe over SSH because the response can interleave with
     unrelated terminal traffic on the wire.
  3. Known-Kitty short-circuit at `:133-135` (`if
     ContainsAnyOf(termType, kittyTerminals...) { return true }`) —
     anything in the kittyTerminals list is known to handle XTVERSION.
  4. Final gate at `:136`: `return okTermProg` — only probe if a
     `TERM_PROGRAM` env var is actually set (i.e., we have positive
     evidence we're in a terminal emulator that self-identifies).
     Plain `TERM=xterm-256color` with no `TERM_PROGRAM` returns
     false now → no probe → no junk.

- **The new test at `capabilities_test.go:11-22` pins exactly the
  regression case:** `TERM=xterm-256color` with no `TERM_PROGRAM`
  must return false. This is the bug-fix invariant in code form. If
  someone later "improves" the predicate and re-introduces the bug,
  this test catches it.

- **`t.Parallel()` at both the outer and subtest level** at lines
  10 and 14 — proper Go test hygiene, makes the suite run
  concurrently with siblings and within itself.

- **Pure logic refactor in `capabilities.go`** — the function
  signature, return type, and call sites are unchanged. Only the
  branch structure inside `shouldQueryCapabilities` is rewritten.
  No public API churn, no consumers need updates.

## Nits (worth fixing before merge)

- **Test coverage is thin for a predicate-rewrite.** The new test
  pins the regression case, but doesn't pin the *positive* paths
  that the new predicate must continue to allow:
  - `TERM=xterm-kitty` (or any `kittyTerminals` member) with no
    `TERM_PROGRAM` — should return true (Kitty branch at `:133-135`).
  - `TERM_PROGRAM=iTerm.app` with `TERM=xterm-256color` — should
    return true (final-gate branch at `:136`).
  - `TERM_PROGRAM=Apple_Terminal` — should return false
    (Apple-vendor branch at `:127-129`).
  - `SSH_TTY=/dev/pts/0` regardless of other env — should return
    false (SSH branch at `:130-132`).

  Four-row table-driven test would be ~20 lines and would lock down
  the new predicate's full truth table, not just the one regression
  case.

- **The deleted second clause** (`!strings.Contains(termProg,
  osVendorTypeApple) && !okSSHTTY`) was the actual broken logic —
  worth a one-line code comment in the new predicate explaining
  *why* the rewrite, e.g. `// Probe only when we have positive
  evidence (TERM_PROGRAM set, or known-Kitty TERM type).
  Pre-#2523 this fell through for unset TERM_PROGRAM.` Future
  maintainers reading just the file shouldn't need to dig out the
  PR.

- **`okTermProg` semantics** — the variable name suggests "TERM_PROGRAM
  was successfully read from env." The new final-gate `return
  okTermProg` reads as "probe iff TERM_PROGRAM is set," which is
  correct but worth verifying that `okTermProg` is actually the
  `_, ok := env.LookupEnv("TERM_PROGRAM")` shape (not "ok =
  termProg != """). The deleted clause used it both ways which made
  the predicate hard to read; the new code should be unambiguous.

- **Test file comment header** would help — single-purpose test files
  benefit from a one-line file comment naming the issue they regress
  against (`// Regression test for #2523: avoid probing
  xterm-256color terminals that don't support XTVERSION.`).

## Risks

- **Behavior change is intentional and narrow:** unknown
  non-Apple-non-Kitty-non-SSH terminals without `TERM_PROGRAM` now
  skip the probe. The cost is "we don't learn capabilities for some
  terminals that *might* have supported it" — but those terminals
  were exactly the ones echoing junk back, so silence is strictly
  better than corruption.
- **Apple Terminal path unchanged** — the early-return at
  `:127-129` still fires before any other check, so users on
  Apple Terminal see no behavior change.
- **SSH path now strictly enforced** — pre-PR, the second OR clause
  could theoretically allow SSH-with-non-Apple-terminal through the
  probe (because `!okSSHTTY` was true... wait, `okSSHTTY=true` in
  SSH would make `!okSSHTTY=false`, killing that clause). Re-reading:
  pre-PR SSH was already gated. Post-PR, SSH is gated explicitly at
  the top — same outcome, more obvious code.

## Verdict reasoning

`merge-after-nits`: real bug, correct fix, regression test pins the
narrow case. The nits are about test coverage breadth and code
documentation; they would harden the PR but the current shape is
already a strict improvement over the broken predicate.
