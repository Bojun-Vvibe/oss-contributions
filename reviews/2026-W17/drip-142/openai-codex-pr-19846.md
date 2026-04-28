# openai/codex#19846 — Add workspace metadata protection policy primitive

- **PR**: https://github.com/openai/codex/pull/19846
- **Author**: @evawong-oai
- **Head SHA**: PR head (see `gh pr view`)
- **Base**: codex-rs main branch
- **State**: OPEN
- **Scope**: medium-large — `codex-rs/protocol/src/permissions.rs`, `codex-rs/protocol/src/protocol.rs`, `codex-rs/linux-sandbox/src/bwrap.rs` (~700 changed lines incl. tests)

## Summary

Promotes "this directory name is protected metadata" from a hard-coded subpath list (`writable_root.join(".git")`, `.join(".agents")`, `.join(".codex")`) into a first-class field `WritableRoot::protected_metadata_names: Vec<String>` plus a top-level helper `forbidden_agent_metadata_write(path, cwd, policy)`. The motivation is a real correctness gap: the old `default_read_only_subpaths_for_writable_root` only added `.codex` if the directory **already existed** (or for the workspace root), and `.git`/`.agents` similarly. So an agent under a `Restricted` policy could create a fresh `.git/`, `.codex/`, or `.agents/` subdirectory under a writable root and bypass the read-only protection by virtue of "the path didn't exist when the policy was projected." The new field-on-`WritableRoot` plus `is_metadata_write_denied(path, cwd)` gate at `can_write_path_with_cwd` closes that gap independent of on-disk existence.

## Diff anchors

- `codex-rs/protocol/src/permissions.rs:21-30` — `PROTECTED_METADATA_PATH_NAMES = &[".git", ".agents", ".codex"]` constant + `is_protected_metadata_name(name: &OsStr)` predicate. The constant is the single source of truth — every other site (`is_metadata_write_denied`, `protected_metadata_names_for_writable_root`, `metadata_path_name`, the test at `:1907-1912`) iterates over it. This is the right shape.
- `:46-75` — `forbidden_agent_metadata_write` returns `Option<&'static str>` naming *which* metadata kind is being protected; the load-bearing escape hatch is `has_explicit_write_entry_for_metadata_path` at `:288-302`, which lets a policy explicitly grant write under a metadata directory (e.g., `.codex/state.json`) and the protection is bypassed for that specific subtree. Without this, the new gate would break legitimate use cases like opencode's own state directory.
- `:83-92` — `can_write_path_with_cwd` now layers two checks: `resolve_access_with_cwd(...).can_write()` AND `!is_metadata_write_denied(...)`, with a fast-path `if has_full_disk_write_access() return true` so unrestricted policies aren't burdened. Order matters: cheap `can_write()` first, expensive metadata-name walk only when needed.
- `:233-256` — `protected_metadata_names_for_writable_root` populates the per-root `protected_metadata_names` field by checking each metadata name against *both* the canonicalized root and every raw writable root (handles symlink-to-real-path divergence so the protection survives canonicalization). The "all metadata_paths reject write → add the name" condition is the right way to avoid double-protection when the policy already has an explicit read-only rule for that path.
- `:316-323` — gitdir pointer file parsing now strictly requires the `gitdir:` prefix (was: any `key: value` was accepted). Old code silently accepted malformed `.git` pointer files and resolved against the wrong base. Right tightening.
- `codex-rs/protocol/src/protocol.rs:1108-1120` — `WritableRoot::is_path_writable` now also calls `path_contains_protected_metadata_name(path)` after the existing `read_only_subpaths` check. Defense-in-depth: even if a downstream sandbox doesn't consult `FileSystemSandboxPolicy` directly, the `WritableRoot` payload still encodes the protection.
- `codex-rs/linux-sandbox/src/bwrap.rs:259-262` — minimal change: adds `protected_metadata_names: Vec::new()` to the constructed `WritableRoot` for the linux bwrap full-disk-write case. Correct because bwrap's full-disk path doesn't need metadata gating; the protection is enforced at the sandbox-policy layer above.
- Test at `:1915-1965` (`filesystem_policy_blocks_protected_metadata_path_writes_by_default`) is the right shape — pins the *positive* invariant ("`.git/config`, `.agents/config`, `.codex/config.toml` all reject write under restricted policy with explicit Write rule on the root") AND the *projected-data* invariant ("`writable_roots[0].protected_metadata_names == [.git, .agents, .codex]`"). Two layers of assertion = two layers of regression detection.
- `:2516-2522` — flips three previous "is NOT direct-runtime-enforced" assertions to "IS direct-runtime-enforced" with the message "metadata-name protections are outside the legacy SandboxPolicy writable-root contract." This is the right call — the new protection is novel and downstream runtimes (bwrap, seatbelt) need to be told "you can't trust the legacy projection alone anymore." However, it does mean every existing callsite of `needs_direct_runtime_enforcement` for restricted policies will now return `true`, potentially expanding the work the runtime layer does. Worth a perf note in the PR body.

## What I'd push back on (nits)

1. **`resolve_candidate_path` semantics changed** — `:149` switches from `AbsolutePathBuf::resolve_path_against_base(path, cwd)` (infallible, normalized) to `AbsolutePathBuf::from_absolute_path(cwd).ok()?.join(path)` (now `Option`). Callers that expected `Some` for any relative path now get `None` if `cwd` itself isn't already canonical-absolute. Walk all callers and confirm none rely on the old infallibility.
2. **No symlink-attack test** — the natural attack is "agent creates a symlink `.git -> /etc` then writes through it." `path_contains_protected_metadata_name` does a `strip_prefix` on the *as-given* path, not a canonicalized one. If the policy's `root` is a real path but the `target` reaches it via a symlinked alias, the prefix check fails and the protection silently degrades. Either canonicalize both sides in `is_metadata_write_denied`, or document the assumption that the seccomp/bwrap layer below has already resolved symlinks.
3. **`metadata_child_of_writable_root` returns first match only** — `.next()` at `:230`. If a target is simultaneously under two writable roots (overlapping policy entries) and only one of them protects metadata, the iteration order determines whether protection applies. Sort `resolved_entries_with_cwd` deterministically (longest-prefix first) before `.next()`, or document the order contract.
4. **`PROTECTED_METADATA_PATH_NAMES` is not configurable** — hard-coded list means any future protected directory (`.continue`, `.cursor`, `.aider`, etc.) requires a code change. Consider exposing as a `&[&str]` parameter on `FileSystemSandboxPolicy::restricted_with_protected_metadata_names(...)` for forward extensibility, and have the current constant be the default.

## Verdict

**merge-after-nits** — the design is sound (closes a real "metadata directory created post-hoc bypasses protection" gap with the right escape hatch for explicit write rules, and the test pins both the behavioral and projected-data invariants), but the symlink-resolution concern is load-bearing for a security primitive and `resolve_candidate_path`'s infallibility regression needs a caller audit before merge.

Repo coverage: openai/codex (sandbox policy primitive — workspace metadata directory protection in the FileSystemSandboxPolicy + WritableRoot data model).
