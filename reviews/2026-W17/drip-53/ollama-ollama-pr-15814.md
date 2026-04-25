# ollama/ollama#15814 — Model support for batching
**SHA**: `7f279fc2` · **Author**: jessegross · **Size**: +2702/-426 across 25 files

## What
Introduces the `x/mlxrunner/batch` package as the canonical
forward-pass input for MLX-backed models, plus the supporting
machinery for multi-sequence batched inference. The new `Batch`
struct carries `InputIDs *mlx.Array`, per-row `SeqOffsets`, per-row
`SeqQueryLens`, and an opaque `CacheState` scratch area owned by the
nn/cache layer. Models receive a `Batch` instead of disconnected
positional args; the cache layer uses `CacheState` to dedup mask-tensor
construction across layers whose masks end up structurally identical
during one forward pass.

## Specific cite
- `x/mlxrunner/batch/batch.go:7-26` defines `Batch`. The doc on
  `SeqQueryLens` ("Values less than L mean the row's tail is padding
  that the cache must mask out") is the load-bearing contract — it
  shifts mask-management responsibility from the model into the cache.
  Models becoming oblivious to padding is the whole point.
- `x/mlxrunner/batch/cachestate.go:11-32` is the scratch type. The
  `Get`/`Put` API on `*CacheState` with lazy map allocation and a
  nil-receiver-safe `Get` is reasonable for a per-forward-pass
  ephemeral. The doc claim "no explicit reset needed" relies on
  `*Batch` being short-lived and unique per forward — worth a comment
  in the runner loop pinning that invariant.
- `cachestate_test.go:16-31` exercises the zero-value path
  (`TestCacheStateZeroGet`) — important because the doc explicitly
  promises the zero value is usable.
- The diff is large (+2702/-426, 25 files) and the surfaced snippets
  are only the new `batch` package; a full review needs to follow the
  call sites in `x/mlxrunner/model/*` and the cache implementations
  to confirm key collisions in `CacheState` are impossible across
  layers (the keys are `any`, so two layers using the same constant
  literal as a key would silently share state).

## Verdict: needs-discussion
Excellent abstraction at the package boundary, but a 2.7k-line PR
touching the runner core deserves a design walk-through. Two open
questions: (1) the `CacheState` key namespace is `any` — should it
be a typed key (e.g. unexported struct per consumer) to make
cross-layer collisions impossible by construction? (2) confirm the
`*Batch` lifetime invariant the GC story depends on is actually
enforced by the runner, ideally with a race/leak test.
