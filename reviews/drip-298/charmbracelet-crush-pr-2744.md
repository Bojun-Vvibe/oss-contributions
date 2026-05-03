# Review: charmbracelet/crush PR #2744 — fix(hooks): respect matching directive

- **Head SHA:** `1d689bcbd9833efc0522293358adf7b674f4e9e6`
- **State:** MERGED (2026-04-28)
- **Size:** +202 / -29, 6 files
- **Verdict:** `merge-as-is`

## Summary
Fixes a bug where `matcher` regexes on hooks would match every tool call
after any `SetConfigField`-triggered config reload. Root cause:
`HookConfig.matcherRegex` was an unexported field on the `HookConfig`
struct, and `ReloadFromDisk` rebuilt the config from JSON without re-running
`ValidateHooks` (which was the only place that compiled the regex). After
reload the field was nil, the runner treated nil regex as "match all," and
every PreToolUse hook fired on every tool.

## What the diff actually does

### Ownership move
- `internal/config/config.go:373-383` — `HookConfig` is now a pure data
  struct. The `matcherRegex *regexp.Regexp` field and the `MatcherRegex()`
  accessor are deleted. Comment explicitly calls out the rationale: "matcher
  compilation is owned by hooks.Runner so a JSON round-trip, merge, or
  reload can't silently drop compiled state." Correct architectural
  conclusion from the bug.
- `internal/hooks/runner.go:+46/-8` — Runner now owns matcher compilation.
  Compiled lazily and cached on the runner, not on the config.

### Reload safety net
- `internal/config/store.go:+6` — `ReloadFromDisk` now calls
  `cfg.ValidateHooks()` after merging, mirroring the `Load` path. This is
  belt-and-suspenders given the ownership move (Runner owns compilation
  now), but it's the right defense: validates regex syntax up front so
  users see config errors at load time, not on the first tool call.
- `internal/config/load.go:907-933` — `ValidateHooks` no longer mutates
  `HookConfig.matcherRegex` (since the field is gone); it only validates
  syntax via `regexp.Compile(h.Matcher)` and discards the compiled value.

### Tests
- `internal/config/reload_hooks_test.go:1-102` (new) — Two regression tests:
  1. `TestReloadFromDisk_CompilesHookMatchers` — direct
     `ReloadFromDisk` call after `Load`, verifies a `^bash$` matcher still
     filters non-bash tools out.
  2. `TestSetConfigField_AutoReload_PreservesHookMatcherFiltering` —
     covers the dominant trigger path: `SetConfigField` → autoReload →
     `ReloadFromDisk` → matcher must still filter.
  The `assertHookFilters` helper at lines 65-80 phrases assertions in terms
  of observable Runner behavior (`runner.Run(...).HookCount`), not
  internal field presence — explicitly noted in the test comment, and it's
  the right call: stays valid as ownership moves.
- `internal/hooks/hooks_test.go:+34` — supplements with Runner-level
  matcher coverage.

## Things to like
- The architectural conclusion is well-grounded. A serializable config
  struct that secretly carries non-serializable cached state is a bug
  factory; moving compilation to the consumer (Runner) is the right
  fix shape, not just patching `ReloadFromDisk` to recompile.
- Tests use `t.Setenv` to isolate `HOME`/`XDG_CONFIG_HOME` (lines 23-28,
  88-91) — correctly avoids picking up the developer's real global config.
  No `t.Parallel()`, also correctly noted.
- The `^bash$` example matcher is the worst-case pinpoint: it's a strict
  regex that should reject every non-bash tool, so a broken matcher is
  unambiguous.

## Minor nits (not blocking — already merged)

1. **`assertHookFilters` constructs a fresh `Runner` per assertion**
   (`reload_hooks_test.go:73`). This sidesteps the question of whether a
   long-lived Runner correctly invalidates its compiled regex cache when
   the config is reloaded. Worth a follow-up test that holds one Runner
   across a `SetConfigField` call and verifies the new matcher takes effect.

2. The deleted `MatcherRegex()` accessor was exported, so technically
   API-breaking for anything importing `internal/config` — but `internal/`
   is unimportable from outside the module by Go convention, so this is
   fine.

3. `ReloadFromDisk`'s new `ValidateHooks` call (`store.go:592`) silently
   discards the compiled regex. A comment at the call site noting "we only
   call this for syntax validation; compilation lives in hooks.Runner"
   would prevent a future contributor from "fixing" the apparent waste.

## Verdict rationale
Already merged. The fix correctly identifies the bug as an ownership
problem, not just a missed-recompile, and the test pair locks in both the
direct `ReloadFromDisk` path and the dominant `SetConfigField` autoReload
trigger. Approve.
