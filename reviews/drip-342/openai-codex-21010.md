# openai/codex #21010 — memories-mcp: reject symlink traversal in local backend

- **Head SHA:** `8b0f758a5e9afdd6cf25412de2b946305aef6c82`
- **Size:** +155 / -18 across 4 files
- **Verdict:** **merge-as-is**

## Summary
Hardens `LocalMemoriesBackend::resolve_scoped_path`
(`codex-rs/memories/mcp/src/local.rs:42`) so it refuses to follow symlinks at
**any** ancestor component of the requested path, not just the leaf. Walks
each component in order, calls `metadata_or_none` and `reject_symlink` per
step, and bails with `InvalidPath` if a non-trailing component is a
non-directory (`local.rs:67-77`). Adds a new `MemoriesBackendError::NotFound`
variant (`backend.rs:121`) and changes `list`/`search` on missing scoped paths
from "return empty result" to "return NotFound" — a behavior change but the
right one for the security goal. Three new tests cover missing paths and a
fourth (`read_rejects_symlinked_ancestor_directories`) reproduces the
specific traversal-via-symlinked-dir attack.

## Strengths
- The component-walk loop (`local.rs:62-79`) is the textbook fix for "the leaf
  is fine but `a/b -> /etc` lets you read `a/b/passwd`". `reject_symlink` is
  invoked on every prefix, which closes the gap.
- The "non-directory in the middle of the path" guard at lines 73-78 catches
  the related case where a regular file appears mid-path; previously the OS
  would surface a confusing ENOTDIR, now it surfaces a typed
  `MemoriesBackendError::InvalidPath` with a clear reason.
- The change from "empty result on missing path" to `NotFound` is correct
  semantics. Returning empty silently masks typos and (more importantly here)
  makes it harder to detect when an attacker is probing for paths that escape
  the sandbox.
- Test coverage lands the right cases:
  `read_rejects_missing_paths` (line 273), `list_rejects_missing_scoped_paths`
  (line 742), `search_rejects_missing_scoped_paths` (line 757), and the
  unix-gated `read_rejects_symlinked_ancestor_directories` at line 811. The
  ancestor-symlink test specifically points the symlink at an *outside*
  directory, which is exactly the threat model.
- `resolve_scoped_path` becoming `async` is propagated cleanly through every
  call site (`local.rs:103, 177, 220`) — no missed `.await`s I can spot in
  the diff.

## Nits
- `request.path.unwrap_or_default()` in the `list`/`search` `NotFound` arms
  (`local.rs:111-113, 229-231`) will report `path: ""` when the caller passed
  no path at all. Cosmetic — but a `path: "<root>"` placeholder would read
  better in error logs.
- `metadata_or_none` is called on every component, so a deep path does N
  syscalls instead of 1. For the memories use case the paths are short, so
  this is fine; worth a comment noting the deliberate trade-off vs. a single
  `canonicalize` (which would defeat the purpose).

## Recommendation
Ship it. This closes a real symlink-traversal hole, the tests match the
threat model, and the API change (NotFound vs empty) is an improvement.
