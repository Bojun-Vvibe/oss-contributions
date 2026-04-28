# openai/codex #20001 — fix(network-proxy): harden linux proxy bridge helpers

- PR: https://github.com/openai/codex/pull/20001
- Head SHA: `c0562f53fb7a748de9eec0d212333415e367cd4a`
- Author: viyatb-oai
- Files touched: `codex-rs/linux-sandbox/src/landlock.rs`, `codex-rs/linux-sandbox/src/proxy_routing.rs`

## Observations

- `codex-rs/linux-sandbox/src/landlock.rs:178-179` — adds `deny_syscall(&mut rules, libc::SYS_process_vm_readv);` and `deny_syscall(&mut rules, libc::SYS_process_vm_writev);` next to the existing `SYS_ptrace` deny. Closes the cross-process memory peek/poke escape that `ptrace`-only blocking leaves open: a sandboxed task with `CAP_SYS_PTRACE` waived can still read/write another process's address space via `process_vm_readv/writev` if those syscalls are unrestricted. Correct fix.
- `codex-rs/linux-sandbox/src/proxy_routing.rs:453` and `:504` — replaces the single `set_parent_death_signal()?;` call at the start of both `run_host_bridge` and `run_local_bridge` with `harden_bridge_process()?;`.
- `proxy_routing.rs:617-630` — new helper `harden_bridge_process()` chains `set_parent_death_signal()` with `disable_process_dumping()`, which calls `prctl(PR_SET_DUMPABLE, 0, ...)`. `PR_SET_DUMPABLE=0` makes the bridge process non-attachable by `ptrace` from the same UID and prevents core-dump-based credential leaks. Pairs well with the seccomp tightening above.
- Error handling in `disable_process_dumping` returns `io::Error::last_os_error()` on non-zero `prctl` return — standard idiom.
- No new tests for the seccomp deny list; harder to assert in unit form, but a small integration test that spawns a sandboxed process and confirms `process_vm_readv` returns `EPERM` would document the contract.

## Verdict: `merge-as-is`

**Rationale:** Defense-in-depth hardening that closes two well-known sandbox-escape primitives (`process_vm_readv/writev`, ptrace-on-same-uid via dumpable). Small, focused, and matches conventional Linux-sandbox hardening patterns. Test coverage is a fair ask but shouldn't gate landing.
