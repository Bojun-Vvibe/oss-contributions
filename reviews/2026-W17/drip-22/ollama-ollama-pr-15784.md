# PR #15784 — sample: implement repeat, frequency, and presence penalties in Go sampler

- URL: https://github.com/ollama/ollama/pull/15784
- Author: 42euge
- Head SHA: `a19534d3da0d106fa14a3cfe70e222e06ab0de8f`

## Summary

The Go-native sampler (`sample/samplers.go`), used by models on the
`ollamarunner` path, accepted `repeat_penalty`, `frequency_penalty`, and
`presence_penalty` via the API but silently ignored them. This PR wires
the API options through to the sampler, adds a `repeatPenalize` transform
that mirrors llama.cpp's `sampling_repetition_penalties`, and threads a
ring-buffer `tokenHistory` through `Sampler.Sample`. Most visible win is
on Gemma 4 e4b audio transcription: WER on long paragraphs drops from
84.7% → 30.6%. Fixes #15783, related #9278.

## Specific callouts

- `sample/transforms.go:114-148` — `repeatPenalize` is the algorithmic
  core. Three correct details:
  1. Early-return when `len(lastTokens) == 0 || (penalty == 1.0 &&
     frequencyPenalty == 0 && presencePenalty == 0)` — keeps the hot path
     allocation-free for the default config. Good.
  2. Asymmetric penalty application:
     ```go
     if ts[i].value > 0 { ts[i].value /= penalty }
     else               { ts[i].value *= penalty }
     ```
     This is the llama.cpp idiom: repeat-penalty must shrink magnitudes
     toward zero regardless of sign, otherwise penalising a negative
     logit by multiplying makes it *more* negative (i.e. less likely),
     which is the desired direction, while penalising a positive logit by
     dividing also reduces likelihood. Behavior matches reference impl.
  3. Frequency uses count, presence is binary (`-= presencePenalty` once
     per token-in-history regardless of count). Matches OpenAI's API
     semantics for those names.

  One nit: `counts := make(map[int32]int, len(lastTokens))` allocates per
  Sample call. For typical `repeatLastN=64` this is a 64-entry map per
  token, i.e. per output token. Consider reusing a per-`Sampler` map
  cleared between calls, or a dense `[]int8` keyed by token id when the
  vocab fits — though that costs memory. Profile first; the current
  shape is correct, just not minimal.
- `sample/samplers.go:20-32` — `Sampler` struct grows by 5 fields. All
  reasonable. `tokenHistory []int32` is a slice not a ring buffer; the
  ring semantics are emulated by the slice trim in `recordToken`:
  ```go
  if len(s.tokenHistory) > s.repeatLastN {
      s.tokenHistory = s.tokenHistory[len(s.tokenHistory)-s.repeatLastN:]
  }
  ```
  This re-slices but doesn't free the backing array, so over a long
  generation the underlying memory grows by one element per call until
  GC reclaims the abandoned prefix. For long generations, switch to a
  preallocated `[repeatLastN]int32` ring with a write index. Not
  blocking, but worth a TODO.
- `sample/samplers.go:158-193` — `NewSampler` signature gains four
  trailing args. This is a breaking change for every test/benchmark that
  constructs samplers. The diff updates all callers in
  `samplers_test.go` and `samplers_benchmark_test.go` correctly (six
  call sites, all passing `0, 0, 0, 0`). For external consumers — if
  any embedder uses `NewSampler` directly — the signature change will
  break the build. Two mitigations to consider:
  1. Make this a constructor with options pattern: `NewSampler(opts...)
     SamplerOption`. More idiomatic Go and makes future additions
     non-breaking.
  2. Or at minimum, document the API break in CHANGELOG.
- `runner/ollamarunner/runner.go:894-901` — Wires `req.Options.RepeatPenalty`
  / `RepeatLastN` / `FrequencyPenalty` / `PresencePenalty` through to
  `NewSampler`. The PR body claims `api/types.go` already defines defaults
  (`repeat_penalty: 1.1`, `repeat_last_n: 64`). Please verify those
  defaults actually flow through `req.Options` for requests that don't
  set them explicitly — if not, the no-penalty fast-path
  (`penalty == 1.0 && ...`) in `repeatPenalize` will be the common case
  and the WER improvements in the PR body won't materialize for default
  requests. A one-line test:
  `assert sampler.repeatPenalty == 1.1` after a default API call.
- `sample/samplers_test.go:152-200` — `TestRepeatPenaltyIntegration`
  covers four cases: no-penalty baseline, penalty steers off token-0,
  history accumulation, ring-buffer cap. Good coverage of the integration
  surface. Missing: a test for the asymmetric positive/negative logit
  branch, which is the most likely place for a sign error to creep in.
  Add a case with `logits=[-5.0, -5.0]`, history=[0], penalty=2.0, and
  assert token 1 wins (because `-5*2 = -10 < -5/1 = -5`... wait, -5 *
  2 = -10 is *more* negative, i.e. token 0 becomes less likely → token
  1 wins). That's exactly the direction the asymmetric branch is
  defending; pin it.

## Risks

- API breaking change for direct `NewSampler` callers. Likely fine for
  ollama's tree but breaks any embedder. Document.
- `tokenHistory` slice grows beyond `repeatLastN` between trims — small
  steady-state leak across very long generations.
- WER numbers in the PR body are impressive but model-specific (Gemma 4
  e4b, audio transcription). Worth a note that the *generic* repeat
  penalty implementation should not regress WER on text generation —
  consider running the existing eval suite at `repeat_penalty=1.0` and
  `1.1` and confirming neutral results.

## Verdict

**Verdict:** merge-after-nits

Correct algorithm, correct integration, real measured win. Three
follow-ups: (1) verify default `repeat_penalty=1.1` from `api/types.go`
actually reaches the sampler in default requests, (2) preallocate the
ring buffer to avoid the steady-state slice growth, (3) add the
asymmetric-sign test case. The `NewSampler` API break should at least
get a CHANGELOG note.
