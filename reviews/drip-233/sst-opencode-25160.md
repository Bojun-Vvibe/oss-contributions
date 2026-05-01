# sst/opencode#25160 — feat: FFF for file search

- **Author:** Dmitriy Kovalenko (dmtrKovalenko)
- **Head SHA:** `f5955c1e09469303e7527da660e73abc2a974e94`
- **Base:** `dev`
- **Size:** +1252 / -452 across 33 files
- **Files changed:** `bun.lock`, `package.json`, `packages/app/src/components/dialog-select-directory.tsx`, `packages/app/src/components/prompt-input.tsx`, `packages/app/src/context/file.tsx`, `packages/core/src/util/log.ts`, `packages/opencode/bench-fff.ts` (+ many more in `packages/opencode/src/file/`)

## Summary

Replaces opencode's existing file-search backend with `@ff-labs/fff-bun@0.6.4` — a Bun-native FFF (file/fuzzy/find) implementation distributed as platform-pinned native binaries (`@ff-labs/fff-bin-{darwin,linux,win32}-{arm64,x64}[-musl]@0.6.4`). Result shape changes from `string[]` to `{path, ...}[]`, callers updated at `dialog-select-directory.tsx:203` and `context/file.tsx:199`. Adds bench harness at `bench-fff.ts` measuring picker create / scan / file search / grep latencies.

## Specific code references

- `package.json:101`: new dep `"@ff-labs/fff-bun": "0.6.4"` and corresponding `bun.lock` hoist at `:9` plus 8 platform-pinned optionalDependencies entries at `bun.lock:1198-1213` (4 OS × 2 arches, with linux split into gnu/musl). Coverage is full and matches opencode's existing platform matrix.
- `bun.lock:2772`: explicit `bun@1.3.13` dependency added at top level — load-bearing because `fff-bun`'s `peerDependencies` declares `"bun": ">=1.0.0"` and the consumer install needs to resolve it consistently across CI.
- `prompt-input.tsx:594-597`: result-shape migration from `files.searchFilesAndDirectories(query)` to `files.searchFiles(seek)` with new `pathy = /[./\\]/.test(query)` heuristic gating fuzzy-vs-literal search. The `seek = query.replaceAll("\\", "/")` normalization at `:595` handles Windows path inputs uniformly. The `if (pathy) return fileOptions` early-return at `:600` skips the agent/pinned merge when user is clearly typing a path — correct UX choice.
- `prompt-input.tsx:603-604`: new `stale: false, fuzzy: (query) => !/[./\\]/.test(query)` options on the at-completion combobox. Same `pathy` predicate as the search-time heuristic, so display behavior and search-mode choice stay coupled to one source-of-truth regex (though duplicated; see nit).
- `dialog-select-directory.tsx:203`: result mapping flipped from `results.map((rel) => joinPath(scopedInput.directory, rel))` to `results.map((item) => joinPath(scopedInput.directory, item.path))` — correct destructure of the new `{path}` result shape.
- `context/file.tsx:199`: symmetric change at the SDK-client boundary `(x.data ?? []).map(path.normalize)` → `(x.data ?? []).map((item) => path.normalize(item.path))`. Both consumer sites are updated, no stragglers in the diff window.
- `core/src/util/log.ts:21-24`: new `currentLevel()` exporter — small surface addition presumably so the new FFF subsystem can gate verbose tracing on the global level. Pure read of the module-level `let level: Level`, no mutation; safe.
- `bench-fff.ts:1-40` (new file): standalone bench harness with `Instance.provide({ directory, fn })` setup and four `performance.now()` checkpoints (picker create / wait-for-scan / file search "fff" / file search "package.json"). Commits should keep this file in `packages/opencode/` rather than shipping it in the published bundle — verify it's excluded from `prepare-package.js` whitelist before merge.

## Reasoning

Substantial backend swap with broad platform implications. The dep matrix is complete (8 native binary variants), the result-shape migration is consistently applied at both consumer sites, and the search-mode heuristic at `prompt-input.tsx:594` thoughtfully separates "user typing a path" (`/[./\\]/.test`) from "user fuzzy-searching by basename" so the two cases get the right backend behavior.

Concerns worth a maintainer pass before merge:
- **Native binary bundling at distribution time** — adding `@ff-labs/fff-bin-*` to `optionalDependencies` is correct for npm install-time resolution, but the opencode standalone binary build (which freezes a Bun snapshot) needs to know to include the right platform's native module. The `bench-fff.ts` working locally doesn't validate this for shipped builds.
- **No end-to-end CI test added for the new at-completion behavior** — the `pathy` heuristic at `prompt-input.tsx:594` and `:604` is duplicated; if one drifts a user-visible regression follows. A unit test pinning `pathy` for at least four representative inputs (`"foo"`, `"foo/bar"`, `"./foo"`, `"foo.ts"`) at the search-time site would prevent that.
- **Bench harness in `packages/opencode/bench-fff.ts`** — should either move to a non-published location (`/scripts/`, `/bench/`) or be explicitly excluded from publish to avoid shipping it to end users.
- **`@ff-labs/fff-bun` is a young package (0.6.x)** — pinning to exact version `0.6.4` is the right caution, but a one-line PR-body note on the upstream's stability story (single maintainer? release cadence? license?) would help the reviewer.

The result-shape migration discipline is the reason this is `merge-after-nits` rather than `needs-discussion`: every call site identified in the diff is updated symmetrically, and the at-completion UX heuristic is genuinely better than the prior "always fuzzy" behavior.

## Verdict

**merge-after-nits**
