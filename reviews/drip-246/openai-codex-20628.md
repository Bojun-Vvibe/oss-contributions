# PR #20628 — fix(linux-sandbox): fall back when system bwrap lacks perms

- Repo: openai/codex
- Author: viyatb-oai
- Head: `a790eda9c5717726bfcf177747531dce722acf21`
- URL: https://github.com/openai/codex/pull/20628
- Verdict: **merge-as-is**

## What lands

Single-file +48/-8 in `codex-rs/linux-sandbox/src/launcher.rs`. Extends the
existing `--argv0` capability probe to also check for `--perms` support, and
falls back to the vendored bwrap when either is missing. New struct
`SystemBwrapCapabilities { supports_argv0, supports_perms }` replaces the
prior bool-returning `system_bwrap_supports_argv0()`.

## Specific findings

- `launcher.rs:26-30` introduces `SystemBwrapCapabilities` as a `Copy +
  PartialEq + Eq` struct with two named fields. Right shape — captures
  exactly the capabilities the launcher needs to decide system-vs-vendored.
- The match-on-Option pattern at `:64-69`:
  ```rust
  let Some(SystemBwrapCapabilities {
      supports_argv0,
      supports_perms: true,
  }) = system_bwrap_capabilities(system_bwrap_path)
  else {
      return BubblewrapLauncher::Vendored;
  };
  ```
  is the correct way to express "must succeed *and* must have perms".
  `supports_perms: true` in the destructuring pattern is the gate; any
  `Some(.. supports_perms: false)` or `None` falls through to vendored.
  Concise and unambiguous.
- `system_bwrap_capabilities()` at `:98-105` probes `--help` once and
  greps both stdout and stderr for both `--argv0` and `--perms`. Single
  process spawn, two flag checks — efficient. `Err(_) => return None`
  at `:97` correctly distinguishes "couldn't even invoke bwrap" from
  "bwrap doesn't have the flag" by collapsing both to the vendored
  fallback (the right behavior).
- Three test cases at `:179-228`:
  - `:179-194` system bwrap with both flags supported → `System` launcher.
  - `:198-211` system bwrap with `argv0=false, perms=true` → `System`
    launcher with `supports_argv0: false` (preserves existing behavior
    where argv0 alone is optional).
  - `:213-228` (NEW) system bwrap with `perms=false` → `Vendored`. This
    is the regression test for the actual bug the PR closes.
  Coverage of the full {has-both, has-only-argv0, missing-perms,
  missing-binary} matrix is correct.
- The justification comment at `:91-94` correctly notes that older distro
  packages (Ubuntu 20.04/22.04) ship without `--argv0`. The PR extends the
  same logic to `--perms` without rewriting the comment to call out *which*
  distros lack `--perms` — minor doc gap, not blocking.

## Risks

- `--perms` was added in bwrap 0.5.0, much earlier than `--argv0` (0.9.0),
  so any system old enough to lack `--perms` is also old enough to lack
  `--argv0` — which means in practice the new `supports_perms: false`
  branch should always coincide with `supports_argv0: false`. The test at
  `:213-228` passes `supports_argv0: false, supports_perms: false` which
  matches that real-world correlation. Fine.

## Verdict

**merge-as-is**. Capability probe extension is mechanically correct, test
matrix is complete, the destructuring-pattern gate at `:65-69` is the
clearest way to express the policy. Ship it.
