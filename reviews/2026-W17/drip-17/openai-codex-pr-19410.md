# openai/codex PR #19410 — Remove js_repl feature

- **Author:** fjord-oai
- **Head SHA:** ad7879c2134fd6ac6e0dd8f530912a1fde26dafb
- **Files:** 63 files changed, +66 / −9244
- **Verdict:** `merge-as-is`

## What the diff does

Removes the in-tree JavaScript REPL tool and all of its scaffolding.

The deletions are sweeping but well-bounded:
- `codex-rs/core/src/tools/js_repl/mod.rs` (−2055), `mod_tests.rs`
  (−2912), `kernel.js` (−1833), `meriyah.umd.min.js` (−6) — the
  entire embedded JS sandbox.
- `codex-rs/core/src/tools/handlers/js_repl.rs` (−300),
  `handlers/js_repl_tests.rs` (−90),
  `tools/src/js_repl_tool.rs` (−55), and the corresponding
  registry/dispatch entries in `tools/spec.rs`, `tools/router.rs`
  (−12), `router_tests.rs` (−172), `tools/mod.rs` (−1).
- `codex-rs/core/tests/suite/js_repl.rs` (−795) and
  `tests/suite/view_image.rs` (−229) — entire test suites that
  depended on the REPL.
- `codex-rs/features/src/lib.rs` (+11 / −13) and `features/tests.rs`
  (+33 / −17): the `JsRepl` feature flag is removed from the enum
  and the tests are tightened to assert the new known-flag set.
- CI plumbing: `.github/actions/setup-bazel-ci/action.yml`,
  `.github/workflows/Dockerfile.bazel`, `run-bazel-ci.sh` all drop
  Node.js installation and the `--use-node-test-env` flag and the
  `CODEX_JS_REPL_NODE_PATH` test env. The `node-version.txt` file
  and `third_party/meriyah/LICENSE` are gone.
- `core/config.schema.json` (−25) drops the `js_repl` config keys.
- `core/src/agents_md.rs` (−42), `agents_md_tests.rs` (−34),
  `core/src/session/mod.rs` (−30), `session/tests.rs` (+2 / −18),
  `session/turn_context.rs` (−5), `tools/code_mode/mod.rs`
  (+1 / −2): all surface-level cleanups for the now-unused REPL
  context.

## Review notes

- The deletion is **structurally clean**. Every deletion has a
  matching call-site update; nothing is orphaned. The
  `features/src/lib.rs` change shrinks the flag enum by exactly one
  variant and the test in `features/src/tests.rs` is updated to
  reassert the new set, which is the right way to prove no stragglers.
- CI removes Node entirely from the Bazel test path. Worth
  double-checking that no remaining suite has an implicit Node
  dependency (e.g. via a `pnpm` install in the codex-cli npm package
  side, which is unrelated and should still ship its own Node).
  The `.codespellrc` change at line 3 also drops the `meriyah` skip
  pattern, which is fine because the file is gone.
- `tui/src/chatwidget/tests/popups_and_settings.rs` loses 24 lines —
  presumably the toggle UI for js_repl. Worth eyeballing that the
  remaining test expectations don't reference `JsRepl` strings.
- The schema change in `config.schema.json` is a soft breaking change
  for users with `js_repl: {...}` blocks — those will now produce
  unknown-key warnings. Given this is a removed feature, a one-line
  changelog entry calling that out is sufficient; the codebase
  already has a "warn and continue on unknown feature requirements"
  path (PR #19038) so existing configs won't crash.
- `NOTICE` and `third_party/meriyah/LICENSE` removal is correct
  third-party hygiene.

This is the kind of subtractive PR every long-running tool needs
periodically. Net `-9178` lines, the call graph closes cleanly, and
the CI surface gets simpler. Ship it.

## What I learned

Removing a feature this deep cleanly is mostly an exercise in
*finding the seams*: feature flag → handler module → router entry →
config schema → test fixtures → CI prereqs → third-party notice.
Missing any one of those leaves zombie code. The fact that this PR
visibly closes all seven seams in the diff is a useful template
for any "we're killing X" PR.
