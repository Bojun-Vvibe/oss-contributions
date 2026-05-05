# openai/codex #21120 — Tighten marketplace root removal

- **Repo**: openai/codex
- **PR**: [#21120](https://github.com/openai/codex/pull/21120)
- **Author**: xli-oai
- **Head SHA**: `a5ca8015c7d01c49693f26d3a478cbb54957cd2b`
- **Base branch**: `main`
- **Created**: 2026-05-05T01:24:44Z

## Verdict

`merge-after-nits`

## Summary of change

Hardens `remove_marketplace_root` in `codex-rs/core-plugins/src/marketplace_remove.rs`:

1. `marketplace_remove.rs:77-82`: `root.exists()` → `root.try_exists()`. The old
   call swallowed I/O errors as "false" (file doesn't exist), so a permission
   error or transient disk failure on the parent directory would silently
   leave the install as "already gone" and the remove call would return
   `Ok(None)`. The new call surfaces the underlying error as
   `MarketplaceRemoveError::Internal` with the path and the io::Error embedded.
2. `marketplace_remove.rs:97-107`: dispatches on `metadata.file_type()` instead
   of just `metadata.is_dir()`. Directories use `remove_dir_all`, regular files
   use `remove_file`, and anything else (symlinks-to-nothing, sockets, fifos,
   block/char devices, etc.) is rejected with a clear
   `"installed marketplace root {} is neither a file nor a directory"` error
   instead of being silently routed to `remove_file` and possibly succeeding
   with the wrong semantics.

## What's good

- `try_exists` vs `exists` is exactly the right tightening — `exists()`
  collapsing I/O errors to `false` is a documented Rust footgun and the only
  correct call when the answer materially matters (here it does: it gates a
  `remove_dir_all`).
- The error message includes `root.display()`, so the caller / log will
  identify the offending path. Consistent with other error-return sites in the
  same file.
- The new "neither file nor directory" branch is a `return Err(...)` with an
  explicit format — does not delegate to `remove_file` and silently succeed on
  a symlink (which the previous `else` branch effectively did, since
  `metadata` follows symlinks but `remove_file` operates on the link name).
- Self-contained 11-line change, easy to backport.

## Nits / questions

- **Symlink semantics are still subtle**: `fs::metadata(root)` on the upstream
  call (visible in the surrounding context just above the diff hunk on
  `:90-95`) follows symlinks. So a symlink pointing to a directory will be
  treated as a directory and `remove_dir_all(root)` is called — but
  `remove_dir_all` operates on the link name, will see the link is not a
  directory at the path level, and will fail with `ENOTDIR`. Likewise a
  symlink to a regular file will hit the `is_file()` branch and `remove_file`
  will unlink the symlink (not the target) — which is probably what's wanted,
  but worth a sentence in the doc comment. Consider switching to
  `fs::symlink_metadata` and routing symlinks explicitly to `remove_file` to
  unlink the link itself.
- **`Internal` for "wrong file type" feels slightly off**: the `Internal`
  variant typically means "we did something wrong"; "your install is corrupt
  in an unexpected way" is closer to a user/data error. If
  `MarketplaceRemoveError` has a `CorruptInstall` or `Unexpected` variant,
  prefer that. If not, ignore — wording matters less than the actionable
  message, which is good.
- No test added. The `try_exists` path is hard to unit-test portably (you'd
  need to set up a directory the test process can't read), but the
  "neither-file-nor-directory" branch can be tested cheaply on Unix with a
  fifo or symlink-to-nothing — worth a `#[cfg(unix)]` test.

## Risk

Low. Both changes are strictly tightening: previously-`Ok(None)` paths that
masked real errors now surface them; previously-`Ok(())` paths that silently
ran `remove_file` on non-files now return a structured error.

## Recommendation

Land after a 1–2 line clarification in the doc comment about symlink behavior
(or a switch to `symlink_metadata`), and ideally one Unix-only test for the
new error branch.
