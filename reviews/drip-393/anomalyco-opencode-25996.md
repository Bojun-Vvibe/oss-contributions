# Review: anomalyco/opencode#25996

- **PR:** [chore(desktop): add @parcel/watcher platform packages to optionalDependencies](https://github.com/anomalyco/opencode/pull/25996)
- **Head SHA:** `493b55c9922f4f6072f8759822f867084614d598`
- **Verdict:** `merge-as-is`

## Summary

Adds the eight `@parcel/watcher-<platform>` prebuilt binary packages to `packages/desktop/package.json` `optionalDependencies` (and the corresponding `bun.lock` rows). This mirrors what `@lydell/node-pty` already does in the same package: declare every platform binary as optional so cross-platform installs don't fail, but the right one is picked up at runtime.

## Specific notes

- `packages/desktop/package.json:66-73` — covers `darwin-arm64`, `darwin-x64`, `linux-arm64-glibc`, `linux-arm64-musl`, `linux-x64-glibc`, `linux-x64-musl`, `win32-arm64`, `win32-x64`. That's the full matrix `@parcel/watcher` ships at 2.5.1 — nothing missing.
- All pinned at `2.5.1` exact. That matches the resolved version that the parent `@parcel/watcher` would dynamically require, so no version-skew risk between the platform shims and the wrapper.
- `bun.lock:271-281` — lockfile updated in lockstep with package.json, no orphan entries.

## Rationale

This is the canonical fix for "binary platform package wasn't installed because npm/bun didn't see it as optional in this workspace." Pattern is already proven in this same package for node-pty. Diff is mechanical, low-risk. Ship.
