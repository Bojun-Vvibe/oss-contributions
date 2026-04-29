---
pr: openai/codex#20240
sha: 0ea76144e4e81125d5d0a54d17ed0e4f52acd50b
verdict: merge-as-is
reviewed_at: 2026-04-29T18:31:00Z
---

# [linux sandbox] Isolate IPC namespace in bubblewrap

## Context

The Linux bubblewrap sandbox (`codex-rs/linux-sandbox/src/bwrap.rs`) was
already passing `--unshare-user` and `--unshare-pid`, but it left the
host IPC namespace shared. That meant a sandboxed command could
`shmget`/`shmat` into a host-side System V shared memory segment owned
by the same uid — an unmediated host communication channel that bypasses
the filesystem and network controls bubblewrap is supposed to provide.
Linear ticket BUGB-16097, Bugcrowd `6e34212b-…`.

## What's good

- The fix is the smallest possible: one extra `"--unshare-ipc".to_string()`
  pushed into both `create_bwrap_flags_full_filesystem` (line 158) and
  `create_bwrap_flags` (line 202). No new conditional, no new option flag,
  no opt-out. That's the right call — there's no legitimate reason a
  sandboxed agent command needs to attach to host SysV IPC, and adding
  a knob would just create a footgun.
- The regression test in `codex-rs/linux-sandbox/tests/suite/landlock.rs`
  is doing the right thing: `HostSysvSharedMemory::new(secret)` allocates
  a real `IPC_PRIVATE` segment in the test process, then runs a
  re-exec'd helper (`sysv_ipc_probe_helper`) inside the sandbox that
  attempts `libc::shmat(shmid, …)`. The helper `panic!`s if the
  attach succeeds and reads the secret back — so the test fails loudly
  if the namespace isolation regresses. Drop impl on `HostSysvSharedMemory`
  uses `IPC_RMID` to clean up even on test failure. Solid.
- The `bwrap_preserves_writable_dev_shm_bind_mount` test (already in
  the suite, line 291) covers the inverse concern: that `--unshare-ipc`
  doesn't break `/dev/shm` workflows that depend on POSIX shm bind-mounts.
  The author kept that test green, which is the only non-obvious risk
  of this change.
- The `linux_run_main_tests.rs` argv-ordering test was updated in lockstep
  (line 79) so the argv-builder unit test continues to assert exact flag
  order. No drift.

## Concerns / nits

- README update is one sentence (`--unshare-ipc`); could mention that
  POSIX shm via `/dev/shm` bind-mount is unaffected, since that's the
  most likely "is this going to break my workflow?" question.

## Verdict

`merge-as-is` — security fix, smallest possible diff, regression test
proves the boundary is closed and the `/dev/shm` carve-out still works.
This should land.
