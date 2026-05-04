# charmbracelet/crush #2613 — fix(agent): prune excess images from history to prevent session deadlock

- SHA: `8ca4435b8e302434a7a15371c2a213bac84f5193`
- State: OPEN, +775/-0 across 2 files (`internal/agent/agent.go`, `internal/agent/prune_images_test.go`)

## Summary

Closes #2604. When conversation image count exceeds a provider's per-request cap (Gemini/VertexAI = 10), the session enters an unrecoverable loop because every subsequent request re-sends the same oversized history. PR adds `pruneExcessImages()` in the `PrepareStep` that strips the *oldest* image `FilePart`s, replacing each with a short text placeholder, leaving database-persisted messages untouched.

## Notes

- `internal/agent/agent.go:1213-1226` — `maxImagesForModel` is a hard-coded switch on `catwalk.InferenceProvider(model.ModelCfg.Provider)` returning 10 for Gemini/VertexAI, 0 (unlimited) otherwise. The TODO to migrate to `catwalk.Model.MaxAttachments` is the right long-term direction. **However**, the PR body advertises a user-configurable `max_images` field in `crush.json` ("two-tier priority: user config first, then provider default"), but the diff contains **no** changes to `internal/config/config.go` and `maxImagesForModel` never reads `model.ModelCfg.MaxImages`. Either the feature description is stale or a commit is missing — must reconcile before merge.
- `internal/agent/agent.go:1257-1296` — pruning logic walks messages in original order and replaces the first `toRemove` image parts with `TextPart{Text: "[Earlier image %q removed to stay within model limits]"}` carrying the original filename. Correct strategy (oldest-first preserves recency relevance). Side effects on the message struct construction at line 1289-1293 correctly preserve `Role` and `ProviderOptions`. One concern: the placeholder is a *new* `TextPart` inserted at the image's original position, which can produce two adjacent text parts in the same message — some providers normalize/concatenate this, others reject. Worth a `mergeAdjacentText` pass or an inline note.
- `internal/agent/agent.go:1299-1303` — `slog.Info` on every prune is correct for observability, but on a long-running session that hits this every turn, it becomes log spam. Consider `slog.Debug` or rate-limiting on the same `(removed, original_count)` tuple.
- `internal/agent/agent.go:288` — call site is well-placed: immediately after `workaroundProviderMediaLimitations` and before the system-message rewrite. This guarantees provider-specific media transforms run first, then the count-based cap, then any subsequent message manipulation. Good ordering.
- `internal/agent/prune_images_test.go` — 666 lines, 25+ test cases covering `maxImagesForModel`, `isImageFilePart`, `countImagesInMessages`, the prune itself, and an end-to-end mock model that enforces the 10-image cap. Coverage looks thorough. **But** because there is no actual `MaxImages` config field in the diff, none of these tests exercise the user-override path — confirms the feature claim is missing implementation.
- Nit: `maxImagesForModel` ignores `model.CatwalkCfg.SupportsImages`. A non-image-supporting model with images in history will skip pruning (returns 0 = unlimited) and rely on the provider to error out. Probably fine — that's pre-existing behavior — but worth documenting.

## Verdict

`request-changes` — the prune logic and tests are well-built and the bug it fixes is real and unrecoverable, but the PR body advertises a `max_images` config-override feature that is **not in the diff**. Either add the `SelectedModel.MaxImages` field + the config-precedence branch in `maxImagesForModel`, or strip the override claim from the description. Also fold adjacent text parts after pruning to avoid surprising providers that reject consecutive text parts in a single message.
