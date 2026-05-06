# block/goose #9045 — feat(acp): expose built-in skills through sources list acp calls

- PR: https://github.com/block/goose/pull/9045
- Head SHA: `143cfbdd93d2875fbf89f0ab1cd0e3c21eff3326`
- Base: `main`
- Size: +183 / -26 across 6 files
- Files:
  - `crates/goose-sdk/src/custom_requests.rs` (+9/-6)
  - `crates/goose/acp-schema.json` (+3/-3)
  - `crates/goose/src/skills/mod.rs` (+2/-0)
  - `crates/goose/src/sources.rs` (+119/-6)
  - `crates/goose/tests/acp_custom_requests_test.rs` (+34/-0)
  - `ui/sdk/src/generated/types.gen.ts` (+9/-6)

## Verdict
**merge-after-nits**

## Rationale
This extends the existing `_goose/sources/list` ACP method to handle a new `type: "builtinSkill"` value, so clients can discover shipped built-ins separately from filesystem-backed (editable) skills. The design split — read-only built-ins kept distinct from writable skill sources — is correct: it lets the UI render a "managed" badge without granting clients write access to baked-in resources.

The schema/SDK side is regen-clean:
- `acp-schema.json` (+3/-3) and `ui/sdk/src/generated/types.gen.ts` (+9/-6) are auto-generated artifacts. Their small line counts are a good signal that the contract change is genuinely additive (one new enum variant + a doc rewrite), not a wider reshape.
- `goose-sdk/src/custom_requests.rs` (+9/-6) updates the source DTO doc-comments to name both the filesystem-backed and built-in cases.

The runtime side concentrates the new behavior in `crates/goose/src/sources.rs` (+119/-6), which:
- Allows source listing for both `skill` and `builtinSkill` types, rejects unsupported types.
- Normalizes built-ins as global read-only entries with no supporting files.
- Adds coverage for built-in listing, filesystem precedence (filesystem skill with the same name wins), and rejecting built-in mutations.

`crates/goose/src/skills/mod.rs` (+2/-0) assigns built-ins a stable `builtin://skills/<name>` synthetic path. Synthetic URI scheme is the right call — gives clients a durable identifier without implying filesystem editability. As long as no other code path tries to `fs::canonicalize` on this string, it's safe.

`crates/goose/tests/acp_custom_requests_test.rs` (+34/-0) adds an integration test proving the new ACP request returns built-in skills. That's the right test layer — exercises the wire contract end-to-end, not just the internal helpers.

## Specific lines (per PR file list)
- `crates/goose/src/sources.rs:` — adds the dispatch on source type. The PR body claims "filesystem precedence" coverage; verify that the test asserts a filesystem `skill` with the same `name` as a `builtinSkill` shadows the built-in (not the other way around), since UI users expect their local override to win.
- `crates/goose/src/skills/mod.rs:` — `builtin://skills/<name>` URI assignment. Good.
- `crates/goose/tests/acp_custom_requests_test.rs:` — new test covers the happy path of `_goose/sources/list` with `type: "builtinSkill"`.

## Nits before merging
1. **Document the `builtin://` URI scheme contract in code.** Today only the PR body describes it. A comment in `skills/mod.rs` next to the assignment naming this as a synthetic, non-filesystem URI (and a list of the ~3 places that *must not* try to canonicalize it) would prevent a future contributor from passing the path to `tokio::fs::read_to_string` and getting confused by `ENOENT`.
2. **Negative test for mutation rejection.** The PR body claims "rejecting built-in mutations" is covered in `sources.rs` test additions, but the integration test in `acp_custom_requests_test.rs` only exercises listing. Worth a parallel integration test that calls a write-path source op against a `builtinSkill` URI and asserts the wire-level error response shape (status code + error message), since a regression here would silently allow clients to attempt destructive ops on baked-in resources.
3. **Schema regen reproducibility.** A short note in the PR body or a `Justfile`/`scripts/` reference for "how to regenerate `acp-schema.json` + `types.gen.ts`" would help reviewers verify the +3/-3 and +9/-6 diffs are the actual generator output and not hand-edited.
4. The PR description references `https://github.com/aaif-goose/goose/...` which appears to be an external/fork URL — confirm that's intentional and that the canonical path is up-to-date in the description before merge.

None of these block — the behavior is right and the wire contract is additive.
