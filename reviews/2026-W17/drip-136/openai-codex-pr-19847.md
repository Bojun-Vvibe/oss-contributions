# openai/codex #19847 — Enforce workspace metadata protections in Seatbelt

- PR: https://github.com/openai/codex/pull/19847
- Head SHA: `a767cac90f8d`
- Diff: 181+/92- across 2 files (`codex-rs/sandboxing/src/seatbelt.rs`, `codex-rs/sandboxing/src/seatbelt_tests.rs`)
- Base: `main`

## Verdict: merge-after-nits

## Rationale

- **Closes a real "writable-root grant silently shadows protected metadata" hole.** Today a `WritableRoot` whose `root.join(".git")` / `.codex` / `.agents` etc. are *not* covered by an existing `excluded_subpaths` carve-out is granted `(subpath (param "WRITABLE_ROOT_N"))` unconditionally, which Seatbelt evaluates as "everything under root is writable" — including the metadata paths the protocol-level `PROTECTED_METADATA_PATH_NAMES` was designed to defend. The PR closes the gap by computing per-root metadata-name requirements at `seatbelt.rs:67-83` (`protected_metadata_names_for_writable_root`) and wiring them into the same `(require-all …)` shape the existing excluded-subpath carve-outs use at `:41-45`.
- **The "name not yet excluded AND not writable per-policy" filter is the correct gate.** `protected_metadata_names_for_writable_root` at `seatbelt.rs:72-82` only adds a metadata name to the deny list if (a) it isn't already in `writable_root.protected_metadata_names` (no double-deny) AND (b) the policy's own `can_write_path_with_cwd(root.join(name), cwd)` says the path *should not* be writable. That second clause is load-bearing — a workspace whose policy explicitly grants `.git` writes (e.g. a tooling profile that needs to commit) won't have `.git` re-denied at the seatbelt layer.
- **Regex shape is right.** `seatbelt_protected_metadata_name_regex` at `:53-65` anchors on `^{root}/{name}(/.*)?$`, escapes both halves with `regex_lite::escape`, special-cases `root == "/"` to avoid `//.git`, and trims trailing slashes from `root` before escaping (`:55-57`). The double-`"`-escape on `:43` (`replace('"', "\\\"")`) is needed because the regex sits inside a `(regex #"…")` form whose Seatbelt-side parser uses `"` as the delimiter.
- **Empty-list short-circuit preserved.** `:31-33` widens the "no excluded subpaths → emit `(subpath …)` directly" optimization to also require `protected_metadata_names.is_empty()`, so writable roots that need no metadata protection still get the cheap one-clause grant.
- **Three call sites updated symmetrically.** `:88-93`, `:96-107`, `:108-114`, `:116-123` — all four `SeatbeltAccessRoot` constructions add `protected_metadata_names`, three of them as `Vec::new()` (full-disk read access path, full-disk write access path, danger-full-access path) and only the policy-derived writable-roots path at `:99-107` actually populates names. That asymmetry is correct: only writable roots can leak writable metadata.
- **Test coverage covers both the "no carve-out" path and the "explicit carve-out" path.** `seatbelt_tests.rs:167-172` asserts the metadata regex deny requirements appear in the full-disk-with-explicit-unreadable policy; `:194-199` asserts them for the cwd-derived writable root in `create_seatbelt_args_with_read_only_git_and_codex_subpaths`. The new helper `seatbelt_protected_metadata_name_requirements` at `:140-158` reproduces the regex shape inline so a future drift in `seatbelt.rs::seatbelt_protected_metadata_name_regex` would fail the assertion.

## Nits / follow-ups

- **`PROTECTED_METADATA_PATH_NAMES` ordering matters for the regex assertion.** The test at `:140-158` joins regex deny clauses in `PROTECTED_METADATA_PATH_NAMES` iteration order; the production code at `:67-83` also iterates in the same order. If a future PR reorders the constant or wraps it in a `HashSet`, the `.contains(&seatbelt_protected_metadata_name_requirements(…))` substring check at `:167-172` will silently break. A one-line "iteration order is part of the assertion" comment on the constant would prevent the trap.
- **No regression test for the policy-grants-metadata-write-back case.** The `can_write_path_with_cwd(...)` short-circuit at `:78` is the load-bearing clause that allows tooling profiles to keep `.git` writable. A test that constructs a `FileSystemSandboxPolicy` explicitly granting `.git` write under a writable root and asserts the resulting policy *does not* contain a `^{root}/\.git(/.*)?$` deny would lock that semantics down.
- **`regex_lite::escape` allocates per call site.** Hot path is once per writable-root × metadata-name on policy build — fine for sandbox-construction call site, but worth a one-line comment that this runs on the privileged setup path, not per-syscall, so the cost is bounded.
- **The `_ = dot_agents_canonical` rebind at test line 187** is a code-style artifact of the test-fixture rename; consider removing the field from `PopulatedTmp` if no test uses it instead of explicitly throwing it away.

## What I learned

The "writable-root → all writes allowed unless explicitly excluded" default is the right shape for Seatbelt grants, but it punts the responsibility for protected-metadata defense onto whoever constructs the writable-root set. Pushing the `PROTECTED_METADATA_PATH_NAMES` enforcement *into* the seatbelt builder (rather than expecting every caller to remember to add `.git`/`.codex` to `excluded_subpaths`) is the right place to put the invariant — it converts a silent-omission footgun ("oops, your writable root grants `.git` writes") into an opt-out gesture ("policy says `.git` is writable, so I won't deny it"). The `can_write_path_with_cwd` guard preserves operator override without forcing it through a separate flag.
