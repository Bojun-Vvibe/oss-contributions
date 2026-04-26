# block/goose#8829 — chore(deps): bump winreg from 0.55.0 to 0.56.0

- **Repo**: block/goose
- **PR**: [#8829](https://github.com/block/goose/pull/8829)
- **Head SHA**: `91782fa60eed`
- **Author**: app/dependabot
- **Base**: `main`
- **Size**: +5 / −5, 2 files

## Context

Single-purpose dependabot bump for the Windows-only `winreg` crate consumed
by `goose-server`. `winreg` is a thin wrapper over the Win32 registry API;
0.55 → 0.56 is a minor bump that, per the upstream changelog, primarily
re-targets the `windows-sys` underpinning.

## Change

`crates/goose-server/Cargo.toml:83` is the only manifest edit:

```toml
[target.'cfg(windows)'.dependencies]
-winreg = { version = "0.55.0" }
+winreg = { version = "0.56.0" }
```

`Cargo.lock` reflects two cascading edits:

1. `winreg` block at line ~12282 bumps version + checksum and bumps its
   transitive `windows-sys` dep from `0.59.0` → `0.61.2`.
2. A separately-flagged unrelated `itertools` bump at line ~7384
   (`0.13.0` → `0.14.0`) inside an unnamed dep block — appears to be a
   transitive resolution shift triggered by `cargo update -p winreg`
   pulling a fresher minor of a shared transitive. **Worth confirming**
   the lockfile delta is exactly what `cargo update -p winreg` produces;
   if the `itertools` bump is unrelated, it should be in a separate PR
   per dependabot's usual hygiene.

## Strengths

- Manifest change is minimal and platform-gated (`cfg(windows)`), so the
  blast radius on macOS/Linux CI is zero.
- Integrity hashes in the lockfile (`7d6f32a0ff...`) are the published
  registry values for `winreg` 0.56.0; verified against
  `https://crates.io/crates/winreg/0.56.0`.
- The `windows-sys 0.61.2` jump is the *intended* effect of this bump —
  upstream `winreg` 0.56 explicitly migrated to a newer `windows-sys`
  generation. Other crates in the lockfile that still pin
  `windows-sys 0.59.0` will coexist (cargo allows multiple semver-compatible
  copies of `windows-sys` to ship).

## Risks / nits

1. **`itertools 0.13.0 → 0.14.0` rider** — the diff at `Cargo.lock:7384`
   bumps `itertools` inside what looks like a `prost-build` (or similar
   codegen) dep block. That's not in the PR title and not in the
   dependabot description. If this was the side effect of
   `cargo update -p winreg --aggressive`, fine; if it crept in via a
   separate `cargo update`, it should be split out. **Ask the author**
   to confirm the regen command.
2. Windows-only path is the least-tested CI lane in goose. The bump itself
   is low-risk, but a one-line "verified `cargo build --target
   x86_64-pc-windows-msvc -p goose-server` locally" comment from the
   reviewer would be useful, since dependabot can't.
3. No code-side changes needed — `winreg` 0.56 kept the public API
   compatible with 0.55. (`HKLM`, `RegKey`, `OpenSubkey` all unchanged.)

## Verdict

**merge-after-nits** — fine to merge once the `itertools` rider is
explained or split. Manifest + Cargo.lock primary edits are coherent.

## What I learned

Windows-only deps on `winreg` always end up dragging a `windows-sys`
generation along. The right hygiene for those bumps is to pre-flight
`cargo tree -i windows-sys` before merging so you know how many copies
of `windows-sys` you're shipping in your binary — having two generations
coexist is fine (cargo dedups within a major), but having three is a
binary-size smell worth flagging.
