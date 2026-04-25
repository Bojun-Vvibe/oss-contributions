# cline/cline #10376 — fix: prevent path traversal in ClineIgnoreController.validateAccess

- **Repo**: cline/cline
- **PR**: [#10376](https://github.com/cline/cline/pull/10376)
- **Head SHA**: `9b369e1e0295bf96cb4af8998a292e15deff3feb`
- **Author**: JasonOA888
- **State**: OPEN (+166 / -35, mostly package-lock.json `peer: true` churn)
- **Verdict**: `merge-after-nits`

## Context

`ClineIgnoreController.validateAccess()` is the gate for `read_file` /
`write_to_file` / `apply_patch` against the agent's `.clineignore`
allow/deny list. Two pre-existing flaws turned this gate into a no-op
for any path outside the workspace:

1. The early-return `if (!this.clineIgnoreContent) return true` meant
   that **the default project (no `.clineignore`)** allowed every path
   the agent could synthesize — including `../../etc/passwd`.

2. The `catch` block returned `true` with the literal comment
   "*We are allowing access to all files outside cwd.*" — when
   `path.relative()` threw or returned a `..`-prefixed path, the
   controller silently allowed it.

In auto-approve / yolo mode this composes into a clean filesystem
read+write primitive against any path the agent process can reach.
That's a real, exploitable issue, not a theoretical one.

## Design

The fix at `src/core/ignore/ClineIgnoreController.ts:163-181` reorders
the logic correctly:

```ts
const absolutePath = path.resolve(this.cwd, filePath)
const relativePath = path.relative(this.cwd, absolutePath)

// Block paths that resolve outside the workspace (path traversal)
if (relativePath.startsWith("..") || path.isAbsolute(relativePath)) {
    return false
}

// Always allow access if .clineignore does not exist
if (!this.clineIgnoreContent) {
    return true
}

return !this.ignoreInstance.ignores(relativePath.toPosix())
```

Boundary check first, allow-list second, `ignore` library third. The
`relativePath.startsWith("..")` + `path.isAbsolute(relativePath)`
combination is the standard Node.js workspace-prefix pattern and
correctly catches:

- `../../etc/passwd` → relative path starts with `..` → deny ✓
- `/etc/passwd` → on POSIX, `path.relative("/work", "/etc/passwd")`
  yields `../etc/passwd` → starts with `..` → deny ✓
- on Windows, `path.relative("C:\\work", "D:\\evil")` returns an
  absolute path → `path.isAbsolute` catches it → deny ✓
- in-workspace paths → relative neither absolute nor `..`-prefixed →
  proceed to allow-list ✓

The `catch` block now returns `false` (deny by default), which is the
correct posture for a security gate.

The companion fix at line 137-148 (`readIncludedFile`) closes a
secondary hole I'd missed reading the description first: `.clineignore`
supports `!include other-file`, and the previous `path.join(this.cwd,
includePath)` made `!include ../../etc/passwd` a way to slurp arbitrary
files into the ignore-pattern set (and from there into the agent's
worldview). Now resolved with `path.resolve` + the same boundary
check.

## Test coverage

Four new test blocks added — collected from the diff:

- `Include Directive Path Traversal Protection` (3 cases at the
  `extra-ignore.txt` happy path + two traversal denies). Verifies the
  `!include` fix at line 137-148 actually rejects `../../etc/passwd`
  and `/etc/shadow` while still allowing in-workspace includes.
- `Path Traversal Protection` (4 cases) covers `..`-traversal,
  absolute-path traversal, in-workspace allow, and the
  no-`.clineignore` default-deny. Last one is the specific regression
  for the most exploitable variant.

The "block traversal when .clineignore does not exist" case is the
most important one — that's the default state for ~90% of repos and
was the wide-open path before this PR.

## Risks / Nits

1. **`path.resolve` is still lexical.** A symlink inside the workspace
   pointing to `/etc` lets the agent write to `/etc/passwd` via
   `workspace/link/passwd`, because `path.relative` only compares
   normalized strings. Same hole as crush #2699 in this drip — worth
   a defense-in-depth `fs.realpathSync` pass before the prefix check,
   gated behind a try/catch (a non-existent file path can't realpath).
   Not a blocker because (a) most LLM-generated paths won't hit this
   and (b) the symlink has to already exist in the workspace, which is
   a much narrower threat model.

2. **`relativePath.toPosix()` postfix on the `ignores()` call** —
   `path.relative` on Windows returns backslashes; `ignore` library
   wants forward slashes. The cast is correct, but the boundary check
   on line 169 runs against the raw (possibly-backslash) `relativePath`
   *before* `.toPosix()` — which works because both `..` and `path.isAbsolute`
   are slash-agnostic, but it's worth a one-line comment so future
   readers don't "fix" it by moving `.toPosix()` upward and breaking
   Windows.

3. **package-lock.json noise** — 130 lines of `"peer": true` additions
   are unrelated to this fix and should ideally be split into a chore
   PR. Not a merge blocker but it makes the diff harder to review and
   risks a security-PR being held up on dependency-review concerns.

4. **Logger.debug on the `!include` block path** at line 144 is fine
   (defensive logging, no PII). Good.

## Verdict rationale

`merge-after-nits`. Correctness is right, the test cases lock down the
exact regression, and the fix is structurally sound. Two follow-ups
would make me happier — symlink resolution and splitting the
lockfile churn — but neither is a reason to hold the security fix.

## What I learned

The "we are allowing access to all files outside cwd" comment in the
original code is a striking artifact of a fail-open security control
that was *documented* as fail-open and survived review. Worth keeping
this PR around as a teaching example for "deny by default in catch
blocks of security gates."
