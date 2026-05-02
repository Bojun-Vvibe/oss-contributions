# sst/opencode PR #25376 — Convert LoadInput.init to Effect + extract InstanceBootstrap as a Service

- **Head SHA:** `2ad596bcc03cdb99da0afcfbeb13d3645bec94ad`
- **Files:** 21 across `packages/opencode/src/{cli,config,file,lsp,project,server,tool,worktree}` + 3 tests
- **LOC:** +224 / −154

## Observations

- `project/bootstrap.ts:11-71` — converting `InstanceBootstrap` from a free `Effect.gen` into a `Context.Service` and yielding each dependency at *layer init* (`bus`, `config`, `file`, `fileWatcher`, `format`, `lsp`, `plugin`, `shareNext`, `snapshot`, `vcs`) is the correct way to break the `Config → Instance → InstanceStore` circular declaration loop noted in the comment. This is a real, non-cosmetic fix; the previous shape forced consumers to thread `R` everywhere.
- `project/instance-store.ts:13-30` — `LoadInput` gaining a generic `<R = never>` and `init?: Effect.Effect<void, never, R>` (replacing the `() => Promise<unknown>` callback) flows cleanly through `load`/`reload`/`provide`. The signature `provide<A, E, R, R2 = never>` properly composes the two requirement sets at `Effect.Effect<A, E, R | R2>` — exactly what you want for "test setup that needs the ALS context."
- `project/instance-context.ts:1-21` — extracting `containsPath()` into the leaf `instance-context` module is the correct dependency direction (config/file/lsp can now import the helper without pulling `Instance`). The `worktree === "/"` short-circuit is a real bug fix: non-git projects previously matched ANY absolute path and would silently bypass `external_directory` permission. Worth a regression test.
- `lsp/lsp.ts:223-226` — replacing the inline `AppFileSystem.contains(...) || worktree-check` with `containsPath(file, ctx)` is a behavior change: the old code returned `[]` clients when the file escaped both directory and worktree; the new helper returns `false` when worktree is `/`, which matches the new semantics but means non-git projects now get LSP clients for paths *outside* the working dir. Confirm this is desired.
- The `export * as InstanceBootstrap from "./bootstrap"` self-re-export at the bottom is a circular re-export trick to keep the legacy import surface working — it works but is fragile under bundler tree-shaking; a deprecation comment would help.

## Verdict: `merge-after-nits`

- Add a regression test for `containsPath` with `worktree === "/"`.
- Document the LSP behavior change for non-git projects in the commit message.
- Comment the `export * as InstanceBootstrap from "./bootstrap"` self-re-export.
