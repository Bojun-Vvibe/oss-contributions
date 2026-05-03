# sst/opencode PR #25560 — fix(packages): convert custom-elements.d.ts symlinks to triple-slash references

- Head SHA: `ae4c01df5430` (full from view)
- Closes: #25564
- Files changed:
  - `packages/app/src/custom-elements.d.ts` (mode 120000 → 100644)
  - `packages/enterprise/src/custom-elements.d.ts` (mode 120000 → 100644)

## Analysis

Symlinks with git mode `120000` pointing to `../../ui/src/custom-elements.d.ts` break on Windows clones without Developer Mode / `core.symlinks=true`. PR replaces both with real files containing a TypeScript triple-slash directive plus an empty re-export:

```
+/// <reference path="../../ui/src/custom-elements.d.ts" />
+export {}
```
(see d1 `packages/app/src/custom-elements.d.ts:1-2` and `packages/enterprise/src/custom-elements.d.ts:1-2`)

Mechanically sound:
- `/// <reference path=...>` injects the referenced declaration file into the compilation context exactly as a symlinked-through `.d.ts` would have, so any custom element typings remain visible to TS consumers.
- `export {}` forces the file to be treated as a module rather than a script, which prevents the ambient declarations from leaking into the global scope inadvertently when imported. Without it, the bare reference file is a script and could change global typing semantics versus the prior symlink behavior. Good defensive choice.
- Hash `bd6fdcad3b4a` is identical for both new files, confirming they are byte-identical, which is what we want for two parallel package shims.

Risks / nits:
- TS `tsconfig.json` for `packages/app` and `packages/enterprise` should already include these files via `src/**/*` globs; otherwise the reference is dead. Not visible in this diff but worth a one-line sanity check by the reviewer.
- The relative path `../../ui/src/custom-elements.d.ts` is identical to the old symlink target, so there is no path-resolution behavior change.
- No CI matrix change is needed; this is purely a portability fix.

## Verdict

`merge-as-is`

## Nits

- Optionally drop a one-line comment in each new `.d.ts` explaining "real file (not symlink) so Windows clones without Developer Mode work" so this does not get reverted next time someone tries to "DRY it up" with a symlink.
