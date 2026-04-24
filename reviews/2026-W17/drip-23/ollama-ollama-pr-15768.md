# ollama/ollama PR #15768 — feat(runner): expose prompt cache hit count in completion response

- **Repo:** ollama/ollama
- **PR:** [#15768](https://github.com/ollama/ollama/pull/15768)
- **Head SHA:** `d6b8a1bdb538d60b3850f2cc7d5ac6ca252e3e9c`
- **Size:** +3/-0 across 2 files
- **Reviewer:** Bojun (drip-23)

## Summary

Three-line additive change that surfaces the existing prompt-cache
hit counter from the runner's per-sequence state up through the
`/completion` HTTP response. Adds:

- `PromptCachedCount int` field to `CompletionResponse` in
  `llm/server.go` (with `json:",omitempty"` so unaffected runners
  serialize empty);
- `numCachedInputs int` field on `Sequence` in
  `runner/ollamarunner/runner.go`;
- one assignment after `cache.Inputs` is loaded
  (`seq.numCachedInputs = len(seq.cache.Inputs)`);
- one passthrough in the response construction
  (`PromptCachedCount: seq.numCachedInputs`).

No behavior change. No new validation. No semantic redefinition of any
existing field. Pure observability surface bump.

## What's changed

### `llm/server.go` (+1/-0)

Diff lines 1–11:

```go
 type CompletionResponse struct {
   Done               bool          `json:"done"`
   PromptEvalCount    int           `json:"prompt_eval_count"`
   PromptEvalDuration time.Duration `json:"prompt_eval_duration"`
+  PromptCachedCount  int           `json:"prompt_cached_count,omitempty"`
   EvalCount          int           `json:"eval_count"`
   EvalDuration       time.Duration `json:"eval_duration"`
```

The new field sits between `PromptEvalDuration` and `EvalCount` —
struct field order is irrelevant for JSON serialization but is
preserved in Go's reflect output, so any downstream tooling that
introspects struct field order will see the new ordering. Almost
certainly fine.

The `,omitempty` tag (line 5) is the right call — it means runners
that don't set this field (i.e., older runner binaries paired with
a newer ollama server, or non-ollamarunner backends like the
llama.cpp runner) will not emit `prompt_cached_count: 0` in the
JSON, so clients can detect "field present" vs "field zero" cleanly.

### `runner/ollamarunner/runner.go` (+2/-0)

Diff lines 13–34:

```go
 type Sequence struct {
   ...
   numPredicted     int
   numPromptInputs  int
+  numCachedInputs  int
 }
```

Then at line 25, after `s.cache.LoadCacheSlot(seq.inputs, false)`
returns successfully:

```go
+ seq.numCachedInputs = len(seq.cache.Inputs)
```

And at line 34, in the final `CompletionResponse` send:

```go
+ PromptCachedCount:  seq.numCachedInputs,
```

`seq.cache.Inputs` is the cache hits found during prefix lookup —
the standard "how many tokens did we skip recomputing thanks to the
KV cache" counter. The naming aligns with the existing
`PromptEvalCount` (= newly-evaluated prompt tokens) so a client can
compute total prompt tokens as `PromptEvalCount + PromptCachedCount`.

## Concerns

1. **Snapshot timing — `seq.numCachedInputs` is captured *before*
   the runner runs**

   The assignment at diff line 25 happens immediately after
   `s.cache.LoadCacheSlot(...)` succeeds — that's correct. But it
   captures the count at *load* time, not at *response* time. For
   normal completion runs that's fine (the cache slot is allocated
   once and consumed). For any future feature that re-loads the
   cache mid-sequence (speculative decoding rollback, retry-with-
   different-tokens, etc.), `seq.numCachedInputs` would become
   stale. Not a current bug; flagging because the field name
   ("Cached Count") doesn't communicate the snapshot semantics.
   Worth a one-line comment: `// snapshot of cache hits at slot
   load time; not updated mid-run`.

2. **Only the `ollamarunner` backend populates this**

   The `runner/ollamarunner/runner.go` change populates the field;
   the older `runner/llamarunner/` backend (if it still exists) does
   not. So users on llama.cpp-based runners will see
   `prompt_cached_count` absent (good — `,omitempty`) but won't get
   the metric. If that backend is being deprecated, fine; if not,
   the asymmetry should be called out in the PR description so
   users know which runner they need to be on.

3. **No test coverage**

   No test asserts that `PromptCachedCount` is non-zero on a
   second identical request to the same model (which is the
   trivial expected behavior — first request prefills cache, second
   request hits cache fully). A simple integration-ish test in
   `runner/ollamarunner/runner_test.go` along the lines of:

   ```go
   func TestCompletionReportsCachedTokens(t *testing.T) {
       // first call: prompt_cached_count == 0
       // second call (same prefix): prompt_cached_count > 0
       // third call (same prefix): prompt_cached_count == prompt_eval_count of first
   }
   ```

   would pin the behavior. Without it, a future refactor of
   `s.cache.Inputs` semantics (e.g., changing what counts as a
   "hit") silently breaks the surfaced metric.

4. **No `/api/generate` / `/api/chat` doc update**

   The PR adds a public field to a public response type but does
   not update `docs/api.md` (or wherever ollama documents the
   completion response shape). Users will see the new field
   appear in their JSON without any documentation explaining what
   it means or that it can legitimately be missing. Worth a
   one-line addition to the API docs:

   > `prompt_cached_count`: number of prompt tokens served from
   > the KV cache (not re-evaluated). Absent when the runner does
   > not provide cache statistics.

5. **Field naming consistency with downstream OpenAI-compat layer**

   ollama's `/v1/chat/completions` OpenAI-compat layer maps these
   counters into the OpenAI `usage` block. If this new field
   propagates through, it should map to the OpenAI
   `usage.prompt_tokens_details.cached_tokens` shape (which
   OpenAI added in late 2025). Worth a follow-up PR to wire that
   through — otherwise the metric is visible only on ollama-
   native API consumers.

## Verdict

`merge-as-is` — it's a 4-line additive change with `,omitempty`
guarding backward compat, sourced from data the runner already
collects. The concerns above are all "follow-up" rather than
"blocker":

- comment explaining the snapshot timing,
- one integration test for the cached-tokens behavior,
- API doc update,
- OpenAI-compat layer wiring.

None of those need to land in this PR; merging this and filing
a single tracking issue for the four follow-ups is the cleaner
shape.

## What I learned

ollama's runner has been quietly tracking prompt-cache hits for
a long time (`seq.cache.Inputs` is pre-existing); this PR just
plumbs the count out. It's the same pattern as a recent codex
PR I reviewed (#19308 — surfacing reasoning_output_tokens on the
exec JSON usage payload): the data exists deep in the runner,
the response-shape upgrade is one field, and the only real risk
is downstream consumers that strict-parse the JSON. The lesson:
when the runtime already collects a metric, surfacing it costs
~5 lines and should be a low-friction default. Prompt-cache hit
visibility is especially valuable for ollama users running
multi-turn chats where the cache hit ratio is a major
performance lever.
