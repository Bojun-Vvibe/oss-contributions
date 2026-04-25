# ollama/ollama PR #15809 — create: prune imported blobs and startup invalid leftovers

- **URL:** https://github.com/ollama/ollama/pull/15809
- **Head SHA:** `e8de8d9e290ec24eeed17492f5640d5a68405ce2`
- **Files touched:** 4 (`server/create.go`, `server/images.go`, two test files)
- **Verdict:** `merge-after-nits`

## Summary

Fixes a regression where successful safetensors imports left their
uploaded source blobs on disk. Two changes:
1. `CreateHandler` defers a manifest-aware prune of request `Files` /
   `Adapters` after every create.
2. Startup blob GC now removes invalid blob filenames (e.g. `*-partial`)
   immediately, instead of waiting out the one-hour grace window.

## Specific references

- `server/create.go:103-106` — two `defer pruneCreateInputBlobs(...)` calls
  registered immediately after `oldManifest` parse, before the heavy
  conversion path. Defer order means `Adapters` is pruned before `Files`,
  which is fine because the manifest check is per-blob.
- `server/create.go:284-318` — `pruneCreateInputBlobs` builds a synthetic
  `Manifest` with the input layers and calls `m.RemoveLayers()`, which
  internally skips digests still referenced by *any* manifest. This is the
  right primitive — reuses existing safety.
- `server/create.go:289` — early-out on `envconfig.NoPrune()`. Good, keeps
  the existing user escape hatch working.
- `server/images.go:11/-12` — restart-time prune now validates blob names
  before applying the recent-blob grace period. The diff makes invalid
  partials disappear immediately on startup.
- `server/routes_create_test.go:+62` covers the safetensors cleanup path;
  `server/images_test.go:+22` covers immediate pruning of recent invalid
  blob files. Both behaviours are now regression-tested.

## Reasoning / nits

- `pruneCreateInputBlobs` swallows errors with `slog.Warn`. Acceptable for
  a deferred best-effort cleanup, but worth including the digest count in
  the log to make post-mortem debugging tractable when disk fills up.
- The `seen` map de-dupes inside one request, but if `Files` and `Adapters`
  share a digest only the first map gets it (defers run in LIFO). Confirm
  with the author that two passes with overlapping digests is fine — it
  should be, because `RemoveLayers` is idempotent and the second call's
  blob is already gone.
- Consider naming the deferred error path so it doesn't fire when the
  whole request errored before any blob was actually used (the current
  behaviour will still try to prune, which is fine but mildly wasteful).

Merge after a quick log-line nit.
