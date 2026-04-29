# Review: sst/opencode#24921 — chore: Refactor dependency installation for multiple targets

- **PR:** https://github.com/sst/opencode/pull/24921
- **Head SHA:** `b93ceb3fba0cda53a5629f8137100a1deab95e28`
- **Author:** ingbyr
- **Diff size:** +5 / -2 (single file: `packages/opencode/script/build.ts`)
- **Verdict:** `merge-after-nits`

## What it does

Closes #23626. Replaces a single pair of "wildcard install everything" calls in the build script with a loop over `allTargets` that installs `@opentui/core` and `@parcel/watcher` once per `(os, arch)` target, with a per-iteration `console.log("Installing ${os} - ${arch} dependencies...")` for visibility.

Before (`build.ts:166-170`):
```ts
if (!skipInstall) {
  await $`bun install --os="*" --cpu="*" @opentui/core@${pkg.dependencies["@opentui/core"]}`
  await $`bun install --os="*" --cpu="*" @parcel/watcher@${pkg.dependencies["@parcel/watcher"]}`
}
```

After:
```ts
if (!skipInstall) {
  for (const { os, arch } of allTargets) {
    console.log(`Installing ${os} - ${arch} dependencies...`)
    await $`bun install --os=${os} --cpu=${arch} @opentui/core@${pkg.dependencies["@opentui/core"]}`
    await $`bun install --os=${os} --cpu=${arch} @parcel/watcher@${pkg.dependencies["@parcel/watcher"]}`
  }
}
```

## Specific citations

- `packages/opencode/script/build.ts:166-176` — the loop body. Note the loop variable is `allTargets`, not `targets` (the latter is filtered by `BUILD_TARGETS` env later in the file at `:172`); this is intentional — install must cover the union, then the per-target build loop picks from `targets`. Worth confirming `allTargets` is in scope at this line in the merged file (it appears to be defined earlier in the same module).
- The PR description includes "How did you verify your code works?" left blank — no local build log attached.

## Nits to address before merge

1. **Show the version being installed in the log line.** `Installing ${os}-${arch} (@opentui/core@${ver}, @parcel/watcher@${ver})...` makes CI logs grep-friendly when an upstream bump misbehaves on one platform only.
2. **Confirm `--os=*  --cpu=*` was actually broken or wasteful.** The PR's stated motivation ("install only target-platform-related dependencies") implies the wildcard form was either failing or pulling in N×M packages anyway. A one-line note in the PR body — "wildcard form caused X / installed N MB of unused binaries" — would justify the trade-off (per-target install is now `len(allTargets)` sequential network calls vs 1 wildcard call).
3. **Sequential `await` inside the loop is the right default** (avoids hammering the registry and keeps logs readable), but if `allTargets.length` is 6+ and CI build time matters, a follow-up could `Promise.all` the inner pair per target. Don't block on this.
4. **No regression risk surface in the diff itself,** but the "what" is "we now run this loop on every CI build" — please confirm at least one CI job has the new log lines on a fresh run before squash-merge, otherwise a typo in `allTargets` shape (e.g. `arch` vs `cpu` field name) will only show up post-merge.
5. **Consider gating verbose `console.log` behind a `--verbose` flag or `process.env.CI`,** since this script may also run interactively for local dev builds and 6+ "Installing..." lines per build is mildly noisy.

## Rationale

7-line refactor of a build helper, no production code path touched, and the shape is correct: the previous wildcard install was either wasteful (downloads every os/arch native binary) or actually-broken-and-relying-on-a-bun-install-quirk. The new loop is explicit and visible. Concerns are cosmetic (log content) and discoverability (no test or before/after metric). Approve after the version-in-log nit and a one-line PR body justification.