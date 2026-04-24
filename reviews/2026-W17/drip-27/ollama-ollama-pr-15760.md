# ollama/ollama PR #15760 — Apply config.json per-tensor quant overrides for mixed-precision MoE

- **Repo:** ollama/ollama
- **PR:** [#15760](https://github.com/ollama/ollama/pull/15760)
- **Head SHA:** `c917ad0bafbe4d9e920cee1333a4a76124c0c22d`
- **Author:** jodagreyhame
- **Size:** +700 / −27 across 10 files (supersedes #15744, stacked on #15759)
- **Reviewer:** Bojun (drip-27)

## Summary

Fixes a `panic: runtime error: index out of range [0] with length 0`
in `SparseMoE.Forward` when running mlx-lm mixed-precision NVFP4 MoE
models imported via `ollama create --experimental` (concrete
repro: Qwen 3.6 35B-A3B NVFP4 crashes on the first token without this
change). Root cause is that mlx-lm stores per-path quantisation overrides
in `config.json` under a `quantization` (or `quantization_config`) block,
and the runtime was ignoring them and falling back to global defaults
that don't match the per-tensor metadata. PR adds a parser
(`config_quant.go`) that reads the block and threads a per-path
override map through layer construction. Also incorporates a P1 fix
from a prior `Codex` bot review on `config_quant.go:77` (the
`QuantType` derivation needed `mode + bits`, not `mode` alone, to
avoid `{mode:"affine", bits:4}` collapsing to `"AFFINE"` and silently
dropping bit info via the `QuantizationParams` unknown-fallback).

## Key changes

### `x/mlxrunner/model/config_quant.go:1–96` — new file

`readConfigQuantOverrides(m *manifest.ModelManifest)` reads
`config.json`, looks for `"quantization"` (or
`"quantization_config"`), and returns:
- a `TensorQuantInfo` for global defaults
- a `map[string]*TensorQuantInfo` keyed by `<path>.weight` for
  per-tensor overrides

Missing or malformed config is a *silent* fallback (zero values, nil
error). Per-tensor overrides default `Mode` to `"affine"`
(matching `mx.nn.Linear.to_quantized`'s default) when omitted.

### `x/mlxrunner/model/config_quant.go:80–96` — `quantTypeForModeBits`

```go
func quantTypeForModeBits(mode string, bits int) string {
    if strings.EqualFold(mode, "affine") {
        switch bits {
        case 4:
            return "INT4"
        case 8:
            return "INT8"
        }
    }
    ...
}
```

This is the function the prior bot review caught. The previous
implementation upper-cased the mode string ("affine" → "AFFINE"),
which then collided with the unknown-quant fallback in
`QuantizationParams` and silently dropped the `bits` info entirely.
The new implementation derives the canonical `INT4` / `INT8` name
from `(mode, bits)` together. This is the only path by which
`{mode:"affine", bits:4}` can survive the round trip into the
quant runtime intact.

### Per-path key shape: `<path>.weight`

Overrides are stored keyed by `<key>.weight` (where `<key>` is the
JSON key inside the `"quantization"` block, e.g.
`"model.layers.0.feed_forward.experts.0.gate_proj"`). This means
downstream lookups in `MakeLinearLayer` need to ask
`overrides[path+".weight"]`, which is consistent with how the
linear factory already constructs paths.

### Silent fallback semantics

Three places where the parser silently returns `(zero, nil, nil)`:
- `m.ReadConfig("config.json")` errors → no config, no overrides
- `json.Unmarshal(data, &cfg)` errors → unparseable config, no overrides
- `json.Unmarshal(raw, &block)` errors → unparseable quant block,
  no overrides

This is the right behaviour for an *experimental* import path — a
malformed config shouldn't prevent loading models that don't have
mixed-precision quant in the first place. But see concern (1) below.

## What's good

- Right diagnosis: the panic is a NULL/empty-tensor read from a
  shape mismatch, and the fix is to *honour* the per-path quant
  metadata rather than to add a defensive null check at the panic
  site. Defensive null checks would have masked the underlying
  shape mismatch and produced wrong outputs silently.
- The bot-feedback fix on `quantTypeForModeBits` is exactly the
  right shape: the previous implementation looked correct but had
  a silent collision with the unknown-quant fallback. The new
  implementation makes that collision impossible by construction.
- `mode` defaults to `"affine"` to match `mx.nn.Linear.to_quantized`,
  which is the upstream-MLX behaviour. This aligns Ollama's import
  path with what mlx-lm itself does, so users who export a model
  with mlx-lm and import via `ollama create --experimental` get the
  same effective quant config.
- Both `quantization` and `quantization_config` JSON keys are
  recognised. Both names appear in mlx-lm exports depending on
  version, and accepting both avoids a "this works on mlx-lm 0.30
  but not 0.31" trap.
- The PR is stacked correctly on #15759 (the naming-recognition
  PR) which means the per-tensor overrides land *after* the
  alternate-naming support — the right order, since per-path
  overrides need both naming forms to be recognised first.

## Concerns

1. The three silent-error returns in `readConfigQuantOverrides` will
   make "I exported with mlx-lm and the model doesn't load right"
   bug reports very hard to diagnose. Even if the *runtime* behaviour
   should be silent fallback, a `slog.Debug` (or `slog.Warn` for the
   "block exists but is malformed" case) at each fallthrough would
   make this much easier to debug:

   ```go
   if err := json.Unmarshal(raw, &block); err != nil {
       slog.Debug("mlxrunner: ignoring malformed quantization block in config.json", "error", err)
       return TensorQuantInfo{}, nil, nil
   }
   ```

   The first variant ("file not found") can stay silent since it's
   the common case for non-MLX models; the latter two are user-
   error signals that should be surfaced.

2. `_ = json.Unmarshal(val, &defaults.GroupSize)` and friends
   discard the error from each per-field unmarshal. If `"bits": "4"`
   (string instead of int) appears in the wild, the field stays at
   zero and the rest of the block parses fine — but the quant params
   for that block will be nonsense. Worth either logging or
   collecting errors and returning them once.

3. The `"affine" + bits → INT4/INT8` mapping in
   `quantTypeForModeBits` only handles bits=4 and bits=8. If
   somebody uses bits=2 or bits=6 (mlx-lm supports both), the
   function falls through to whatever the default branch returns
   — needs a quick check in the diff that the fallthrough matches
   what `QuantizationParams` expects, otherwise we recreate the
   exact "silently drops bit info" bug the PR was fixing for the
   bits=2/bits=6 case.

4. No test in this PR (per the diff sample I have) explicitly
   validates the `{mode:"affine", bits:4} → "INT4"` derivation. The
   prior review caught it, so the next regression should be guarded
   by a unit test on `quantTypeForModeBits`. Even one table-driven
   test asserting the four canonical pairs (`affine/4 → INT4`,
   `affine/8 → INT8`) would prevent recurrence.

5. The `<key>.weight` key shape assumes downstream lookups always
   append `.weight`. If a future contributor adds a quantized layer
   that uses a different aux name (e.g. `.bias` for biased quant),
   they'll silently skip the per-path override. Worth a doc comment
   on the returned map noting the keying convention.

## Risk

Moderate. The fix is in the import path for `ollama create
--experimental`, which is explicitly an experimental surface, so
the blast radius is bounded to users who opt in. Inside that
audience the behavioural change is "models that previously panicked
on first token now load and produce output" — which is the right
direction, but worth verifying against at least one
non-mixed-precision MLX model to confirm the unchanged path stays
unchanged.

## Verdict

**merge-after-nits**

The diagnosis and the fix are right, and the response to the prior
bot review (the `quantTypeForModeBits` change) is exactly what was
needed. The blockers I'd want addressed before merge are:

- Concern 1 (`slog.Debug` on the two malformed-block fallthroughs)
- Concern 4 (a 10-line table-driven test on `quantTypeForModeBits`)

Concerns 2, 3, and 5 are improvements that can land as follow-ups.

## What I learned

The "uppercase the mode string" approach to deriving a canonical
quant-type name is a classic example of a fix that looks identical
to the right answer but silently collides with an unknown-fallback
sentinel downstream. The right answer is to derive the canonical
name from the *full* identifying tuple (mode + bits) at the
boundary, so that downstream code never has to re-derive bits from
a stringly-typed mode. Worth keeping in mind for any "stringly
typed enum with default fallback" pattern — the silent-collision
failure mode is the worst kind of bug because it produces wrong
output rather than a crash.
