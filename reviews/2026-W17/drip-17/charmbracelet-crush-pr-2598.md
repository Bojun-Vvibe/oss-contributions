# charmbracelet/crush PR #2598 — feat: PreToolUse hook

- **Author:** meowgorithm
- **Head SHA:** 06c19827e7bbbb925d106f605b1b884aa1af4edd
- **Files:** 49 files changed, +6791 / −4579 (most diff is regenerated test
  YAML fixtures); the engine is ~1200 lines.
- **Verdict:** `needs-discussion`

## What the diff does

Adds a Crush-native PreToolUse hook system — user-defined shell
commands declared in `crush.json` that fire before each tool call,
with the ability to allow/deny/modify the tool input.

Engine layout (`internal/hooks/`):
- `hooks.go` (+192): event constants, decision types, aggregation
  logic that combines parallel hook outputs into a single
  allow/deny/modify decision.
- `runner.go` (+178): parallel execution with per-hook timeout and
  dedup of identical commands.
- `input.go` (+205): builds the JSON stdin payload, env vars, and
  parses stdout — explicitly designed to be Claude-Code-compatible
  with one intentional divergence (`updated_input` is shallow-merge
  not full-replace, called out in `docs/hooks/README.md`).
- `hooks_test.go` (+622): the engine's own tests.

Wiring:
- `internal/agent/hooked_tool.go` (+119): a `hookedTool` decorator
  that wraps a `tools.Tool` so the agent coordinator runs the
  PreToolUse hook *before* permission checks (intentional, called
  out at `AGENTS.md` line 80–86).
- `internal/agent/coordinator.go` (+15): plumbs hooks through
  during tool registration.
- `internal/config/config.go` (+32) + `internal/config/load.go`
  (+49): adds `hooks` config section + project-vs-global
  precedence (project wins).
- `internal/agent/tools/crush_info.go` (+46) + `.md` + test:
  exposes hook context to agents.
- `internal/skills/builtin/crush-config/SKILL.md` (+112) and
  `crush-hooks/SKILL.md` (+254): in-tree skill docs for the
  feature.
- `docs/hooks/README.md` (+550) and `docs/hooks/examples/
  rtk-rewrite.sh` (+60): user-facing protocol doc + worked example.
- The 4143 net lines of `internal/agent/testdata/.../glm-5.1/*.yaml`
  churn are regenerated tool-call recordings, not real diff.

## Review notes

- **Order of operations is a real design decision.** The PR places
  hooks *before* permission checks ("If a hook denies a tool call,
  you'll never see a permission prompt for it" — `docs/hooks/
  README.md` line 24). That's the right call — permissions are an
  internal trust check, hooks are a user-controlled policy layer —
  but it should be explicit in the security section of the docs that
  a permissive hook can therefore *allow* tools that the permission
  system would otherwise prompt for. A line saying "hooks can grant
  but cannot escalate beyond what the underlying tool would do" is
  worth adding.
- **`updated_input` shallow-merge divergence** from Claude Code is
  the kind of thing that *will* trip users porting hooks. The doc
  calls it out, good — but consider also emitting a one-time warning
  when a ported `replace_input` style payload is detected (full
  replacement). That's a 5-line ergonomic win.
- **No Windows story.** All hooks run as shell commands. The doc
  centers on bash. Crush runs on Windows; either the hook engine
  needs to document that hooks are Unix-only on Windows you'd use
  `cmd /c` / `pwsh -Command`, or `runner.go` needs platform-aware
  shell selection. Currently `runner.go` invokes via `exec.Command`
  with a path; the assumption that the path is executable on Windows
  needs spelling out.
- **Parallel execution + dedup** in `runner.go` is sensible, but
  parallel hooks with side-effects (write to stdout, mutate
  `updated_input`) interact in ways the test suite should pin
  down. `hooks_test.go` covers single-hook decisions extensively;
  worth adding a test that 3 hooks all returning `updated_input`
  patches merge in deterministic order.
- **`hooked_tool.go` decorator pattern** is the right
  abstraction — it keeps the engine independent of the tool
  implementations, as `AGENTS.md` line 79 explicitly states. Good
  separation.
- **Doc README is good.** Particularly the "Hot Hook Facts" framing
  at the top and the explicit Claude-Code-compat note. The
  `rtk-rewrite.sh` example is a small, real, copy-pastable use case
  — exactly what hook docs need.
- **Fixture churn (~4000 lines) is hard to review.** Worth
  regenerating in a separate commit on the same PR so the engine
  diff is easier to read in isolation.

This is a substantial public API surface — JSON wire protocol +
config schema + plugin contract — and worth one more design pass
on Windows behavior, the warning UX for the Claude-Code merge
divergence, and the parallel-merge ordering before locking in.

## What I learned

Hooks-before-permissions is the inversion that makes user-defined
policy actually meaningful: if the trusted internal check ran
first, the user could only ever *additionally* deny, never override
or shape the allow path. Crush picking that order matches
established practice and is worth noting for any future tool-gating
system.
