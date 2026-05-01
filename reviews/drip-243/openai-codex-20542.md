# Review: openai/codex #20542 — Prune unused code-mode globals

- **PR**: https://github.com/openai/codex/pull/20542
- **Author**: cconger (Channing Conger)
- **Head SHA**: `9739efc3de0d1aa43d20fe8aca42f87e199ca5f4`
- **Base**: `main`
- **Files**: `codex-rs/code-mode/src/runtime/globals.rs` (+18/-5),
  `codex-rs/core/tests/suite/code_mode.rs` (+0/-3)
- **Verdict**: **merge-as-is**

## Reasoning

Hardening cleanup that hides three V8 globals — `Atomics`,
`SharedArrayBuffer`, `WebAssembly` — from the code-mode JS runtime
surface, on the rationale that the harness exposes no worker support
and exposes no use case for these APIs. The diff also factors out
the existing `console`-deletion pattern into a reusable `delete_global`
helper.

What the diff actually does:

- `globals.rs:14-19`: replaces the inline `console`-deletion block
  with four calls to a new `delete_global(scope, global, name)` helper:
  `console`, `Atomics`, `SharedArrayBuffer`, `WebAssembly`.
- `globals.rs:144-156`: the new `delete_global` helper mirrors the
  existing `set_global` shape — allocates a V8 string for the key,
  calls `global.delete(scope, key.into())`, and returns
  `Err(format!("failed to remove global \`{name}\`"))` on
  `Some(false) | None`. This matches the prior inline behavior exactly,
  so the refactor is behavior-preserving for `console`.
- `core/tests/suite/code_mode.rs:2371-2419`: removes three entries
  (`Atomics`, `SharedArrayBuffer`, `WebAssembly`) from the
  `Object.getOwnPropertyNames(globalThis).sort()` golden assertion
  to match the new pruned surface. The test is the load-bearing
  check that the deletion actually took effect — if any of the
  three globals fail to delete, the golden test will fail.

Why this is solid:

- **Right shape for the refactor.** Three call sites would have
  led to drift between deletion attempts and error messages; the
  helper makes adding a fourth or fifth global a one-liner.
- **`SharedArrayBuffer` and `Atomics` removal is correct.** Those
  two only have meaningful semantics when paired with a worker
  pool or a `MessageChannel`, neither of which exist in the
  code-mode harness. They're not dangerous in isolation, but they
  are nonzero attack surface (timing-side-channel literature) and
  zero useful surface here, so deletion is pure win.
- **`WebAssembly` removal is the most consequential of the three**
  and the right call. Even without workers, `WebAssembly.compile`
  is a JIT path that increases the V8 attack surface and is wholly
  unused by harness scripts.
- The error-message shape (`failed to remove global \`{name}\``)
  makes a future deletion regression diagnosable from logs.

## Suggested follow-ups

- Consider also pruning `eval` and `Function` (the constructor) at
  the same layer — both are dynamic-code-evaluation primitives that
  the harness presumably doesn't want either. Out of scope for this
  PR but a natural next step in the same hardening direction.
- Add a one-line module doc comment on `globals.rs` listing the
  deletion-vs-allowlist policy so the next contributor doesn't
  re-add `WebAssembly` thinking it was an oversight.
- Optional: invert the model to an explicit allowlist of permitted
  globals rather than a denylist of pruned ones. Larger refactor,
  but it's the long-run correct shape for sandboxed runtimes.
