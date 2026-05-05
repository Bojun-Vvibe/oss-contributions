# openai/codex PR #21184 — Clean Windows metadata sentinels before handle close

- Head SHA: `9f298583f2bbc09b6b9456386808c7c7c3306439`
- Author: `evawong-oai`
- Size: +1 / -1
- Verdict: **merge-as-is**

## Summary

One-line ordering fix in
`codex-rs/windows-sandbox-rs/src/protected_metadata.rs:78` that moves
`self.sentinel_handles.clear()` from *before* the `for path in
…sentinel_paths.iter()` removal loop to *after* it. The bug: dropping
the open `HANDLE`s held in `sentinel_handles` (which is what
`Vec::clear()` does) before issuing the `DeleteFile`/`unlink`-style
removal in the loop means the loop tries to delete sentinel files
whose protective open handle has just been released — racing any
other process that might have grabbed the now-deletable file, and on
Windows specifically losing the `FILE_FLAG_DELETE_ON_CLOSE` /
share-mode protection that the sentinel handle was providing.

## What the diff actually does

```
-        self.sentinel_handles.clear();
         let mut removed = Vec::new();
         for path in self.monitored_paths.iter().chain(self.sentinel_paths.iter()) {
             …
             removed.push(existing_path);
         }
+        self.sentinel_handles.clear();
```

The cleanup ordering invariant the PR establishes: **handles outlive
the path removal**. While the loop runs, the open handles continue to
hold the per-path locks/share-modes that the rest of the
`ProtectedMetadataGuard` design relies on. Only after every path the
guard is responsible for has been removed (or skipped via
`existing_metadata_path` returning `None`) do we drop the handles via
`clear()`. This is the canonical "release locks last" RAII pattern.

## Why merge-as-is

- Minimal, surgical: 1 line moved, no semantic refactor, no API
  change, no allocation change. `removed` Vec returned to caller is
  bit-identical.
- The reordering is provably correct: the loop body at lines
  79-87 calls `existing_metadata_path(path)?` and a remove call,
  neither of which mutates `sentinel_handles`, so `clear()`-after-
  loop and `clear()`-before-loop produce identical
  `sentinel_handles` post-state. The only observable difference is
  the lifetime overlap between handle ownership and remove
  syscalls, which is the entire point of the fix.
- `cleanup_created_paths` is `pub(crate)` and the only caller is
  intra-crate guard cleanup, so there's no external contract to
  worry about.
- The PR author (evawong-oai) is the same author landing the larger
  Windows missing-metadata sentinel monitor stack (#21172, #21173,
  #21174 in the same drip window), so this is a follow-up shake-out
  fix from the integration of that subsystem — fits the pattern of
  "land big mechanism, then tighten the invariants".

## Optional nits (not blocking)

- A regression test that opens a sentinel handle, calls
  `cleanup_created_paths`, and asserts via a Windows-specific harness
  that the file deletion succeeded *without* a transient
  share-violation window would be ideal but is genuinely hard to
  write without flakiness. The 1-line ordering invariant is
  obvious-on-inspection enough that I'd accept it without a test.
- A short doc-comment on the function explaining the ordering
  invariant ("handles must outlive the removal loop on Windows
  because…") would help a future contributor avoid re-introducing
  the bug. Pure documentation.
- No changelog/AGENTS.md note required for an internal cleanup
  ordering fix.
