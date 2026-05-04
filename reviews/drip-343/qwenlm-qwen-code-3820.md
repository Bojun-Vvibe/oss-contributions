# QwenLM/qwen-code #3820 — fix(core): unescape shell-escaped file paths in Edit, WriteFile, and ReadFile tools

- PR: https://github.com/QwenLM/qwen-code/pull/3820
- Head SHA: `92bb271a601a320fdd889daf080ca58bf12c707b`
- Diff size: +562 / -167 across 23 files

## Summary

When the model produces tool inputs containing shell-escaped paths
like `path/to/my\ file.txt`, the tool layer now unescapes them
centrally in `coreToolScheduler.ts` before the tool handler sees the
arguments. A new `PATH_ARG_KEYS` constant + `unescapePath` helper in
`packages/core/src/utils/paths.ts` is reused across:
`coreToolScheduler.ts`, `speculationToolGate.ts`, and the
`edit/glob/grep/ls/lsp/read-file/ripGrep/write-file` tools.

## Citations

- `packages/core/src/core/coreToolScheduler.ts:145` — imports
  `unescapePath, PATH_ARG_KEYS`. The pre-hook normalization loop
  (`:158-164`) iterates `PATH_ARG_KEYS` and unescapes string values
  in-place on `normalizedArgs`. Centralizing in the scheduler is
  the right layering — individual tools shouldn't all reimplement
  shell-unescape semantics.
- `coreToolScheduler.ts:181-184` — second unescape loop on
  `toolInput`. The author added an explicit comment at `:194-196`
  noting that `unescapePath` is **not idempotent**: `file\\(X)`
  collapses one backslash per pass, so a second pass would corrupt
  the path. This is an excellent landmine-flag, but it also signals
  fragility — the contract "exactly one unescape pass per tool
  invocation" is enforced only by convention. If anyone later adds
  another normalization step downstream that touches the same field,
  silent corruption results. Consider tagging the args with a
  `Symbol('unescaped')` brand or wrapping in a typed `UnescapedPath`
  newtype.
- `packages/core/src/followup/speculationToolGate.ts:117-122` —
  replaces the previously-inlined `pathKeys = ['file_path', ...]`
  array with the shared `PATH_ARG_KEYS`. Good de-duplication; ensures
  the speculation gate sees the same keyset as the real scheduler so
  speculative pre-checks don't diverge.
- `packages/cli/src/ui/hooks/atCommandProcessor.test.ts:222, 788` —
  tests that exercise escaped-space paths are now wrapped in
  `it.skipIf(process.platform === 'win32')`. Reasonable: backslash
  has different semantics in Windows paths so the test fixture
  doesn't translate. Worth a follow-up to add Windows-specific
  coverage that asserts the *non*-unescape behavior on win32.
- 9 tool files (`edit.ts`, `glob.ts`, `grep.ts`, `ls.ts`, `lsp.ts`,
  `read-file.ts`, `ripGrep.ts`, `write-file.ts`) all touched. Spot
  check needed to confirm none of them was *also* doing its own
  unescape that now double-processes — the author's idempotency
  warning suggests they were aware of this but reviewer should
  verify by grep'ing for `\\\\\\\\` literal handling in those files
  pre-and-post diff.

## Risk

Moderate. The fix is correct in spirit but the non-idempotent helper
+ multiple call sites is a footgun. Likely impact on existing users:
positive (escaped paths work) with a small risk of regression on
paths that legitimately contain a literal backslash followed by a
space character (rare on POSIX, common on Windows).

## Verdict

`request-changes` — recommend (1) brand the unescaped path with a
nominal type (or at minimum a `WeakSet`-tracked sentinel) so a
future second-unescape can be detected at runtime, and (2) add
Windows-platform coverage (not just `skipIf`). The Windows skip
without a paired Windows-positive test is a coverage hole on the
exact platform where path semantics differ most.
