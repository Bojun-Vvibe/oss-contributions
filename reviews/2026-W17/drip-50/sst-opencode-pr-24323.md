---
pr: 24323
repo: sst/opencode
sha: c146a7eb0947329d35a5ea3cae47664a4a65a69d
verdict: merge-after-nits
date: 2026-04-25
---

# sst/opencode#24323 — fix(editor): reject lock files with no workspace match for cwd

- **URL**: https://github.com/sst/opencode/pull/24323
- **Author**: jjjermiah
- **Closes**: #24295

## Summary

Without a minimum-match threshold, `resolveEditorLockFile()` would pick
*any* VSCode IDE lock file in `~/.claude/ide/` and connect to it,
even when none of that lock's `workspaceFolders` had any relation to
the current working directory. Concretely: a Wezterm session in
`~/code/proj-A` could end up receiving `selection_changed` events from
an unrelated VSCode window open in `~/code/proj-B`, because the old
`scoreEditorLock()` collapsed "any match" to a 1/0 bit and then
fell back to `mtime` — so the most-recently-touched IDE always won
even with zero workspace overlap.

## Reviewable points

- `packages/opencode/src/cli/cmd/tui/context/editor.ts:255` — new
  `bestMatchLength()` helper computes
  `Math.max(0, ...lock.workspaceFolders.map((folder) => pathContainsLength(folder, cwd)))`.
  This is the right primitive: it keeps the
  "longest-prefix-wins" semantics needed when a user has nested
  workspaces (e.g. `~/code` and `~/code/sub` both open) — `~/code/sub`
  should win when `cwd === ~/code/sub/x`.

- Line ~261 adds `.filter((entry) => bestMatchLength(entry) > 0)` — the
  hard cutoff. Without this, a stale lock with mtime-from-now would
  still beat a closed-IDE one that actually matches the cwd. The
  filter is the actual bug fix; the longest-match sort is the polish.

- `pathContainsLength()` (lines 291–295) returns `resolved.length` on
  match, `0` on miss. One small concern: `resolved.length` ranks by
  *resolved-path string length*, not directory depth. On macOS,
  `/Users/foo/x` (12 chars) vs `/private/tmp/xx` (15 chars) — the
  symlink-vs-realpath difference will skew the ranking. In practice
  workspace folders for a given user will share a prefix so this
  rarely matters, but a comment explaining "we're comparing
  *resolved-path-string lengths*, not segment counts" would save a
  future reader 5 minutes.

- The `.sort()` tiebreaker at line ~263 falls back to
  `right.mtimeMs - left.mtimeMs` only when the best-match lengths are
  *equal*. That's exactly right and a behaviour-preserving carry-over
  from the old `scoreEditorLock` (where mtime was the low-order
  bits). No regression in the "two valid IDEs, pick the most recently
  active one" case.

- The old `scoreEditorLock`/`pathContains` pair is fully deleted, not
  just unused. Good — leaving them would invite a future contributor
  to reach for the old binary helper.

## Rationale

Real cross-process privacy/correctness bug with a clean fix. The
implementation is sound and the only nits are documentary: (a) note
that `pathContainsLength` ranks by string length not depth, (b)
ideally a regression test under
`packages/opencode/test/cli/...editor...` that fixtures two lock
files (one matching, one mtime-newer-but-non-matching) and asserts
the matching one wins. PR ships only the fix, no test added — for a
function that controls cross-IDE event routing, locking the
"unrelated lock is not picked" property in CI seems worth doing
before merge.

## What I learned

Composite scoring functions (`workspaceMatch * 1e12 + mtimeMs`) that
encode a hard-cutoff predicate as a bit-shifted weight are a known
trap: they look like they implement "match wins, tiebreak on mtime"
but actually implement "the highest score wins, and a 0-score lock
can still be selected if it's the only one". The right shape is
*filter then sort*, not *score-as-numeric-bitfield*. The cleanup also
removed an unintended ordering quirk — under the old scheme a lock
with `mtimeMs > 1e12` (any timestamp after Sept 2001) could in
theory shift workspaceMatch's "1" out of significance via floating
point precision, though in practice f64 had enough mantissa. Easier
to reason about with explicit filter + explicit comparator.
