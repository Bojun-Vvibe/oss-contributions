# PR #24863 — fix: prevent sibling instruction path leaks

- **Repo:** sst/opencode
- **Link:** https://github.com/sst/opencode/pull/24863
- **Author:** rubencu
- **State:** OPEN
- **Head SHA:** `1cc2ef3f60af9a63ab32a45ef2761b6e131d887d`
- **Files:** `packages/core/src/filesystem.ts`, `packages/opencode/src/util/filesystem.ts`, `packages/opencode/src/session/instruction.ts`, plus 3 test files (+88/-10)

Closes #24862.

## Context

`AppFileSystem.contains(parent, child)` and `.overlaps(a, b)` were the load-bearing predicates deciding whether a candidate AGENTS.md file lives "inside" the project root and should be auto-attached to the model context. The old implementations were one-line wrappers around `path.relative(parent, child).startsWith("..")`. That predicate is **wrong by construction** for sibling paths whose names share a prefix:

```
relative("/tmp/project", "/tmp/project-sibling/file.ts") === "../project-sibling/file.ts"
```

That string starts with `..` so the old `contains` actually returned `false` for sibling paths — but the bug was in the *consumer*, `instruction.ts:201`:

```
while (current.startsWith(root) && current !== root) {
```

`current.startsWith(root)` is the substring check the title is calling out. Walking up from `/tmp/project-sibling/nested/file.ts` with `root = "/tmp/project"`, every parent of the file *does* start with `"/tmp/project"` as a substring, so the walker happily attached AGENTS.md from the sibling directory.

## What changed

Three layered changes:

1. **`instruction.ts:201`** — replace the substring check with `AppFileSystem.contains(root, current)`. This is the actual fix for the reported bug and is one line.

2. **`packages/core/src/filesystem.ts:227-247` and `packages/opencode/src/util/filesystem.ts:158-180`** — both copies of `contains`/`overlaps` (the duplication is itself a smell, see Risks) gain a small helper layer:
   - `isWindowsAbsolutePath(p)` — `/^[A-Za-z]:[\\/]/.test(p) || p.startsWith("\\\\")` covers drive-letter and UNC.
   - `relativePath(from, to)` — picks `win32.relative` when either operand is a Windows absolute path; falls back to platform `relative` otherwise. This is the right call: a POSIX `relative` on `"C:\\a"` vs `"D:\\a"` returns gibberish.
   - `isParentRelativePath(p)` — accepts both `../` and `..\\` separators.
   - `isContainedRelativePath(p)` — `!p || (!isParentRelativePath(p) && !win32.isAbsolute(p))`. The `!p` arm preserves the "same path is contained in itself" invariant; the `!win32.isAbsolute(p)` arm catches the cross-drive case where `relativePath("C:\\a", "D:\\a\\b")` returns an absolute `"D:\\a\\b"` instead of a `..`-prefixed one.

3. **Tests** added across three files:
   - `packages/core/test/filesystem/filesystem.test.ts:328-344` — covers `..c` (a name that *literally* starts with `..` but isn't a parent traversal), Windows drive containment, cross-drive non-containment.
   - `packages/opencode/test/file/path-traversal.test.ts:38-44` — Windows mirrors of the existing POSIX sibling-prefix tests.
   - `packages/opencode/test/session/instruction.test.ts:112-138` — the integration-level regression: `tmp/project/src/file.ts` as the active file with `tmp/project-sibling/AGENTS.md` present should produce `results === []`. This is the test that pins the bug shut at the layer the user actually sees it.

## Design analysis

The decomposition is good. `isContainedRelativePath` is the single small predicate that captures the actual semantics ("relative path stays inside or at the original anchor"), and the win32-vs-posix routing is pushed all the way down into `relativePath` so the call sites stay readable. The `..c` test in particular is the kind of edge case authors miss — a name like `..config` is a perfectly legal child name that the naive `startsWith("..")` check would have rejected.

Two structural observations:

1. **Duplicated definitions.** `packages/core/src/filesystem.ts` and `packages/opencode/src/util/filesystem.ts` now contain *identical* copies of `isWindowsAbsolutePath`, `relativePath`, `isParentRelativePath`, `isContainedRelativePath`, `overlaps`, and `contains`. The duplication predates this PR (the PR keeps both copies in sync) but the next drift will reintroduce the bug in whichever copy gets edited in isolation. Worth a follow-up to consolidate one onto the other (probably `core` exporting and `opencode/util` re-exporting).

2. **The `instruction.ts` walker still has a structural smell.** The walk is `current = path.dirname(current)` until either `current === root` or `current` escapes `root`. The new condition `AppFileSystem.contains(root, current)` is correct, but on Windows the `dirname` chain eventually returns the same drive root (`"C:\\"` ⇒ `dirname` = `"C:\\"`) — an infinite loop unless the `current === root` check or `contains` short-circuits first. The integration test only exercises POSIX paths via `tmpdir`. A sanity test on Windows with a file located at e.g. `D:\file.ts` (not under any project root) would close that gap.

## Risks

- **Cross-platform `path` import vs `win32` mixing.** `relativePath` uses `win32.relative` for Windows-shaped inputs and the platform-default `relative` otherwise. On macOS/Linux that's POSIX, fine. On a Windows host with POSIX-style path inputs (e.g. an LSP result that uses forward slashes), the platform default *is* win32. The switch in `relativePath` is `if (isWindowsAbsolutePath(from) || isWindowsAbsolutePath(to)) return win32.relative(from, to)` — so a fully-POSIX-style relative call on Windows falls through to `path.relative` which is `win32.relative` anyway. Behaviorally consistent. Worth a comment confirming the intent.
- **`overlaps` semantics.** Old: `!relA || !relA.startsWith("..") || !relB || !relB.startsWith("..")`. The OR-chain was *too permissive* — if either of `relA`/`relB` was empty (same path) it returned true (correct), but it also returned true if either was *not* a `..` path (wrong for true siblings, but accidentally still correct for *this particular* bug because the asymmetric direction passed). New: `isContainedRelativePath(relativePath(a,b)) || isContainedRelativePath(relativePath(b,a))` is structurally cleaner and now correct symmetrically. No callers should regress, but worth running grep over the codebase to confirm no caller was relying on the old loose semantics.

## Verdict

**Verdict:** merge-after-nits

Correct fix, correct decomposition, correct test layering (unit + integration). Nits:
- Add a TODO/follow-up to dedupe the `core/src/filesystem.ts` and `opencode/src/util/filesystem.ts` copies onto a single source.
- Confirm the Windows-host POSIX-input fallthrough behavior with a comment near `relativePath`.
- Optional: a one-liner sanity test that walking from `D:\file.ts` with project root `C:\project` exits the `instruction.ts:201` loop (terminating-on-`current === root` covers it but pinning it in a test is cheap).

---

*Reviewed by drip-156.*
