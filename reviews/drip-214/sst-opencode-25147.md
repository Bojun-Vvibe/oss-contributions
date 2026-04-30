# Review: sst/opencode#25147 — Isolate TUI thread cwd resolution test

- PR: https://github.com/sst/opencode/pull/25147
- Merge SHA: `cedff6fb89f1c2d4e3b74803b72304b30d37c079`
- Files: `packages/opencode/src/cli/cmd/tui/thread.ts` (+6/-4), `packages/opencode/test/cli/tui/thread.test.ts` (+7/-119)
- Verdict: **merge-as-is**

## What it does

Refactors the `TuiThreadCommand` test to stop driving the full handler (which
required mocking `App.tui`, `UI.error`, `Timeout.withTimeout`, `Network`,
`Win32`, `globalThis.Worker`, `process.stdin.isTTY`, `process.env.PWD`, and
`process.cwd`). Extracts the symlink-resolution branch into a pure
`resolveThreadDirectory(project?, envPWD?, cwd?)` function at `thread.ts:74-78`
and tests it directly.

## Notable changes

- `thread.ts:74-78` — new exported pure function with three injectable
  parameters that default to `process.env.PWD` / `process.cwd()`. The body is
  the exact same `Filesystem.resolve` ladder that previously lived inline at
  `thread.ts:127-130`: resolve `envPWD ?? cwd` to get `root`, then either
  resolve `project` (absolute or joined under `root`) or resolve `cwd`
  directly. Behaviour is byte-identical for the production path because
  `TuiThreadCommand` at `thread.ts:133` now calls
  `resolveThreadDirectory(args.project)` with no override args, which
  re-reads the same env/cwd globals.
- `thread.test.ts:1-28` — entire test fixture replaced. The previous setup
  spun a `TestWorker` extending `EventTarget` with handcrafted `postMessage`
  RPC handling for `fetch`/`server`/`snapshot`, `spyOn`'d 6 modules, mutated
  `process.stdin.isTTY` via `Object.defineProperty`, swapped
  `globalThis.Worker`, then invoked the full handler and caught a sentinel
  `stop` error. New version just calls `resolveThreadDirectory(project, link,
  tmp.path)` and asserts the return value.
- `thread.test.ts:18-19` — the `await using tmp = await tmpdir({ git: true })`
  + `fs.symlink(tmp.path, link, ...)` setup is preserved because the test's
  point is still that `PWD` pointing at a symlink resolves to the real path.
  The two `test.serial(...)` calls collapsed to plain `test(...)` because the
  test no longer mutates real env vars.

## Reasoning

This is a textbook "shrink the unit under test" refactor. The previous test
was a 119-line integration harness that exercised the full handler graph just
to verify a 4-line resolution decision; any of the 8 mocked dependencies
breaking would fail this test for unrelated reasons. The new test is 7 lines
and pins exactly the property the comment on the original code calls out
("Resolve relative `--project` paths from PWD, then use the real cwd after
chdir so the thread and worker share the same directory key" — `thread.ts:128`
in the prior revision).

The extracted function preserves the production path because the call site
passes no overrides, so `envPWD` and `cwd` still default to the same globals
they used to be read from inline. The author also kept the symlink-creation
test fixture rather than substituting a fake `Filesystem.resolve`, which is
the right call: the load-bearing behaviour is that `Filesystem.resolve`
follows symlinks, and faking it would test nothing.

The test fixture removal is also a known-good-direction cleanup: the deleted
`TestWorker` was a custom `EventTarget` subclass with hand-rolled RPC
matching, and the deleted `setup()` helper depended on Bun's `mock.restore()`
correctly resetting `spyOn`s but not `mock.module()`s — a known fragility the
prior revision called out in a 4-line comment that is now obsolete.

## Nits (non-blocking)

1. `thread.ts:74` — the parameter name `envPWD` reads a bit cryptically;
   `pwd` or `envPwd` would match the surrounding camelCase convention. Not
   worth churning.
2. `thread.test.ts:23-24` — the `try { fs.symlink(...); expect(...) } finally
   { fs.rm(link) }` shape works but the `await using tmp` already provides
   cleanup for `tmp.path`; the `link` cleanup could be lifted into a
   `using`-style fixture for symmetry. Not a real win.
3. The PR removes the comment block about Bun `mock.module()` cache
   semantics (`#7823`/`#12823`); the repo loses that institutional knowledge.
   If the broader codebase still has tests using `mock.module()`, consider
   migrating the comment to a top-level testing-conventions doc.
