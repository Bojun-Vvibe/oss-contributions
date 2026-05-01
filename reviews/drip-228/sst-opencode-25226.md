# sst/opencode #25226 — Preapprove agent tmp directory access

- **PR**: https://github.com/sst/opencode/pull/25226
- **Head SHA**: `915a9ceb99037996f3a10bfc90b862bb48fa05d4`
- **Files reviewed**:
  - `packages/core/src/global.ts` (+5 -0)
  - `packages/opencode/src/agent/agent.ts` (+5 -1)
  - `packages/opencode/src/tool/bash.ts` (+2 -0)
  - `packages/opencode/src/tool/bash.txt` (+2 -0)
- **Date**: 2026-05-01 (drip-228)

## Context

Today, when an agent needs scratch space outside the workspace (build
artifacts, decompressed downloads, intermediate JSON for piping
between commands), every write triggers the external-directory
permission gate. The user hits the approval prompt for every single
`mktemp`/`/tmp/foo`/`os.tmpdir()` use. This PR carves out one
opencode-owned subdir under the OS temp directory, preapproves it, and
tells the model about it via the bash-tool prompt so the model
actually targets it.

## Diff walk

**`global.ts:14, 25, 37, 50, 62`** — adds `tmp` to the `Path` registry,
the `Interface` shape, and the `make()` factory. Created via
`fs.mkdir(Path.tmp, { recursive: true })` at module init alongside
`data`/`config`/`state`/`log`/`bin`. Path is `path.join(os.tmpdir(),
app)` — namespaced under the app slug so multiple opencode instances
don't collide with unrelated tmp users.

**`agent/agent.ts:84-88`** — extends the per-instance `whitelistedDirs`
list:

```diff
-const whitelistedDirs = [Truncate.GLOB, ...skillDirs.map((dir) => path.join(dir, "*"))]
+const whitelistedDirs = [
+  Truncate.GLOB,
+  path.join(Global.Path.tmp, "*"),
+  ...skillDirs.map((dir) => path.join(dir, "*")),
+]
```

The `*` suffix scopes the allowance to children of the tmp dir, not
the dir itself — matches the existing skillDirs pattern.

**`tool/bash.ts:590`** — surfaces `${tmp}` as a template variable in
the bash-tool prompt alongside `${directory}`/`${os}`/`${shell}`.

**`tool/bash.txt:7-8`** — the new prompt copy:
> Use `${tmp}` for temporary work outside the workspace. This
> directory is pre-approved for external directory access.

## Observations

1. **Whitelist semantics depend on the gate's interpretation of `*`.**
   `path.join(Global.Path.tmp, "*")` becomes a glob like
   `/tmp/opencode/*` — that's only one level deep. If the agent does
   `mkdir -p /tmp/opencode/build/x86_64/objects && touch
   .../foo.o`, whether that file is whitelisted depends on whether the
   permission check uses literal-`*` (single segment, blocks deep
   paths) or glob-`**` semantics. The skillDirs entry uses the same
   shape so behavior should be consistent, but worth a one-line test:
   `assert(allowed("$tmp/build/foo.o"))` to lock in the intended deep
   match.

2. **Tmp not added to `cleanup`.** The other `Path.*` entries that
   accumulate state (`cache`, `log`) are typically reaped on a schedule;
   this PR creates `Path.tmp` at startup but adds no reaper. Over time
   an agent that downloads large artifacts will fill `/tmp/opencode/`
   without bound. Recommend either (a) wipe on instance start (matches
   the "scratch" intent), (b) per-instance subdir under `Path.tmp/<pid>`
   that's removed on graceful shutdown, or (c) document it as the
   user's responsibility. Right now it silently grows.

3. **Prompt copy is precise enough to be useful.** The `${tmp}` token
   is rendered to an absolute path before the model sees it (via the
   `replaceAll` chain at `bash.ts:590`), so the model sees a concrete
   path it can paste into commands. Good — vague "use a temp dir"
   guidance routinely gets ignored.

4. **No test that tmp survives `Permission.fromConfig` evaluation.**
   The whitelist is consulted by the permission layer but the PR
   doesn't add an assertion that a write to `Path.tmp/foo` actually
   bypasses the prompt. One arm in the existing agent permission tests
   would lock the contract.

## Verdict

**merge-after-nits** — the approach is right; ship after deciding the
cleanup story (even a one-line "TODO: reap on startup" comment is
better than silent growth) and adding the deep-glob assertion.

## What I learned

When you preapprove a path for a tool, you also implicitly accept
responsibility for that path's hygiene — disk fill, leftover
credentials, or stale build artifacts become the project's problem,
not the user's. Pairing the preapproval with a startup-wipe (or at
minimum a documented growth policy) is the load-bearing half of the
change.
