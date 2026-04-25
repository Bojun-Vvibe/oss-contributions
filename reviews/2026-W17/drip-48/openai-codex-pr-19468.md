# openai/codex#19468 — Fix Bazel cargo_bin runfiles paths

- **URL**: https://github.com/openai/codex/pull/19468
- **Author**: fjord-oai
- **Head SHA**: `8fc5b239860e661fcec25d985441e901c6fb40a6`
- **Verdict**: `merge-as-is`

## Summary

`resolve_bin_from_env` in `codex-rs/utils/cargo-bin/src/lib.rs:91` was
returning whatever path `runfiles::rlocation!` produced, then checking
`.exists()` against it directly. Under Bazel runfiles, `rlocation!` can
return a path that is relative to the runfiles root (e.g. when called
from a different working directory than where the runfiles tree is
materialised), and the bare `.exists()` probe then fails even though
the binary is present. The fix joins a relative `resolved` to
`std::env::current_dir()?` before the existence check.

## Reviewable points

- The structural change replaces the `let-chain`
  `if let Some(resolved) = ... && resolved.exists()` with a nested
  `if let Some(mut resolved) = ...` followed by an explicit
  `if !resolved.is_absolute()` branch. This is the right shape: the
  original `&&` short-circuit meant a relative-but-existing-when-joined
  result was silently dropped.
- `current_dir()` errors are surfaced via the existing
  `CargoBinError::CurrentDir { source }` variant — no new error type
  needed, and the call site is consistent with the `else if raw.is_absolute()`
  branch below it.
- One small observation: when `resolved.is_absolute()` is already true
  but `.exists()` is false, the function falls through to the rest of
  the resolution logic. That matches the prior behaviour, so no
  regression — just worth noting that this PR doesn't change the
  fallback chain, only the relative-path probe.

## Rationale

Surgical, two-branch fix to a real Bazel resolution bug; the new
control flow cleanly preserves the existing fallthrough semantics and
reuses an existing error variant. No tests were added but the bug is
environment-dependent (requires a Bazel runfiles tree where the runfile
is reported relative), which is hard to reproduce in unit tests.
Mergeable as-is.
