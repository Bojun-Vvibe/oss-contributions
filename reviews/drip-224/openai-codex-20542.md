# openai/codex #20542 — Prune unused code-mode globals

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20542
- **HEAD SHA:** `9739efc3de0d1aa43d20fe8aca42f87e199ca5f4`
- **Author:** cconger
- **Verdict:** `merge-as-is`

## What the diff does

Two-file V8-globals attack-surface reduction in the code-mode runtime:

1. `codex-rs/code-mode/src/runtime/globals.rs:14-18` — replaces the inline
   per-key `v8::String::new` + `global.delete` block (which currently only
   handles `console`) with three additional `delete_global(scope, global,
   "Atomics" | "SharedArrayBuffer" | "WebAssembly")?` calls, factored
   through a new `delete_global` helper at `:145-157` that mirrors the
   shape of the existing `set_global` at `:142`.

2. `codex-rs/core/tests/suite/code_mode.rs:2374/2408/2422` — drops
   `"Atomics"`, `"SharedArrayBuffer"`, and `"WebAssembly"` from the
   sorted-`globalThis` snapshot test that pins the exact set of globals
   the runtime exposes.

## Why the change is right

The factor-out is mechanical and clearly the right shape: the prior code
had `console`-deletion inlined as 4 lines, but the same "allocate v8
string + global.delete + check return" sequence is pure boilerplate. The
new `delete_global` helper at `:145-157` is byte-identical in structure to
the existing `set_global` at `:142` and uses the same
`Some(true)`-on-success convention, so it composes cleanly into the
"delete-then-set" prelude.

The motivation in the PR body — "the harness does not expose worker support
or need those APIs" — is the right justification:

- `Atomics` and `SharedArrayBuffer` are the worker-thread shared-memory
  primitives. Without `postMessage` / worker spawn (which V8 doesn't expose
  by default and which the code-mode runtime has no shim for), they are
  just an attack-surface increase: SharedArrayBuffer in particular has
  historically been the root of Spectre-class side-channel disclosures and
  CPU-time-burn DoS via spin-locks on `Atomics.wait`.
- `WebAssembly` opens a JIT-compiled bytecode execution path that is
  irrelevant to a sandbox whose entire job is to mediate text/JSON
  agent-tool calls. Removing it forecloses an entire class of
  `WebAssembly.compile`-based bypass attempts against the JS-level
  surface restrictions.

The test update at `code_mode.rs:2374/2408/2422` is the contract-pin that
makes this irreversible: the sorted-globals snapshot is the spec form of
"these are the names the runtime exposes," and removing the three names
forces any future reintroduction to surface as a snapshot diff in code
review rather than landing silently.

## Verdict rationale

Surgical defense-in-depth at the V8 sandbox boundary, factored cleanly
through a helper that matches the existing `set_global` shape, with the
removed names pinned by the globals-snapshot test so a future regression
can't slip in unnoticed. No moving parts, no behavioral change for the
documented harness, and the security argument for each of the three names
is straightforward.

`merge-as-is`
