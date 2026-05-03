# Review: openai/codex #20890 — Thread Windows protected metadata targets

- PR: https://github.com/openai/codex/pull/20890
- Head SHA: `872136802587e0569dd8fa560aea35e1848a9ac9`
- Author: evawong-oai
- Size: +78 / -0

## Summary
Threads the previously-planned `ProtectedMetadataTarget` list (#20889)
through the Windows sandbox setup boundary: from `core/exec.rs` into
the sandbox crate's `ElevatedSandboxCaptureRequest`, the legacy
`run_windows_sandbox_capture_legacy` entry point, and
`require_logon_sandbox_creds` / `SandboxSetupRequest`. This is the
"plumbing only" PR — no enforcement logic yet.

## Specific citations
- `codex-rs/core/src/exec.rs:649-670` — projects each
  `WindowsProtectedMetadataTarget` into a `codex_windows_sandbox::
  ProtectedMetadataTarget`, mapping the two `Mode` variants
  (`ExistingDeny`, `MissingCreationMonitor`) explicitly. Match is
  exhaustive — good.
- `codex-rs/core/src/exec.rs:689` and `:702` — the elevated and legacy
  spawn paths both receive `&protected_metadata_targets`. Symmetric.
- `codex-rs/windows-sandbox-rs/src/elevated_impl.rs:5,21,131,154` —
  adds the field to `ElevatedSandboxCaptureRequest`, destructures it,
  and forwards it into `require_logon_sandbox_creds`. The destructure
  at line 131 is exhaustive (no `..`), which is the right call so a
  future field addition fails compilation here.
- `codex-rs/windows-sandbox-rs/src/identity.rs:143,206,226` — adds the
  parameter and threads it into both branches of
  `require_logon_sandbox_creds` (the network-identity refresh branch
  at :206 and the cached-identity branch at :226). Both populate
  `SandboxSetupRequest.protected_metadata_targets` with the same
  vector — consistent.
- `codex-rs/windows-sandbox-rs/src/lib.rs:174-177` — re-exports
  `ProtectedMetadataMode` and `ProtectedMetadataTarget` so the core
  crate can name them. Properly gated by `#[cfg(target_os = "windows")]`.
- `codex-rs/windows-sandbox-rs/src/lib.rs:351,366` — the legacy
  `run_windows_sandbox_capture_legacy` shim now takes
  `_protected_metadata_targets: &[ProtectedMetadataTarget]` (note the
  `_` prefix). Confirms this PR is plumbing-only; #20891 wires the
  enforcement.

## Verdict
**merge-as-is**

(Pure plumbing PR, exhaustive destructures, symmetric across all
spawn paths, properly gated for non-Windows. Nothing actionable.)
