# openai/codex #21101 — Make managed-network unified exec tests local

- **Head SHA:** `e5369cecce0e2d1a15a2d38492fd47aae2928b28`
- **Author:** @starr-openai
- **Verdict:** `merge-as-is`
- **Files:** +5 / −1 in `codex-rs/core/tests/suite/unified_exec.rs`

## Rationale

Surgical, well-justified one-line behavior change in a single test setup. The diff at `codex-rs/core/tests/suite/unified_exec.rs:944-948` swaps `builder.build_remote_aware(server)` for `builder.build(server)` and lands a four-line TODO comment that explains both *why* the swap is needed (remote exec-server can't currently enforce managed-network proxy denials inside the remote environment) and *what triggers the switch back* (when remote sandboxing gains that capability). That TODO formulation is exactly what's needed for a temporary downgrade — it pins the precise condition under which the change reverts, so a future reader won't have to do code-archaeology to figure out whether the local-only path is still required.

The PR body explicitly notes the test exercises "managed network proxy denial cancels/fails a unified exec process" — that's the assertion at line 949 (`test.config.permissions.network.is_some()`) plus whatever the rest of the test (not in the diff) checks against the proxy. By forcing the local builder, the test now actually exercises the local sandbox+proxy code path that *can* enforce denial today, instead of running through a remote harness that silently passes because it never enforces denial in the first place. That's a strict improvement: a previously-meaningless green pass becomes a meaningful one.

Validation is appropriately minimal for a test-only change: `just fmt` + `git diff --check` is sufficient because the production code path is untouched, and `rust-ci-full` Linux remote CI will exercise the test itself. There's no new dependency, no new public surface, no schema or protocol change, and the blast radius is bounded to one test function.

The only thing I'd note is that the TODO doesn't reference an issue number to track the "switch back" condition. If there's an internal tracker for "wire sandboxing through remote exec-server", linking it would let a future contributor grep for the issue and find both the symptom (this TODO) and the fix site at the same time. But this is genuinely a nit on a 5-line fix, and the comment is descriptive enough that a future contributor working on remote sandboxing will encounter it through normal `rg` patterns. Ship it.
