# sst/opencode #24930 — fix(nix): remove stale packages/shared filter

- **PR:** https://github.com/sst/opencode/pull/24930
- **Head SHA:** dd49d374f196ce1f1b1f642591d8308ac8736f23
- **Files changed:** 1 file, +0 / −1 (`nix/node_modules.nix`)
- **Verdict:** `merge-as-is`

## What it does

Drops one line from the `pnpm install` filter list in the Nix derivation
(`nix/node_modules.nix:58`):

```diff
       --filter './packages/opencode' \
       --filter './packages/desktop' \
       --filter './packages/app' \
-      --filter './packages/shared' \
       --frozen-lockfile \
```

`packages/shared/` no longer exists in the workspace, so passing
`--filter './packages/shared'` causes pnpm to error out with "no projects matched the
filters" (or, depending on pnpm version, silently treat the filter as a no-match and
warn). Either way, the Nix build of `node_modules` is currently broken on a clean
checkout, and removing the dead filter restores it.

## What's good

- Single-line surgical fix scoped to the actual bug.
- Cannot regress anything: the package being filtered does not exist, so removing the
  filter changes the install set from "error / warning" to "the three real packages
  that are still listed".

## Nits

None blocking. Optional: a one-line PR-body sentence pointing at the commit that
removed `packages/shared/` would help future archaeology, but the diff speaks for
itself.

## Verification suggestion

Run `nix build .#node_modules` (or whatever the derivation attribute is named in this
repo) on a fresh clone before and after; the "before" should fail at the pnpm step,
the "after" should produce a `result/` symlink.

## What I learned

Workspace filters in Nix derivations are silent landmines after package renames /
deletions — the build switches from "everything is selected" to "filter fails" in
exactly one pnpm-version-dependent way, and CI only catches it if the Nix path is
actually exercised on PRs.
