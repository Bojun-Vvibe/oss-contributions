# sst/opencode #25394 — fix(opencode): support env-prefixed plugin paths in config

- URL: https://github.com/sst/opencode/pull/25394
- Head SHA: `75a22ef0f56d4c86a4ce4f56d2be528f3c9411c5`
- Author: @himax12
- Stats: +86 / -2 across 2 files

## Summary

Teaches `resolvePluginSpec` to expand path-variable prefixes
(`~`, `$VAR`, `${VAR}`, `%VAR%`) in plugin specifiers so config files
like `{"plugins": ["$HOME/plugin"]}` resolve correctly across POSIX and
Windows. Adds two new helpers and two tests.

## Specific feedback

- `packages/opencode/src/config/plugin.ts:53-79` — `expandPathVariablePrefix`
  is well-scoped: only the **prefix** is treated as a variable, anchored
  with `(?=$|[\\/])` so `$HOMER` is not accidentally expanded into
  `$HOME` + `R`. Good guard against substring collision.
- `packages/opencode/src/config/plugin.ts:81-95` — `hasPathVariablePrefix`
  duplicates the regex/logic of the expander. Suggest collapsing into a
  single function returning `{ expanded: string, isVariablePrefix: boolean }`
  so the two stay in lockstep when (e.g.) someone later adds support for
  PowerShell `$env:VAR` syntax.
- `packages/opencode/src/config/plugin.ts:97-101` — the new triple-guard
  `if (!isPathPluginSpec(raw) && !hasPathVariablePrefix(raw) && !isPathPluginSpec(spec))`
  is correct but hard to reason about. A single comment explaining "raw
  path-spec OR raw var-prefix OR var-prefix that expanded to a path-spec"
  would help future maintainers.
- `packages/opencode/src/config/plugin.ts:54` — `os.homedir()` already
  consults `$HOME` / `%USERPROFILE%`, so the dedicated `~` branch and the
  `$HOME` / `%USERPROFILE%` branches will return the same string in the
  common case. That's fine, but worth a one-line note so a reader doesn't
  hunt for behavioural differences.
- `packages/opencode/test/config/config.test.ts:1971-2012` — both new
  tests carefully save/restore `process.env.HOME` and `process.env.USERPROFILE`
  with `try/finally`. No coverage for the `${BRACED}` form or the
  not-set-variable fallthrough — easy add.
- Edge case worth a test: `~user` (Unix tilde-with-user) is NOT handled
  and will fall through as an unknown spec. If that's intentional, a
  one-line code comment would help; if not, document the limitation.
- No security concern: expansion is bounded to envvars the user controls
  in their own shell, and the resolved path still goes through the
  existing `resolvePathPluginTarget` / `isPathPluginSpec` validation.

## Verdict

`merge-after-nits` — correct, defensible, adds reasonable test coverage.
The duplicated regex pair and the triple-guard readability are the only
real asks; both are non-blocking.
