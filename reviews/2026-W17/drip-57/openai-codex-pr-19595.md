# openai/codex #19595 — [codex] Bypass managed network for escalated exec

- URL: https://github.com/openai/codex/pull/19595
- Head SHA: `f5ece67867980b4bff3020821c2550b2a986077e`
- Verdict: **needs-discussion**

## What it does

When `sandbox_permissions = "require_escalated"` (an explicit user
approval to run *outside* the filesystem/platform sandbox), this PR
also stops routing the child process through codex-managed network
proxy state. Concretely: skip managed network approval specs in the
runtime, pass `network: None` into shell / zsh-fork shell / unified
exec sandbox preparation, and strip codex-managed proxy env vars when
`CODEX_NETWORK_PROXY_ACTIVE` is set.

## Diff notes

- New helper at `codex-rs/core/src/tools/runtimes/mod.rs:46-67` —
  `exec_env_for_sandbox_permissions` clones the env and, if
  `requires_escalated_permissions()` and the proxy active marker is
  set, removes everything in `PROXY_ENV_KEYS`. macOS additionally
  strips `GIT_SSH_COMMAND` when it starts with the codex marker.
  Correct pattern: only strip env we know we set; leave user proxy env
  alone unless `CODEX_NETWORK_PROXY_ACTIVE` proves we own it.
- Touch points in `shell.rs`, `unix_escalation.rs`, `unified_exec.rs`
  consistently pass `network: None` for the escalated branch and
  consistently call the new env helper. The three sites are kept in
  lockstep, which is exactly what the comment in
  `sandboxing.rs:11` calls out as required.
- Test coverage in `mod_tests.rs` (133 new lines) builds a full
  `NetworkProxy` with a `StaticReloader` and asserts the prepared exec
  request for escalated sandboxing. That regression test is the right
  shape — it's the kind of thing that would silently regress otherwise.

## Why this verdict

The mechanics are clean and well-tested. The discussion point is the
**security surface this opens**, which the PR description names
explicitly:

> If an untrusted command or script is allowed to run with
> `require_escalated`, its network calls are unsandboxed: codex-managed
> network allowlists and denylists are not respected for that process,
> so the command can exfiltrate any data it can read.

That is a real semantic widening. Today some users may have
`require_escalated` configured for specific commands relying on the
managed proxy as a *second* layer of egress control (allowlist) even
when the filesystem sandbox is bypassed. After this PR, "approve to
run outside the sandbox" silently also means "approve all outbound
network." That should be a release-note headline and ideally an
opt-out config knob (e.g. `keep_managed_network_for_escalated = true`),
not just an unconditional behavior change.

Asking for: (1) explicit changelog entry naming the security widening,
(2) consideration of an opt-out for users who relied on the prior
two-layer behavior. Code itself is fine to ship once that's settled.
