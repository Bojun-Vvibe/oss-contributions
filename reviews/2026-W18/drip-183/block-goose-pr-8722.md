---
pr: block/goose#8722
sha: 8eae573cde1469ee772fbf3540b913ae9a8b933d
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: reuse goose2 vite server on port conflict

URL: https://github.com/block/goose/pull/8722
Files: `ui/goose2/justfile`, `ui/goose2/scripts/start-dev-vite.sh` (new)
Diff: 41+/8-

## Context

`ui/goose2/justfile` derives a stable Vite dev port from the working
directory hash (`vite_port` recipe-level binding). When a developer ran
`just goose2 dev` twice (e.g. relaunched after a Tauri crash, or ran
`dev` and `dev-debug` back-to-back), the second invocation hit a Vite
`--strictPort` failure and tore down the relaunch even though the existing
server was a perfectly usable Goose2 dev instance pointing at the same
project directory. The author replaces inline Vite-startup with a small
helper that detects "is the port owner *our* dev server?" and either
reuses it or fails loud.

## What's good

- Renames `vite_port` → `default_vite_port` (`justfile:1-3`) and adds the
  `VITE_PORT="${VITE_PORT:-$DEFAULT_VITE_PORT}"` indirection at all four
  recipe sites (`dev` `:86`, `dev-debug` `:136`, `kill` `:190`). Net effect:
  the hash-derived port is still the default, but `VITE_PORT=5173 just
  goose2 dev` is now a supported escape hatch — addresses the "what if
  someone else has my hash port?" follow-up implied by the bug.
- New helper `scripts/start-dev-vite.sh:1-29` does the right detection
  pair: `lsof -ti :PORT` for PID, then `lsof -a -p PID -d cwd -Fn` for
  the holder's `cwd`, and `ps -p PID -o args=` for the command line. The
  reuse predicate at `:90` requires *both* `PROC_CWD == PROJECT_DIR` and
  `PROC_ARGS == *"vite --port ${VITE_PORT}"*` — that conjunction is the
  load-bearing safety check (won't reuse a vite server pointed at a
  different project, won't reuse a non-vite process that happens to hold
  the port).
- Failure path at `:95-98` prints both the holder identity and the exact
  recovery command (`just goose2 kill` or `VITE_PORT=<free-port> just
  goose2 dev`) — operator-friendly error message.
- The Tauri `beforeDevCommand.script` indirection at `justfile:103,153`
  changes from `cd ${PROJECT_DIR} && exec pnpm exec vite ...` to
  `exec ./scripts/start-dev-vite.sh ${VITE_PORT}` with `cwd: ${PROJECT_DIR}`,
  so the reuse logic is now uniformly applied across both `dev` and
  `dev-debug` paths instead of being duplicated.
- The `kill` recipe at `justfile:189-194` got the same `default_vite_port`
  treatment so `VITE_PORT=5173 just goose2 kill` actually kills the right
  thing.

## Nits

- `start-dev-vite.sh:90` uses the substring match `*"vite --port ${VITE_PORT}"*`
  on `ps args=`. If a hypothetical sibling tool ever spawns `vite --port
  $PORT --mode preview`, the reuse path will eat it — tighten with an
  explicit `vite --port ${VITE_PORT} --strictPort` match (the actual
  spawned command), since `--strictPort` is uniquely ours.
- `lsof -a -p "${PID}" -d cwd -Fn 2>/dev/null | sed -n 's/^n//p' | head -1`
  at `:89` is correct on macOS but on Linux `lsof` may emit additional
  metadata fields with `-Fn` — worth a comment that this is macOS-tested
  and add a Linux smoke test if Goose2 dev is supported there.
- The new script lacks a shebang test (`bash` vs `/usr/bin/env bash`
  is fine, but `set -euo pipefail` will mask the "PID held but lookup
  failed" case at `:88` because the trailing `|| true` swallows it.
  Consider an explicit `if PID && holder-info-missing → bail loud` arm.
- `dev-debug` at `:135` adds `PROJECT_DIR=$(pwd)` but `dev` at `:84` did
  not need it because the original used `${PROJECT_DIR}` already. Quick
  audit to ensure the `dev`-recipe `${PROJECT_DIR}` assignment is in
  fact set somewhere upstream (it's referenced at `:103` and `:153`).
- The change re-routes the Tauri config to use a script file rather than
  an inline command — worth a one-line note in the goose2 README so
  contributors know `start-dev-vite.sh` is the source of truth.

## Verdict reasoning

Correct UX fix for a real recurring developer pain point, with the
load-bearing identity check (cwd AND args) implemented properly so reuse
can't accidentally hijack a wrong process. Nits are about hardening the
match string, Linux portability, and one README touch — none blocking.
