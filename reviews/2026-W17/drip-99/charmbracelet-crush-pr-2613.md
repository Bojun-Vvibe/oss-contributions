# charmbracelet/crush#2613 — fix(agent): prune excess images from history to prevent session deadlock

- **Head**: `8ca4435b8e302434a7a15371c2a213bac84f5193`
- **Size**: +775/-0 across 2 files
- **Verdict**: `merge-after-nits`

## Context

Fixes a hard session-deadlock bug for image-capable Gemini / Vertex AI
sessions. Gemini's API enforces a per-request limit of 10 images. Crush
sends the full conversation history with every request, so once a session
accumulates 11+ images cumulatively, **every subsequent request fails
with the same "too many images" error** — even text-only follow-up
prompts. The session is permanently bricked from the user's POV; the
only recovery is to start a new session, losing all context.

The fix prunes the oldest images from the request payload (replacing
each with a short `[Earlier image "..." removed to stay within model limits]`
text placeholder) when the count exceeds the provider limit, while
leaving the persisted DB messages untouched.

## Design analysis

### Insertion point: `internal/agent/agent.go:288`

```go
prepared.Messages = a.workaroundProviderMediaLimitations(prepared.Messages, largeModel)
+ prepared.Messages = pruneExcessImages(prepared.Messages, largeModel)
```

Correct location — runs after the existing media-limitation workaround
(which handles per-message image-count caps) but before any other
serialization. The pruning sees the same already-normalized message
array that the provider client will serialize, so there's no risk of
counting an image that gets dropped later or vice-versa.

Critically, the pruning happens on `prepared.Messages` (the in-memory
copy for this request), not on the persisted session messages. Re-reading
the function shows no assignment back to a persisted store; only the
local slice is rebuilt. This is the right invariant: the user's history
is not lossily edited by transient provider limits.

### Provider-limit table: `agent.go:1213-1221`

```go
func maxImagesForModel(model Model) int {
    switch catwalk.InferenceProvider(model.ModelCfg.Provider) {
    case catwalk.InferenceProviderGemini, catwalk.InferenceProviderVertexAI:
        return 10
    default:
        return 0
    }
}
```

Hardcoded `10` for Gemini + Vertex; `0` (sentinel for "unlimited") for
everything else. The TODO comment at `:1209-1211` flags the right
follow-up: push this into `catwalk.Model` as a `MaxAttachments` field
so the limit isn't duplicated client-side. For now, hardcoding is fine
because the limit is documented and stable on Google's side.

The Anthropic / OpenAI tests at `prune_images_test.go:118-130` correctly
assert `0` for those providers — i.e. the function does nothing for them.
That's the safe default: if a provider isn't on the explicit list, the
prune step is a no-op.

### Pruning algorithm: `agent.go:1235-1283`

Walks messages in order; for each one, walks parts in order; replaces
the first `toRemove` image parts with a placeholder TextPart. Stops as
soon as `removed >= toRemove`. Three things to verify here:

1. **Oldest-first** — yes. The outer `for _, msg := range messages` walks
   in insertion order (oldest message first), and `msg.Content` is also
   walked in order. So the first images the user uploaded are pruned
   first, which is the right policy: recent images are more likely to be
   referenced by the current turn.
2. **No mutation of original messages** — the code builds `newParts` and
   `result` slices fresh; original `fantasy.Message` structs are never
   modified. ✓
3. **Provider options preserved** — line 1276 explicitly carries
   `ProviderOptions: msg.ProviderOptions` onto the rebuilt message. ✓

One subtle correctness check: the placeholder is a `TextPart`, not a
zero-byte image part. That matters because some providers reject empty
or zero-byte image parts with their own error. Substituting text avoids
that class of failure.

### Telemetry: `agent.go:1276-1282`

```go
if removed > 0 {
    slog.Info("Pruned excess images from conversation history",
        "removed", removed,
        "max_allowed", maxImages,
        "original_count", total,
    )
}
```

