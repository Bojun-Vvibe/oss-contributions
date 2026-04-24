# ollama/ollama PR #15766 — server: populate parameter_size in /api/tags for safetensors models

- **Repo:** ollama/ollama
- **PR:** [#15766](https://github.com/ollama/ollama/pull/15766)
- **Head SHA:** `b9c487c74df9f373e1429c4bb078bd29c2c47bff`
- **Author:** nandanadileep (Nandana Dileep)
- **Size:** +81/-7 across 2 files
- **Reviewer:** Bojun (drip-24)

## Summary

Closes #15679. `ListHandler` (the `/api/tags` endpoint) only used
manifest config fields (`ModelType`, `FileType`) to populate the
`ParameterSize` and `QuantizationLevel` of each model in the
response. Those fields aren't written for local safetensors
models at creation time, so `/api/tags` returned an empty
`parameter_size` for safetensors models like `gemma4:26b-mxfp8`
and `gemma4:26b-nvfp4`, even though `/api/show` returned the
correct values for the same models (because `/api/show` already
called `xserver.GetSafetensorsLLMInfo` / `GetSafetensorsDtype`).

This PR mirrors that enrichment step into `ListHandler`. Adds a
regression test.

## Key changes

### `server/routes.go` (+27/-7)

The new enrichment block at line 1424 in `ListHandler`:

```go
modelDetails := api.ModelDetails{
    Format:            cf.ModelFormat,
    Family:            cf.ModelFamily,
    Families:          cf.ModelFamilies,
    ParameterSize:     cf.ModelType,
    QuantizationLevel: cf.FileType,
}

// For safetensors LLM models (experimental), enrich details from config.json
// to match the behaviour of /api/show for these models.
if cf.ModelFormat == "safetensors" && slices.Contains(cf.Capabilities, "completion") {
    if info, err := xserver.GetSafetensorsLLMInfo(n); err == nil {
        if arch, ok := info["general.architecture"].(string); ok && arch != "" {
            modelDetails.Family = arch
        }
        if paramCount, ok := info["general.parameter_count"].(int64); ok && paramCount > 0 {
            modelDetails.ParameterSize = format.HumanNumber(uint64(paramCount))
        }
    }
    if modelDetails.QuantizationLevel == "" {
        if dtype, err := xserver.GetSafetensorsDtype(n); err == nil && dtype != "" {
            modelDetails.QuantizationLevel = dtype
        }
    }
}
```

Three things worth noting:

1. **Gate is `cf.ModelFormat == "safetensors" &&
   slices.Contains(cf.Capabilities, "completion")`.** The
   `completion` capability check is right — embedding-only
   safetensors models don't need the LLM info enrichment, and
   `GetSafetensorsLLMInfo` would presumably return uninteresting
   data (or fail) on them. But "completion" capability is a
   stringly-typed check; if ollama ever adds a new capability
   string for, say, vision-language models that should also get
   parameter-size enrichment, this code silently skips them.

2. **Errors from `GetSafetensorsLLMInfo` are silently swallowed.**
   `if info, err := ...; err == nil` — if the call fails, the
   model still appears in the list with empty `ParameterSize`,
   no log line, no warning. For `/api/tags` semantics that's
   defensible (the endpoint shouldn't crash a list request over
   one bad model), but a `slog.Debug("safetensors LLM info
   probe failed", "model", n, "err", err)` would help debug
   why a particular model has empty size.

3. **`format.HumanNumber(uint64(paramCount))` for parameter
   count.** Matches the format `/api/show` produces (e.g.,
   `"26B"` rather than `26000000000`). Good consistency.

4. **`QuantizationLevel` fallback only if not set in manifest.**
   The `if modelDetails.QuantizationLevel == ""` check means
   manifest-provided values win. That's the right precedence
   — a manifest-config override takes priority over auto-
   detected dtype.

### `server/routes_list_test.go` (+54/-0)

`TestListSafetensorsDetails` builds a manifest with explicit
`ModelFormat: "safetensors"`, `ModelType: "26B"`,
`FileType: "mxfp8"`, `Capabilities: []string{"completion"}`,
writes it to a tempdir-rooted models dir, hits `ListHandler`,
and asserts:

- `got.ParameterSize` is non-empty (loose check),
- `got.QuantizationLevel == "mxfp8"` (exact check).

The test exercises the **manifest-provided** values, not the
**enrichment-from-config.json** path. The PR description claims
the bug is "ModelType / FileType are not written for local
safetensors models at creation time" — i.e., the actual bug
case has empty manifest values, and the enrichment fills them
in from `GetSafetensorsLLMInfo`. The test as written would pass
*without* the enrichment block, because it provides
`ModelType: "26B"` directly. **The regression isn't actually
covered.**

To pin the actual fix, the test should set
`ModelType: ""`, `FileType: ""`, and assert that
`ParameterSize` and `QuantizationLevel` are populated *from*
`config.json`. That requires either creating real safetensors
files in the tempdir or mocking `xserver.GetSafetensorsLLMInfo`.

## Concerns

1. **Test doesn't actually exercise the bug fix.** (See above.)
   The new test passes whether or not the enrichment block is
   present. It covers the manifest-fallback path correctly but
   not the `config.json`-enrichment path that this PR adds.

2. **Two fs probes per safetensors model on every `/api/tags`
   call.**

   `GetSafetensorsLLMInfo(n)` reads (and parses?) the model's
   `config.json`; `GetSafetensorsDtype(n)` similarly probes
   safetensors files. For users with many local safetensors
   models, every `/api/tags` call (which is hit on UI refresh,
   `ollama list`, etc.) now does N filesystem reads. Worth
   measuring — if any of these calls is slow (e.g.,
   `GetSafetensorsDtype` opens a multi-GB safetensors file),
   `/api/tags` latency degrades linearly.

   A small in-memory cache keyed on `(n, mtime)` would close
   this — the model metadata is immutable for the lifetime of a
   given digest, so the cache can be unbounded with no
   eviction.

3. **Type assertions on map values.**

   ```go
   info["general.architecture"].(string)
   info["general.parameter_count"].(int64)
   ```

   The comma-ok form (`v, ok := info[...]`) is used, which
   handles a missing key safely. But the `int64` assertion
   assumes the gguf/safetensors metadata loader always returns
   parameter counts as `int64`. If a future change starts
   returning `uint64` (which would be more correct since
   parameter counts are non-negative), the comma-ok returns
   `false` and the field stays empty silently. Probably fine,
   but worth a `slog.Debug` on the failure case.

4. **`modelDetails` variable lives outside the `for` loop scope
   in the diff context.**

   Need to verify the variable is declared *inside* the loop
   body, otherwise the `if` block's mutations from one iteration
   would leak to the next. The diff shows `modelDetails :=
   api.ModelDetails{...}` which uses short-var-decl, so it
   shadows per-iteration. Good.

## Verdict

`merge-after-nits` — the fix is correct and matches
`/api/show`'s existing behavior, addressing a real
inconsistency. The headline issue is the test doesn't actually
exercise the enrichment path. Three follow-ups:

- rewrite `TestListSafetensorsDetails` to set empty manifest
  values and assert enrichment from `config.json` populates them,
- add a `slog.Debug` on `GetSafetensorsLLMInfo` failure so
  silent empty-size cases are debuggable,
- benchmark `/api/tags` latency with N safetensors models and
  decide whether a `(n, mtime)`-keyed cache is worth it.

## What I learned

The "two endpoints that should return the same data return
different data" bug is endemic to systems where the
serialization layer for each endpoint is hand-rolled. The
clean fix is a shared `buildModelDetails(name, cf)` helper that
both `ShowHandler` and `ListHandler` call — that way a future
contributor adding a new endpoint that returns model metadata
gets the safetensors enrichment for free instead of
rediscovering this bug. The lesson: when a fix consists of
"copy logic from endpoint A to endpoint B", the next refactor
should extract that logic into a shared helper that both
endpoints depend on.
