# PR #19852 — Enforce preserved path names in Linux sandbox

- **Repo**: openai/codex
- **PR**: #19852
- **Head SHA**: `87b8a740`
- **Author**: evawong-oai
- **Size**: +1296 / -86 across 3 files
- **URL**: https://github.com/openai/codex/pull/19852
- **Verdict**: **merge-after-nits**

## Summary

This PR teaches the Linux `bwrap` sandbox to enforce the
`is_preserved_path_name` contract introduced earlier in the
preserved-path stack. Previously a `FileSystemSandboxPolicy` could
declare protected paths but the bubblewrap argv builder didn't
materialize them as actual mount targets, which meant a process
inside the sandbox could (intentionally or accidentally) create or
write through paths that the host considered preserved. The fix
introduces two new mount-target primitives — `SyntheticMountTarget`
(empty file or empty directory bind-mounted RO over the real
preserved location) and `ProtectedCreateTarget` (path that must not
exist post-exec and gets a tombstone bind, preventing creation) —
plumbs them through `BwrapArgs`, and has `linux_run_main` materialize
them before launching bwrap with deterministic cleanup on exit using
`FileIdentity { dev, ino }` to avoid clobbering a real pre-existing
file when the inode doesn't match.

## Specific changes

- `codex-rs/linux-sandbox/src/bwrap.rs:104-225` — adds the
  `FileIdentity { dev: u64, ino: u64 }` newtype with
  `from_metadata(&Metadata)`, plus `SyntheticMountTargetKind`
  (`EmptyFile` | `EmptyDirectory`), `SyntheticMountTarget` with
  three constructors (`missing`, `missing_empty_directory`,
  `existing_empty_file(path, &Metadata)` — the last captures the
  pre-existing inode), and `ProtectedCreateTarget::missing`.
  The `pre_existing_path: Option<FileIdentity>` field is the
  correct hedge against the race where the host file/dir is
  recreated between scan and cleanup with a different inode.
- `codex-rs/linux-sandbox/src/bwrap.rs:401-1132` — `BwrapArgs` gains
  `synthetic_mount_targets` and `protected_create_targets`. The
  filesystem-arg construction at `:474-635` now scans the policy's
  preserved-path set, classifies each entry by existence/kind,
  appends the appropriate `--ro-bind` / `--bind` argv pair, and
  records the cleanup target. The `is_preserved_path_name` check
  from `codex_protocol::permissions` is invoked at every site where
  a writable root or filesystem entry is materialized (`:401`,
  `:474`, `:541`, `:579`).
- `codex-rs/linux-sandbox/src/bwrap.rs:1132-1238` — new
  `is_within_allowed_write_paths(path, &[PathBuf]) -> bool` helper
  used to gate whether a preserved path is *allowed* to be written
  (i.e. the policy explicitly carved it out) vs blanket-protected.
  Correctly uses path-component prefix match, not string prefix,
  so `/foo/bar2` does not match `/foo/bar`.
- `codex-rs/linux-sandbox/src/linux_run_main.rs:599-1153` — the
  outer driver now materializes `synthetic_mount_targets` (creating
  empty files/dirs at host paths via `tempfile::TempDir` siblings)
  and `protected_create_targets` (placing tombstones), then runs
  bwrap, then deterministically cleans up post-exit by checking
  `FileIdentity::from_metadata(...)` against the captured
  `pre_existing_path`. If the inode no longer matches, cleanup
  skips the unlink (correct — that means the target was replaced
  during exec by something the user actually owns).
- `codex-rs/linux-sandbox/src/bwrap.rs:1600-1845` — extensive new
  test coverage: the `mod tests` block adds end-to-end argv
  assertions for missing-file vs existing-empty-file vs
  missing-directory cases, plus the inode-races-detected case
  pinning that cleanup leaves the host file alone when the inode
  diverges. ~270 lines of test, with `pretty_assertions` used
  throughout.

## Risks

The reviewer-focus admission that "this is UX, not security" from
the broader stack still applies — the *real* enforcement still
lives in the kernel's bind-mount semantics. The fix is correct in
that the `--ro-bind` over a tombstone makes the protected path
unwritable from inside the sandbox, but a process that escapes
bwrap (which is its own threat model) is unaffected.

Two diff-level nits:

1. The `FileIdentity` struct does not include `mtime` or `ctime`,
   so a TOCTOU window exists where the host file is removed and
   re-created between the pre-scan capture and the cleanup pass
   *with a recycled inode*. On modern Linux ext4/xfs this is rare
   but not impossible; `tempfile`'s typical churn can recycle inodes
   within seconds. Either adding `mtime` to the identity tuple or
   documenting the race in the `FileIdentity` doc comment would
   close the gap.

2. The `linux_run_main.rs` cleanup uses `eprintln!` (or equivalent)
   to log skip decisions but doesn't surface them as a structured
   warning that the outer event stream sees. A user whose preserved
   path was replaced mid-exec deserves to know cleanup skipped it
   rather than silently leaving the artifact on disk.

## Verdict rationale

Architecturally correct: capability-narrowing change pushed down to
the layer that actually constructs the bwrap argv, with the
`FileIdentity` race-close on cleanup being exactly the right shape.
Test coverage is substantial and pins the right invariants
(missing/existing kinds, inode-race skip). Merge after the
`FileIdentity` mtime nit is addressed (or the race documented) and
the cleanup-skip logging is upgraded to a structured warning.
