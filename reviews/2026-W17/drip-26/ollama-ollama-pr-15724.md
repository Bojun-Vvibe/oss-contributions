# ollama/ollama PR #15724 — convert: support fp8 safetensors import

- **Repo:** ollama/ollama
- **PR:** [#15724](https://github.com/ollama/ollama/pull/15724)
- **Head SHA:** `799c0e9cc578f1461b27144a6ceee5762b669f26`
- **Author:** dhiltgen (Daniel Hiltgen)
- **Size:** +884 / −33 across 8 files
- **Reviewer:** Bojun (drip-26)

## Summary

Adds first-class support for importing HuggingFace `F8_E4M3`
(FP8 with 4-bit exponent / 3-bit mantissa) safetensors weights
into Ollama's GGUF converter. Two coupled pieces:

1. **Reader side** (`convert/reader_safetensors.go`,
   `convert/convert.go`): detect F8_E4M3 weight tensors plus their
   companion block-scale tensors (`weight_scale`,
   `weight_scale_inv`, etc.), pair them up via
   `collectSafetensorsFP8Scales`, and during `WriteTo` decode the
   FP8 bytes back to FP32 by multiplying by the per-block scale.

2. **Quant side**: record per-output-tensor source-precision metadata
   (`sourceTensorKV`) in the GGUF header so that
   `create`-time quantization can make precision-aware choices —
   default FP8-sourced weights to Q8_0; when targeting Q8_0,
   leave non-FP8 tensors at their original precision; when
   targeting Q4_K, promote *non-FP8* quantizable tensors to Q8_0
   so the FP8-source tensors remain the bottleneck rather than
   being further degraded.

This is the largest patch in this drip and the only one that
adds new quant-pipeline logic.

## Key changes

### `convert/reader_safetensors.go:25–110` — pairing FP8 weights with their scales

```go
func parseSafetensors(fsys fs.FS, replacer *strings.Replacer, ps ...string) ([]Tensor, error) {
    fp8Block, err := safetensorsFP8BlockSize(fsys)
    if err != nil {
        return nil, err
    }
    ...
    fp8Scales, err := collectSafetensorsFP8Scales(n, headers)
    ...
    for _, key := range keys {
        if value := headers[key]; value.Type != "" {
            if _, ok := fp8Scales.consumed[key]; ok {
                continue  // skip the scale tensors themselves
            }
            ...
            var scale *safetensorScale
            if value.Type == "F8_E4M3" {
                if !fp8Block.ok {
                    return nil, fmt.Errorf("missing fp8 block size metadata for tensor %q", key)
                }
                scale = fp8Scales.byWeight[key]
                if scale == nil {
                    return nil, fmt.Errorf("missing fp8 scale companion for tensor %q", key)
                }
            }
            ts = append(ts, safetensor{ ..., scale: scale, fp8Block: fp8Block, ... })
        }
    }
}
```

Two important guarantees expressed here:

- A weight whose dtype is `F8_E4M3` *must* have block-size
  metadata at the file level *and* a scale companion in the
  same file, or the converter hard-fails. No silent "import
  without scales and produce garbage outputs" path.
- The scale tensor is consumed (`fp8Scales.consumed[key]`) so it
  doesn't appear as a separate output tensor in the GGUF — it
  exists only as multiplicative state during decode.

### `convert/reader_safetensors.go:282–end` — `safetensorsFP8Scale` candidate matching

```go
candidates := safetensorsFP8ScaleCandidates(key)
var scaleKey string
var scaleValue safetensorMetadata
if strings.HasSuffix(key, ".weight") {
    base := strings.TrimSuffix(key, ".weight")
    candidates = appendUnique(candidates, base+".weight_scale")
    candidates = appendUnique(candidates, base+".weight_scale_inv")
}
```

Handles both the modern naming (`<base>.weight` /
`<base>.weight_scale_inv`) and the compressed-tensors export
naming (with the scale suffix between the module path and the
weight suffix). Reasonable.

### `convert/reader_safetensors.go:165–172` — Kind() override

```go
func (st safetensor) Kind() uint32 {
    ...
    if st.dtype == "F8_E4M3" && kind != tensorKindFP32 {
        kind = tensorKindBF16
    }
    return kind
}

func (st safetensor) SourceDType() string {
    return st.dtype
}
```

`SourceDType()` is the new accessor that the quant side reads to
make precision-aware decisions. The `Kind()` override forces FP8
output to BF16 (unless explicitly FP32) — the same write-side
type the rest of the GGUF tensor table uses for "high-precision
intermediate".

### `convert/convert.go:390–393` — sourceTensorKV pulled into header

```go
for k, v := range sourceTensorKV(ts) {
    kv[k] = v
}
```

The `sourceTensorKV(ts)` builder isn't visible in this excerpt of
the diff but presumably records "tensor X originated from FP8" as
a KV pair under a stable key prefix. This is the metadata that the
quant pipeline later reads to make Q8_0/Q4_K decisions.

### `safetensor.WriteTo` (lines 244–254) — actual decode path

```go
case "F8_E4M3":
    u8s := make([]uint8, st.size)
    if err = binary.Read(br, binary.LittleEndian, u8s); err != nil {
        return 0, err
    }
    f32s, err = st.decodeFP8E4M3(u8s)
    if err != nil {
        return 0, err
    }
```

Standard pattern: read raw bytes, dispatch to a private
`decodeFP8E4M3` (presumably handles the FP8 → FP32 reconstruction
plus per-block scale multiplication using `st.scale` and
`st.fp8Block`). Body of `decodeFP8E4M3` is in the part of the
diff we couldn't view in this review window — that's the
correctness-critical math and needs separate eyes (see Concerns).

### `Clone()` updates

`safetensor.Clone()` and the new `(*safetensorScale).Clone()`
properly deep-copy the new fields. Important because the converter
repacker path calls `Clone()` on tensors and would otherwise share
mutable scale state across goroutines.

## What's good

- **Hard-fail on missing scales.** The
  `missing fp8 block size metadata` and `missing fp8 scale
  companion` returns close the most dangerous failure mode for
  this kind of import: silently producing weight tensors with
  no scale applied, which would yield outputs that look
  syntactically valid but are mathematically wrong (off by the
  per-block scale factor). Returning an error here means the
  user sees the failure at convert time rather than at
  inference time.
- **`consumed` set deduplication of scale tensors.** Catching
  `"fp8 scale companion %q is used by multiple tensors"`
  prevents accidentally pairing one scale to multiple weights —
  which would happen if the candidate-name matching in
  `safetensorsFP8Scale` produced an ambiguous match.
- **Source-precision metadata is preserved end-to-end** through
  to the quant pipeline. That's the right architecture: the
  reader records what it knows about the source, the quant
  stage makes the precision-aware decision later. The
  alternative (forcing the reader to make the quant decision)
  would not generalise beyond FP8.
- **Two naming conventions handled.** Catching both
  `<base>.weight_scale_inv` and the compressed-tensors variant
  means real-world HF FP8 exports actually load.
- **`Clone()` correctness.** Adding `scale.Clone()` and
  `slices.Clone(ss.shape)` to the new clone path is exactly
  what's needed to keep the existing repacker concurrency safe.

## Concerns

1. **The actual FP8 decode math (`decodeFP8E4M3`) is not in the
   visible diff window.** Everything above is plumbing; the
   correctness of the import depends entirely on the FP8 → FP32
   conversion routine. Specifically:
   - F8_E4M3 has 4 exponent bits, 3 mantissa bits, 1 sign bit;
     bias is 7; max representable value is 448; subnormals exist;
     no NaN/Inf encoding (E4M3FN convention) vs the rarer
     E4M3FNUZ. Which convention is decoded matters for the
     extreme values.
   - The block-scale multiplication has to use the right block
     dimension order — HF F8_E4M3 typically stores per-block
     scales in row-major across [out_features, in_features] in
     blocks of e.g. 128×128, and getting the indexing wrong
     produces output that's correct in shape but wrong
     element-wise.
   - Need to confirm whether the scale tensor is interpreted as
     `weight_scale` (multiply by) or `weight_scale_inv` (divide
     by) — the code candidate-matches both names but the
     mathematical operation differs. If there's no per-tensor
     branch on the suffix, one of the two will silently produce
     reciprocal-scaled outputs.
2. **No new unit/integration tests visible.** A patch of this size
   touching the converter math should ship a small fixture-driven
   test: a tiny synthetic F8_E4M3 safetensors file with a known
   scale, run through `parseSafetensors` + `WriteTo`, assert
   the resulting FP32 values are within tolerance of a
   precomputed reference. Without this, every future refactor of
   the decode path risks silently regressing model fidelity.
3. **Quant defaults are policy choices.** "default FP8-sourced
   GGUFs to Q8_0" and "promote non-FP8 quantizable tensors to
   Q8_0 for Q4_K requests" are reasonable defaults but they
   change the meaning of `ollama create -f Modelfile` for users
   who request Q4_K on an FP8 source — they'll get a
   mixed-precision file with non-FP8 layers at Q8_0, not a
   homogeneous Q4_K file. This needs to be called out in
   release notes; users running benchmarks comparing Q4_K
   ollama output to Q4_K llama.cpp output will see different
   sizes/speeds.
4. **`sourceTensorKV(ts)` collision with existing KV keys.** The
   merge `for k, v := range sourceTensorKV(ts) { kv[k] = v }` at
   `convert.go:390` overwrites without checking. If `ts` ever
   contains a key that collides with a converter-set KV (model
   architecture, vocab size, etc.), the source-precision value
   silently wins. A `if _, ok := kv[k]; ok { return error }`
   guard would catch namespace collisions at convert time.
5. **`sync.Mutex`-style protection of shared state on `*safetensorScale`?**
   The `Clone()` is good but there's no obvious guard against
   two goroutines calling `WriteTo` on a clone vs the original
   while the original is still being read. Probably fine because
   the converter is single-pass per file, but worth confirming.
6. **`appendUnique`** is presumably a small slice-dedup helper —
   not visible in the diff. If it's O(n²), the candidate list
   stays small (≤ ~6 names), so cost is negligible.
7. **Error wrapping verbosity.** The two new `fmt.Errorf` calls
   include the tensor key, which is great for debugging. They
   don't include the file path; for a multi-file export this
   would help users locate the broken file.

## Risk

Medium. The plumbing changes (pairing, Clone, dispatch) are well
structured and fail loudly on missing companions. The
*correctness* of the actual FP8 decode is the gating risk and is
not reviewable from this diff window — the math has to be right
for the imported model to produce valid outputs at all.

## Verdict

**needs-discussion**

Substantively this looks like the right design (hard-fail on
missing scales, propagate source precision to the quant stage,
two naming conventions handled, clones updated). But the gating
question — "is `decodeFP8E4M3` mathematically correct, including
E4M3FN vs E4M3FNUZ, scale-vs-scale_inv, and block-major
ordering?" — needs a maintainer with FP8 reference numbers in
hand to validate, plus a reproducible fixture test.

Concrete asks:

- Add at least one fixture-driven test producing a deterministic
  output for a known synthetic F8_E4M3 input (concern #2).
- Document in code which FP8 convention (E4M3FN vs E4M3FNUZ) is
  assumed and which scale interpretation
  (multiply-vs-divide) is used per suffix (concerns #1, #1).
- Add a release-notes-worthy paragraph on the Q4_K-mixed-precision
  default (concern #3).
- Add a collision guard on the `sourceTensorKV` merge
  (concern #4).

If the math is verified by the maintainer team and the test fixture
is added, this is a strong contribution.

## What I learned

When importing weights from a quantized format with separate scale
tensors, the *pairing* logic is the single highest-leverage place
to make the import robust: every other "did the math work?"
question downstream is gated on "did we even pair the right
tensors?". This patch's choice to fail loudly with the tensor name
on missing scales/blocks (rather than warn-and-continue or
substitute defaults) is exactly the right posture — precision
imports either succeed end-to-end or they fail explicitly, never
"work but with wrong values". The cost is one less convenience
case ("just import what you can"), which is a fair price for
not silently corrupting weights.
