# openai/codex PR #20891 — Enforce Windows protected metadata targets

- Link: https://github.com/openai/codex/pull/20891
- Head SHA: `24c064dec138507231a180daee78ba123ad2b125`
- Author: evawong-oai
- Size: +153 / −3 (4 files)

## Files changed
- `codex-rs/windows-sandbox-rs/src/elevated_impl.rs` — `+10 / −1`
- `codex-rs/windows-sandbox-rs/src/lib.rs` — `+14 / −2`
- `codex-rs/windows-sandbox-rs/src/protected_metadata.rs` — new (76 lines)
- `codex-rs/windows-sandbox-rs/src/setup_main_win.rs` — `+53`

## Reasoning

This is a tightly-scoped Windows sandbox security change. The model is:
- For `ProtectedMetadataMode::ExistingDeny` targets that already exist on disk, push the path into the sandbox deny set so the WFP/ACL layer rejects writes.
- For `ProtectedMetadataMode::MissingCreationMonitor` targets, watch and *delete after the fact* anything the sandboxed command created at those paths, and force a non-zero exit if violations were found.

Specific call-outs:

1. `codex-rs/windows-sandbox-rs/src/protected_metadata.rs:25` is the cleanup loop:
   ```
   pub(crate) fn cleanup_created_monitored_paths(&self) -> Result<Vec<PathBuf>> {
       let mut removed = Vec::new();
       for path in &self.monitored_paths {
           if std::fs::symlink_metadata(path).is_err() {
               continue;
           }
           remove_metadata_path(path)
               .with_context(|| format!("failed to remove protected metadata {}", path.display()))?;
           removed.push(path.clone());
       }
       Ok(removed)
   }
   ```
   This is post-hoc enforcement (TOCTOU): a sandboxed process can create the file, *exfiltrate or use* its contents, then we delete it. For metadata-creation prevention this is acceptable — the goal is "the artifact must not survive the sandbox run", not "the artifact must never be created" — but the PR description should make that contract explicit so future readers don't think it's an actual block.

2. `codex-rs/windows-sandbox-rs/src/lib.rs:443` ties the cleanup to the exit code:
   ```
   let protected_metadata_violations =
       protected_metadata_guard.cleanup_created_monitored_paths()?;
   if !protected_metadata_violations.is_empty() && exit_code == 0 {
       exit_code = 1;
   }
   ```
   Forcing `exit_code = 1` on violation is right, but the violation list is silently discarded — the caller learns "exit 1" but not *which* path was created. Recommend logging the removed paths via `log_error` (you already have `log_success`/`log_start` infrastructure in this module) so post-mortems can identify the bypass attempt.

3. `codex-rs/windows-sandbox-rs/src/elevated_impl.rs:144` calls `prepare_protected_metadata_targets(...)` *before* the sandboxed command launches, but the elevated path doesn't seem to push deny paths into the WFP/ACL set the way `lib.rs:438` does. That looks intentional (elevated runner has its own deny mechanism) but the asymmetry warrants a one-line comment explaining why the elevated path uses cleanup-only enforcement.

4. `remove_metadata_path` at `protected_metadata.rs:62` correctly distinguishes `is_dir()` (recursive `remove_dir_all`) from files/symlinks (`remove_file`), and the `metadata.is_dir() && !metadata.file_type().is_symlink()` guard prevents accidentally following a symlink into a directory the sandbox shouldn't reach. Good.

PR's known gap (full PowerShell sandbox runner blocked by a baseline failure on `main`) is acknowledged. Validation runs cited (Rust formatting, sandbox unit tests, core unit tests, cmd sandbox smoke) cover what can be covered.

## Verdict

`merge-after-nits`

Solid, well-bounded sandbox hardening. Before merge:
- Log the violation paths in `lib.rs:445` and the elevated equivalent so the security-relevant signal isn't lost.
- Add a comment at the top of `protected_metadata.rs` explicitly documenting the post-hoc-deletion semantics (current doc-comment is "Layer: Windows enforcement..." which is right but skips the contract).
- Note the elevated-vs-non-elevated deny-path asymmetry in `elevated_impl.rs:144`.
