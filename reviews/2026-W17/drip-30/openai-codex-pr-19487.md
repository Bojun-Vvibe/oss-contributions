# openai/codex#19487 — [codex] Move config loading into codex-config

- PR: https://github.com/openai/codex/pull/19487
- Author: pakrym-oai
- +484 / -502
- Base SHA: `8a559e7938bd841d78dc2c442deaa76424faf1d9`
- Head SHA: `072823f35c4b86d9bf78770d64126707bd50c8fb`
- State: OPEN

## Summary

Pure refactor that lifts `config_loader` (and the
`CloudRequirementsLoader`, `CloudRequirementsLoadError`,
`LoaderOverrides`, `project_trust_key`) out of the giant `codex-core`
crate into `codex-config`. The `Cargo.lock` deltas show `codex-config`
gaining `base64`, `codex-exec-server`, `codex-git-utils`,
`core-foundation`, `dunce`, and `windows-sys` while `codex-core`
loses the same set — a clean transplant rather than duplication.
Downstream crates (`app-server-client`, `app-server`,
`codex-message-processor`) update their `use` paths from
`codex_core::config_loader::*` to `codex_config::*`.

## Specific findings

- `codex-rs/Cargo.lock:2287` — `codex-config` picks up the OS-shim
  deps (`core-foundation`, `windows-sys`, `dunce`) that the loader
  needs for path canonicalization and project-trust key derivation
  on macOS/Windows. Mirroring the removal at
  `codex-rs/Cargo.lock:2399` for `codex-core` confirms ownership
  has fully moved; no risk of two copies of `CloudRequirementsLoader`
  diverging.
- `codex-rs/app-server-client/src/lib.rs:41` — the `use` block now
  pulls `CloudRequirementsLoader`, `LoaderOverrides`,
  `NoopThreadConfigLoader`, `RemoteThreadConfigLoader`, and
  `ThreadConfigLoader` from `codex_config`. Re-export discipline is
  important here: any external SDK consumer that imported these
  through `codex_core::config_loader` will break. The PR title says
  "[codex]" so this is internal, but a `pub use
  codex_config::CloudRequirementsLoader;` shim in `codex_core` for
  one release would smooth out-of-tree forks.
- `codex-rs/app-server/src/codex_message_processor.rs:220` — both
  `CloudRequirementsLoadError` and the lower-cased
  `codex_config::loader::project_trust_key` come from the new home.
  Worth double-checking that `loader::` is the canonical sub-module
  name (vs. re-exporting `project_trust_key` at the crate root); the
  asymmetry between top-level `CloudRequirementsLoader` and the
  module-qualified `loader::project_trust_key` is a minor wart.
- The `+484/-502` shape (net `-18`) is consistent with a move plus
  some dead-import cleanup, not a behavior change. No new tests are
  needed *if* the existing config-loader tests moved with the code;
  reviewers should confirm `codex-config/tests/` got the
  `cloud_requirements_*` cases that previously lived under
  `codex-core/tests/`.

## Risks

- The `codex-core` crate dropping `core-foundation` and `windows-sys`
  could break a downstream crate that pulled them in transitively
  via `codex-core` rather than declaring its own dep. Worth a
  `cargo tree -i core-foundation` check across the workspace.
- `codex-exec-server` is now a dep of `codex-config`, which inverts
  the usual layering (config → exec rather than exec → config). If
  this is just to share a struct definition, extracting that into a
  `codex-config-types` crate would keep `codex-config` lightweight
  for consumers like the SDK.

## Verdict

`merge-after-nits`

## Rationale

Mechanically correct refactor that materially shrinks `codex-core`'s
surface, which has been the right direction for this codebase for
several drips. The two nits — re-export shim for one release, and
the `codex-config → codex-exec-server` reverse-direction dep — are
small and either resolvable in this PR or follow-up. No behavior
change visible in the diff slice reviewed.

## What I learned

When a project has been steadily decomposing a god-crate
(`codex-core` here), each move PR is also an audit of which
out-of-tree consumers were silently coupled to the old import
path. A one-release `pub use` shim is cheap insurance.
