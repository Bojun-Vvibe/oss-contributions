---
pr: 2519
repo: charmbracelet/crush
sha: 57e5ba2008acd5be7a8b400e5b6c13479c96e9c4
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2519 — fix(config): honor initialize_as context files

- **Author**: officialasishkumar
- **Head SHA**: 57e5ba2008acd5be7a8b400e5b6c13479c96e9c4
- **Link**: https://github.com/charmbracelet/crush/pull/2519
- **Size**: +82/-23 across `internal/config/init.go`, `internal/config/load.go`, plus `init_test.go` (new) and `load_test.go`.

## Scope

`ProjectNeedsInitialization()` was using `contextPathsExist(workingDir)` which only ever scanned files in `workingDir` and matched against the **hard-coded** `defaultContextPaths` filename list. That means a project that set `options.initialize_as: "docs/AGENTS.md"` or `"bazel-bin/AGENTS.md"` would still be flagged as needing initialization (the wizard would re-create the context file) because the existing context lived at a non-default location. Fix routes the existence check through the resolved `Options.ContextPaths` list, which now includes `Options.InitializeAs`.

## Specific findings

- `init.go:45` — call site changes from `contextPathsExist(store.WorkingDir())` to `contextPathsExist(store)`. The store reference is passed through so the new function can read both `Config().Options.ContextPaths` and call `store.Resolve()` on `$VAR`-style entries.
- `init.go:65-83` — `contextPathsExist(store)` now iterates `store.Config().Options.ContextPaths`, calls `resolveContextPath()` per entry, and `os.Stat()`s each. Errors other than `IsNotExist` propagate; `IsNotExist` is silently treated as "this one doesn't exist, try the next". Correct three-way handling (exists / not-exists / hard-error).
- `init.go:85-100` — new `resolveContextPath()`: applies `home.Long(path)` for `~` expansion, dispatches to `store.Resolve(path)` for `$VAR` references, and falls back to `filepath.Join(store.WorkingDir(), path)` for relative paths. Order is right (`~` → `$VAR` → relative-to-workdir → absolute pass-through).
- `load.go:405-412` — `c.Options.InitializeAs = cmp.Or(c.Options.InitializeAs, defaultInitializeAs)` is now hoisted **above** the `ContextPaths` defaulting block (was at `:445` before, now at `:405`). Then `c.Options.ContextPaths = append(c.Options.ContextPaths, c.Options.InitializeAs)` appends the resolved initialize-target into the path list, followed by the existing `slices.Sort` + `slices.Compact` dedupe. This is the structural fix that makes the new `contextPathsExist` actually find the user's chosen target.
- `init_test.go:11-35` — new test `TestProjectNeedsInitializationRespectsInitializeAsPath` constructs a workdir with `bazel-bin/AGENTS.md` (gitignored), sets `InitializeAs: "bazel-bin/AGENTS.md"`, and asserts `needsInitialization == false`. Hits the exact regression. Uses `testStore(cfg)` helper plus a manual `store.workingDir = workingDir` assignment, which suggests the package has a private constructor — fine for an in-package test.
- `load_test.go:63-77` — `TestConfig_setDefaultsIncludesCustomInitializeAsInContextPaths` asserts that a custom `InitializeAs: "docs/LLMs.md"` ends up in `cfg.Options.ContextPaths` after `setDefaults`. Locks down the exact mechanism the init.go fix depends on.

## Risks

Negligible. The `os.Stat`/`IsNotExist` handling is correct, the path resolver covers `~`/`$VAR`/relative/absolute, and the two new tests pin down both the defaulting behaviour and the regression they're fixing. `slices.Compact` (after `slices.Sort`) handles the case where `InitializeAs` already exists in `ContextPaths`. The only subtle behaviour is that `InitializeAs` now influences `ContextPaths` even when the user didn't explicitly include it — but that's the *point* of the fix, and it matches user expectation ("if I told it to initialize at X, X is part of my context").

## Verdict

**merge-as-is** — surgical fix on a real bug, with two well-scoped tests covering both the defaulting and the existence-check paths. The path resolver is properly layered and the load.go reordering is the minimum surgery needed.
