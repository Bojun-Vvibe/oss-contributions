# Review — sst/opencode#25631 — fix(opencode): normalize subdirectory git paths

- PR: https://github.com/sst/opencode/pull/25631
- Head: `1e1439bed1d64d0abcbefef4e0d7ad1dfb1dd8fb`
- Verdict: **merge-after-nits**

## What the change does

Fixes file paths returned from `git diff/status/numstat/ls-files` when
opencode is launched from a subdirectory of a repo. New helpers in
`file/index.ts` and `git/index.ts` (`slashes`, `stripPrefix` / `stripGitPrefix`)
strip the `git rev-parse --show-prefix` value from each emitted path, and the
git invocations now pass `-- .` so git limits output to the cwd subtree.
Also adjusts `git/index.ts` `show()` so `target` becomes `./{file}` when there
is no prefix, avoiding ambiguity for files whose names look like refs.

## Strengths

- Root cause is right: `git diff --numstat HEAD` reports paths relative to
  the **repo root**, not cwd, so file events for subtree-launched sessions
  pointed to wrong locations. Adding `-- .` plus prefix-stripping is the
  correct combination.
- `slashes()` normalizes `\` → `/` before comparison, so the prefix check
  works on Windows checkouts too.
- The change is symmetric: `Git.status`, `Git.diff`, `Git.stats`, plus all
  three call sites in `File.status` (`diff --numstat HEAD`, `ls-files
  --others`, `diff --diff-filter=D HEAD`) all use the same trim.

## Nits / asks

1. `stripPrefix` is duplicated verbatim in `file/index.ts` and `git/index.ts`.
   Move it (and `slashes`) to a shared util — drift between the two copies
   would re-introduce the same bug.
2. `Git.show()`: switching `target` to `./{file}` when prefix is empty is a
   behavior change for callers who pass an absolute-ish ref. Worth a unit
   test asserting `git show <ref>:./<file>` resolves identically to plain
   `<file>` in a non-subtree repo.
3. `prefix` in `git/index.ts` is fetched on every `status/diff/stats` call.
   Cheap, but an in-process memo per `cwd` would save a `git rev-parse`
   round-trip on hot paths like file-watcher refresh.

Mergeable after extracting the shared helper.
