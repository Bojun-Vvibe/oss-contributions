# charmbracelet/crush PR #2730 — feat(hooks): use embedded shell by default

- URL: https://github.com/charmbracelet/crush/pull/2730
- Head SHA: `5dacf9e40568e6f027ff2964c49617ddd25d75bb`
- Verdict: **merge-after-nits**

## What it does

Routes hook command execution through Crush's embedded mvdan POSIX shell
interpreter by default instead of shelling out to `sh`/`bash`. The
practical wins:

- **Windows**: hooks become functional without WSL or Git Bash on PATH.
- **Builtins**: hooks gain access to the same builtin handlers as the
  in-app `bash` tool (since `standardHandlers()` is now shared).
- **Performance**: no fork/exec to a system shell.

Files with an explicit shebang (`#!/usr/bin/env bash`, `#!/usr/bin/env
python3`) still dispatch to the named interpreter; hooks pointing
directly at a binary (`node ./script.js`) also bypass the embedded
interpreter.

## Specifics I checked

- `.agents/skills/shell-builtins/SKILL.md` is updated to reflect that
  `builtinHandler()` has moved from `internal/shell/shell.go` to
  `internal/shell/run.go` and is shared between the stateful `Shell`
  type and a new stateless `Run` entrypoint used by the hook runner.
  This is the load-bearing structural change; the rest is wiring.
- The skill doc adds an explicit "**Poll `ctx` in every unbounded
  loop**" rule (step 4) with a concrete code example, plus the contract
  that builtins should return `ctx.Err()` (not `interp.ExitStatus(n)`)
  on cancellation. Crucial for hook timeouts to actually interrupt
  long-running builtins.
- `docs/hooks/FUTURE.md` deletes ~120 lines of "Cross-platform shell
  (Windows support)" planning doc that this PR realizes — confirming
  the design previously lived in FUTURE.md and is now done.

## What I like

- The dispatch matrix in the (now-removed) FUTURE.md doc is the right
  mental model and matches what's been implemented:
  `./script.sh (no shebang)` → mvdan, `./script.sh (#!/bin/bash)` →
  `bash ./script.sh`, `./script.exe` → direct exec.
- One fresh `interp.Runner` per hook invocation — addresses the parallel
  hook state-leakage hazard called out in FUTURE.md.
- Context cancellation is plumbed through to `exec.CommandContext` for
  shebang-dispatched children, so timeouts kill grandchildren.

## Concerns / nits

1. **The diff I can see is mostly skill-doc + FUTURE.md deletion.**
   The actual `internal/shell/run.go` and `internal/hooks/runner.go`
   changes weren't in the truncated portion of the PR I reviewed. Worth
   confirming in the full diff that:
   (a) `Run` does not share the `interp.Runner` between hook
   invocations,
   (b) the dispatch heuristic for "is this a path?" matches the
   FUTURE.md contract (`./`, `../`, `/`, `~/` only — `relative/path.sh`
   is correctly mvdan'd, not file-dispatched),
   (c) shebang parsing handles `#!/usr/bin/env -S NAME args` and CRLF
   endings as the contract promised.
2. **No bundled bash on Windows.** Right design call — but the user
   message when a hook with `#!/bin/bash` runs on Windows without bash
   on PATH should be clear ("interpreter `bash` not found on PATH; the
   hook's shebang requires bash"), not a generic exec error. Worth a
   smoke test.
3. **Existing hooks already authored against system `sh`** may rely on
   non-POSIX bashisms (`[[ ]]`, `<<<` herestrings, arrays). These will
   silently change behavior under mvdan. A short CHANGELOG note
   recommending hook authors keep to POSIX or add an explicit
   `#!/usr/bin/env bash` shebang would help.
4. **Backward compat opt-out?** Some users may rely on the system
   shell's environment. No flag to force the old behavior. Worth
   considering, even if "use a `#!/bin/sh` shebang to opt back into the
   system shell" is a valid answer to bake into the docs.

## Risk

Medium for existing hook authors who depended on bashisms; low for
everyone else. Net positive — Windows users gain hooks, performance
improves, builtins become uniformly available. Verdict
`merge-after-nits` pending the verification points in (1) and a docs
note on (3).