Logs only when something happened — good. The user-visible signal is
the `[Earlier image "..." removed ...]` placeholder text, which the
model will see and can mention in its response if relevant. That's
actually a reasonable UX: the model can say "I no longer have access
to the first three screenshots you shared" instead of silently
hallucinating.

### Test coverage: `prune_images_test.go:1-666`

666 lines is heavy for a single feature, but the cases are well-chosen:

- `TestMaxImagesForModel_*` — provider table coverage (Gemini, VertexAI,
  Anthropic, OpenAI).
- `TestIsImageFilePart_*` — content-type discrimination
  (`image/png`, `text/plain` filepart, `TextPart`).
- `TestCountImagesInMessages_*` — empty / no-images / mixed-content
  counting, plus the cumulative-across-messages case.
- `TestPruneExcessImages_*` — no-limit-provider passthrough, under-limit
  passthrough, at-limit edge.

The diff truncates before showing the over-limit cases, but the file
length suggests they exist. Worth confirming there's an explicit
"exactly one over limit" test (where `total = maxImages + 1` and
`removed` should be 1) and a "user adds 5 images in one turn after
already at limit" test (where the new images themselves should not be
the ones pruned, since they're the most recent).

### `max_images` config field

PR body claims a user-overridable `max_images` field in `crush.json`,
but the diff hunks I can see don't show it being read. If it's wired
through `model.ModelCfg.MaxImages` or similar, that needs a code path
that takes precedence over `maxImagesForModel(model)`. If a user sets
`max_images: 20` for Gemini, the override should win — but with the
current `maxImagesForModel` returning a hardcoded `10`, it currently
won't.

This is the main concern. The config-precedence path is either present
in a part of the diff I haven't seen, or it's a body-vs-code
discrepancy. Either way, worth confirming the config override is wired
end-to-end with a test.

## Concerns / nits

1. **Verify `max_images` config override actually wins** over the
   hardcoded provider table. PR body lists it as a feature; the visible
   diff doesn't show it. This is the only material concern.
2. **666-line test file for ~100 lines of production code** is a high
   ratio. Most of it is justified (table-driven cases for each
   provider × each scenario), but a few of the
   `TestMaxImagesForModel_OpenAI` style single-assertion tests could be
   collapsed into one parameterized case to reduce maintenance surface.
3. **Placeholder text** `[Earlier image %q removed to stay within model limits]`
   includes the original filename. If the filename contains user-private
   info (paths, dates), it's now visible to the model when previously
   only the image bytes were sent. Minor privacy nit; consider
   `[Earlier image removed ...]` without the name, or gate the name
   behind a debug flag.
4. **No explicit interaction test with `workaroundProviderMediaLimitations`**.
   Both run on the same `prepared.Messages` in sequence; if the
   workaround already drops some images, the prune step's `total` count
   will see the post-workaround count, which is correct — but worth
   pinning with a test.
5. **`slog.Info` is unconditional** when `removed > 0`. For long
   sessions on Gemini, every request after the threshold will log this.
   Consider downgrading to `slog.Debug` after the first occurrence per
   session, or include a session-id key so log aggregation can dedupe.

## Verdict reasoning

The core algorithm and integration point are correct, and the
no-mutation-of-persisted-history invariant is preserved. The verdict
drops to `merge-after-nits` solely on the `max_images` config override
ambiguity — that's a feature-claim that needs end-to-end verification
before merge. The other items are cosmetic. Once the config override is
confirmed wired (or removed from the PR description if it's
out-of-scope), this is mergeable.

## What I learned

"Persisted history vs. request payload" is a useful invariant boundary
in agent loops. Many provider-limit workarounds (image cap, token cap,
tool-result media split) want to mutate the request without polluting
the user's session record. The pattern here — rebuild a fresh
`[]Message` slice for the request, never assign back to storage — is
the right shape, and worth replicating for token-budget pruners and
similar logic. The placeholder-text-instead-of-empty-part choice is
also a small but important detail: the model gets to know *that*
something was removed, which keeps it honest about its actual context.
