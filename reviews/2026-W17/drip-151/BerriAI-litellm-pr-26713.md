# BerriAI/litellm#26713 — fix(otel): serialize non-string content attrs and guard non-dict response_obj

- **Repo:** [BerriAI/litellm](https://github.com/BerriAI/litellm)
- **PR:** [#26713](https://github.com/BerriAI/litellm/pull/26713)
- **Head SHA:** `05df3377e5c97a17c74c311bd675da91acc2365b`
- **Size:** +2461 / -2454 (one file: `litellm/integrations/opentelemetry.py` — line-ending churn obscures a tiny semantic diff)
- **State:** OPEN
- **Closes:** #24057, #24516

## Context

Two related crash classes in the OTel integration that fire on routine
production traffic:

1. **`gen_ai.prompt` rejection** (#24057): For multimodal messages,
   `msg["content"]` is a `list[dict]` of content blocks (image, tool
   result, document). The OTel SDK rejects non-primitive attribute
   values, so `attrs["gen_ai.prompt"] = msg["content"]` raises
   `TypeError` and the span fails to export.

2. **`AttributeError` on non-dict `response_obj`** (#24516): The Usage
   AI chat flow surfaces `response_obj` as a `list`. Several call sites
   guard with truthiness (`if response_obj:`) and then call
   `response_obj.get(...)` — which crashes on lists.

Both are silent-degradation bugs from the user's POV (logging fails,
real request still completes), but they nuke entire telemetry pipelines.

## Design analysis

The diff is enormous (`+2461 / -2454`) because the file was rewritten
top-to-bottom with what looks like a line-ending normalization
(CRLF → LF or similar). The actual semantic change is small and lives
in three spots — easy to lose if you skim. Worth asking the author to
either separate the line-ending churn into its own commit or rebase
onto a clean base.

### Fix 1: `gen_ai.prompt` non-string handling

At `opentelemetry.py:3567-3570`:

```python
if isinstance(content, str):
    attrs["gen_ai.prompt"] = content
else:
    attrs["gen_ai.prompt"] = safe_dumps(content)
```

Right pattern, mirrors the already-existing `safe_dumps` calls for
`transformed_choices` (`:1727`) and `finish_reasons` (`:1739`) elsewhere
in the file. `safe_dumps` is the project's own JSON-with-fallback
serializer (handles `bytes`, datetime, etc.) so this should never raise
even on exotic content blocks.

### Fix 2: `isinstance(response_obj, dict)` guards

Multiple sites converted from truthy to type-checked guards. The key
ones:

- `:3339`: `isinstance(response_obj, dict)` before `.get("usage")` in
  `_record_metrics`
- `:3428`: `if isinstance(response_obj, dict) and (usage := response_obj.get("usage")):`
- `:3526`: `if not isinstance(response_obj, dict): return` early-bail
  in `_emit_semantic_logs`
- `:4075`, `:4093`, `:4100`, `:4178`: same pattern repeated through
  `set_attributes`

This is the correct primitive — `bool(list)` is true for non-empty
lists, so the previous truthy guards passed and then crashed on the
attribute access. `isinstance(..., dict)` is the right check to gate
`.get()`.

### Fix 3: `gen_ai.system` defaulting

`custom_llm_provider` was previously defaulted via `.get("custom_llm_provider", "Unknown")` which fails when the key is present
but `None`. The change to `or "Unknown"` (visible at `:3300`, `:3561`,
`:3588`) handles both missing-key and present-but-`None` cases. Small
but right.

## Risks / nits

1. **Line-ending churn obscures the diff.** The `+2461 / -2454` shape
   means the entire file was deleted and re-added with one or two char
   differences per line, or with CRLF→LF conversion. Reviewers can't
   easily see the actual semantic change without doing
   `git diff -w` / line-ending-aware diff. Strongly request: split into
   two commits — one pure line-ending normalization, one semantic fix.
   Otherwise future `git blame` on this file will point at this PR for
   every line, hiding the actual authors.

2. **`safe_dumps` non-deterministic ordering.** If `safe_dumps` doesn't
   pass `sort_keys=True`, two semantically-identical multimodal messages
   could produce different `gen_ai.prompt` values across runs (Python
   dict insertion order is stable per process, but content-block dicts
   from different code paths may be assembled in different orders).
   Worth confirming `safe_dumps` is sort-stable, or accepting that the
   `gen_ai.prompt` attribute is opaque-bytes-for-replay rather than
   stable-content-hash.

3. **`gen_ai.prompt` size.** Multimodal content blocks can include
   base64-encoded images. Dumping that into a single OTel attribute
   means a single span attribute can be megabytes. Most OTel collectors
   have a per-attribute size cap (8KB or 32KB depending on backend).
   Worth a length cap with a `... [truncated, N bytes]` suffix to
   prevent silent attribute drops at the collector. The PR description
   doesn't address this and the existing `transformed_choices` /
   `finish_reasons` paths likely have the same gap.

4. **No regression test for either fixed crash.** The PR body mentions
   no tests added. Both bug shapes are easy to write a unit test for:
   one with a `list[dict]` content, one with `response_obj = []`. Without
   tests, the next refactor of these helpers reintroduces the same
   crash. Strongly recommend adding two minimal regression tests against
   the issue numbers (`#24057`, `#24516`).

5. **`isinstance(response_obj, dict)` is fragile if the project starts
   using a `Mapping` subclass.** A `MappingProxyType` or `OrderedDict`
   passes (subclass), but a `pydantic.BaseModel` or `TypedDict`-typed
   value would not. Right call for now (matches the actual production
   shape) but worth a comment that the guard is dict-by-design.

## Verdict

**merge-after-nits.** The semantic fixes are correct and the bug shapes
are real. Pre-merge asks: split the line-ending churn out (nit 1 — this
will save every future reviewer time) and add at least one regression
test per closed issue (nit 4). Nits 3 and 5 are follow-ups; nit 2 is a
question to confirm.

## What I learned

- `bool(some_list)` is true for non-empty lists, so a truthy guard that
  was *intended* to gate dict access fails open as soon as a non-dict
  shape sneaks through the call site. `isinstance(x, dict)` is the
  primitive you actually wanted.
- `safe_dumps`-style "JSON-with-fallback" serializers exist for
  exactly this kind of telemetry attribute boundary — where the input
  is "anything the user model could produce" and the output must be a
  primitive string. Using the project's existing helper (vs `json.dumps`
  with a custom default) keeps the failure modes consistent across the
  codebase.
- A pure line-ending normalization commit is essentially free to do
  separately and pays back enormous review-cost dividends. Mixing it
  with a semantic fix means every reviewer has to do `git diff -w`
  manually before they can see what changed.
