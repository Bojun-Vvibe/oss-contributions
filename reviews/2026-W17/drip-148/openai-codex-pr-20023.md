# openai/codex #20023 — Respect filesystem read policy for fileParams uploads

- PR: https://github.com/openai/codex/pull/20023
- Head SHA: `0d5a38b145e016172370b52755d156b6e13dea1f`
- Author: colby-oai
- Files: `codex-rs/core/src/mcp_openai_file.rs` (+67/−0)

## Observations

- This closes a real sandbox-bypass hole: before this PR, the `openai/fileParams` MCP tool path called `build_uploaded_local_argument_value` and unconditionally opened/uploaded any resolved local path, even if the active `FileSystemSandboxPolicy` denied read access to it. That meant a tool call could exfiltrate files the user had explicitly scoped out of the agent's read perimeter — a textbook confused-deputy. The fix is the right one: gate the upload on `turn_context.file_system_sandbox_policy().can_read_path_with_cwd(...)` *before* the `auth` check so policy denial is reported regardless of auth state.
- Predicate ordering at `:103-115` is correct: the policy check fires first, *then* the `auth.is_none()` arm, *then* the actual file open. This means a denied path returns a stable, deterministic error message even if the user is logged out — important because the alternative ordering (auth-first) would have leaked policy info via differing error messages depending on session state. The error string at `:113-117` correctly distinguishes the array-index-bearing case (`field_name[index]`) from the scalar case, matching the existing convention in this file for surfacing argument provenance to the model.
- The denial message says `is blocked by the filesystem sandbox policy` — accurate and not over-specific. It deliberately does not echo the *resolved* absolute path (only the user-supplied `file_path`), which is the right info-disclosure stance: a malicious upstream tool author can't probe CWD by seeding relative paths and reading the resolved absolute path back in the error. (Compare to a naïve `format!("blocked: {resolved_path:?}")` which would be a side-channel.)
- Test cell at `:268-313` is well-shaped: it constructs an explicit `FileSystemSandboxPolicy::restricted` with a `Root`-rooted Read entry plus a path-specific `FileSystemAccessMode::None` override on the *exact* file the tool tries to upload. This pins both halves of the policy semantics: (a) global Read root permits the rest of the FS, (b) per-path None overrides that root for the specific blocked-file case. Without that override, the test wouldn't actually exercise the deny path. The `expect_err` + exact-string `assert_eq!` is strict but appropriate for a security-relevant error message — silently changing the wording would now require an intentional test update rather than a silent regression.
- Notable that `turn_context.permission_profile = PermissionProfile::from_runtime_permissions(...)` is set explicitly at `:289-292` before the call. This is the right pattern for these tests because the policy-check function reads from `permission_profile` (via `file_system_sandbox_policy()`) — without rebuilding the profile from the synthetic policy, the test would have exercised whatever the default `make_session_and_context` baked in.

## Risks / nits

- The success path (file *is* readable, upload proceeds) has no new test cell. The existing positive tests at `rewrite_argument_value_for_openai_files_rewrites_scalar_path` etc. do exercise the upload code, but none of them set up an explicit Read-allow policy — they rely on the test default policy permitting reads. Worth adding a positive cell with the same `restricted` shape but `FileSystemAccessMode::Read` on the file, just to pin that the gate doesn't accidentally over-block.
- No test for the `index` arm of the error message (`field_name[index]` form). The negative test only exercises the scalar `index: None` case at `:308`. The two error formats at `:111-117` are independent string templates, so a future refactor could silently break the array-index path without any test catching it. Add a parallel cell driving the array-index codepath.
- `turn_context.file_system_sandbox_policy()` is called twice if the success path proceeds (once here, once inside the actual file-open code that follows). For deny-of-many-files this is fine, but for the upload of a large fileParams batch it's a per-file double-lookup — likely a no-op in practice given the policy is in-memory, but worth noting if `file_system_sandbox_policy()` ever becomes more expensive.
- `can_read_path_with_cwd(resolved_path.as_path(), turn_context.cwd.as_path())` — passing `cwd` here is correct for relative-path resolution semantics inside the policy engine, but worth verifying that `resolved_path` is guaranteed to already be absolute (via `turn_context.resolve_path`). If it can be relative, the policy lookup might evaluate against the wrong base. A doc comment on `resolve_path`'s post-condition would close this.

## Verdict: `merge-as-is`

**Rationale:** Surgical security fix in the right primitive at the right boundary — gate the upload at the policy layer before the auth or file-open paths fire. Test pins both halves of the policy (Root Read + per-path None override) plus the exact deny-message shape. The info-disclosure stance (don't echo resolved path back) is right. Nits above are positive-path-coverage hygiene, not blockers — the deny direction is the security-critical one and that's covered.

Date: 2026-04-29
