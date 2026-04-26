# BerriAI/litellm #26526 — fix arize observability bugs

- **Repo**: BerriAI/litellm
- **PR**: #26526
- **Author**: mubashir1osmani
- **Head SHA**: d778be1f24fa94aa7688f45a6c919b71b545e1cb
- **Link**: https://github.com/BerriAI/litellm/pull/26526
- **Size**: ~720 diff lines, two files:
  `litellm/integrations/arize/_utils.py` (+~640) and
  `litellm/integrations/arize/arize_phoenix.py` (~+30 / −7).

## What it changes

A grab-bag of fixes to the Arize / Phoenix OTEL integration, none
of which the PR body documents (the "Relevant issues" section is
literally empty: `Fixes # ` placeholder). What the diff actually
does:

1. **Pydantic-aware response coercion** at `_utils.py:99-108` —
   `_set_response_attributes` now `model_dump()`s `BaseModel`
   responses before passing them to dict-keyed setters. Without
   this, `ImageResponse` / `ResponsesAPIResponse` dropped silently
   on the next `.get()`.
2. **Image span rendering rewrite** (`:120-176`) — old
   `_set_image_outputs` (`:227-242`, deleted) wrote `IMAGE_URL.{i}`
   keys; new version writes the openinference-canonical
   `LLM_OUTPUT_MESSAGES.0.MESSAGE_CONTENTS.{idx}.MESSAGE_CONTENT_IMAGE.IMAGE_URL`
   path so Phoenix's renderer actually picks up the image inline.
   Adds a 32 KB inline cap (configurable via
   `LITELLM_ARIZE_MAX_INLINE_IMAGE_BYTES`, set to ≤0 to disable);
   over-cap b64 payloads get an omission-notice text
   (`"[image omitted from trace: N bytes... sha256=...]"`).
3. **Mime-type resolver** (`:521-541`) — `image/png` was hardcoded;
   now resolves per-image `mime_type` → per-image `output_format`
   → response `output_format` → `image/png` default. `jpg → jpeg`
   normalisation included.
4. **Structured-output dict/Pydantic uniform accessor**
   (`:281-321`) — `_set_structured_outputs` was using `getattr` /
   `hasattr` on `output[]` items, so dict-shaped Responses-API
   payloads from the proxy path were silently skipped. New `_get`
   helper handles both shapes.
5. **New `set_parent_span_attributes`** (`:874-940`) — replaces
   `self.set_attributes` on the parent CHAIN span so the parent
   gets only session/request metadata and **doesn't double-count
   tokens** that the child LLM span already carries. The
   call-site swap is at `arize_phoenix.py:253-257` and `:300-307`.
6. **Guardrail-span parenting fix** (`arize_phoenix.py:236-244`,
   `:289-297`) — in SDK mode (no proxy parent), the guardrail
   span was an orphan root; now it's parented to the
   `litellm_request` span via `_trace.set_span_in_context(span)`.
7. **`async_post_call_success_hook` no-op override**
   (`arize_phoenix.py:90-99`) — overrides the base class to skip
   the hook because guardrail spans are now created at the global
   tracer level; the comment says "skipping this hook prevents
   both the orphan and the duplicate guardrail spans".
8. **Universal `_extract_chain_input` / `_extract_chain_output`
   helpers** (`:617-770`) for tool/agent/retriever/reranker/
   guardrail span shapes, with documented dispatch order:
   tool-call shape → chat messages → query+documents (reranker)
   → input/prompt/text → standard_logging_payload fallback.

## Strengths

- **The image-rendering fix is correct and verifiable.** The old
  `IMAGE_URL.{i}` key didn't match anything Phoenix renders;
  Phoenix keys off `LLM_OUTPUT_MESSAGES.0.MESSAGE_CONTENTS.{idx}.
  MESSAGE_CONTENT_IMAGE.IMAGE_URL` (matching what
  `openinference.instrumentation.ImageMessageContent` produces).
  Anyone with an image-gen request and a Phoenix UI could confirm
  this end-to-end.
- **The 32 KB inline cap with sha256 omission notice
  (`:582-596`) is the right tradeoff** — a 5 MB base64 PNG would
  blow out the OTEL exporter's per-span attribute size limit and
  silently drop the span. Including the sha256 short-hash means
  ops can correlate omitted images with stored payloads.
- **`set_parent_span_attributes` (`:874-940`) addresses a real
  double-counting bug** — calling the same `set_attributes` on
  parent+child meant token usage and cost were emitted twice and
  Phoenix's per-trace cost rollup was 2x reality. The split is
  the right shape.
- **Guardrail-span parenting fix is small and surgical**
  (`arize_phoenix.py:236-244`); the `_trace.set_span_in_context`
  fallback is exactly how OTEL recommends rooting orphan-prone
  spans.

## Concerns / asks

- **Zero tests for ~640 lines of new logic.** The PR's "Added
  testing" checkbox is *unchecked* in the body, and the maintainer
  template says "Adding at least 1 test is a hard requirement".
  This is the single biggest blocker. At minimum:
  - A `_get_image_trace_payload` test asserting the omission-
    notice path fires above the cap.
  - A `_set_structured_outputs` test for dict-shaped
    `output[]` items (the bug that motivated `_get`).
  - A `set_parent_span_attributes` test asserting tokens are
    NOT set on the parent span.
- **`_get_max_inline_image_bytes` (`:560-577`) parses
  `LITELLM_ARIZE_MAX_INLINE_IMAGE_BYTES` on every call.** For a
  high-throughput image endpoint this is one `os.getenv` +
  `int()` per image, per request. Cheap, but cache it — or
  resolve once at module load with a `functools.lru_cache(None)`.
- **The "PR scope is isolated, only solves 1 specific problem"
  checkbox is *unchecked***, and accurately so — this is at
  least 8 distinct fixes (enumerated above). The Arize team will
  have to bisect any future regression across all of them. Should
  be split into: (a) image rendering + cap; (b) Pydantic
  coercion + dict-shape `_get`; (c) parent-span deduplication;
  (d) guardrail-span parenting; (e) chain input/output extraction
  for non-LLM call types.
- **`async_post_call_success_hook` no-op override
  (`arize_phoenix.py:90-99`) silently breaks any subclass that
  relied on the base implementation.** The comment "already
  handled in global tracer provider" is the rationale, but the
  base class's behavior isn't documented in this diff. If a user
  has a subclass of `ArizePhoenixLogger` that depends on the
  parent hook firing, this is a silent regression. Consider
  raising `NotImplementedError` for subclasses, or document the
  contract change in a CHANGELOG entry.
- **`from opentelemetry import trace as _trace` is imported
  inside the method** (`arize_phoenix.py:241`, `:294`). This is
  done twice. Hoist to module level — the module already imports
  `opentelemetry` types via the existing OTEL plumbing.
- **The `_extract_chain_input`/`_extract_chain_output` dispatch
  order is undocumented as a contract.** The order matters
  (e.g. tool-call shape is checked first because `messages` may
  also be present in tool-augmented agent traces). One paragraph
  in the docstring explaining "we check shape X before Y because
  Z" would future-proof this.
- **Image MIME resolver normalises `jpg → jpeg` but not other
  variants** (`:537`). What about `JPEG`, `image/JPG`, `webp`,
  `gif`? Either enumerate the supported set or use the
  `mimetypes.guess_type`-derived list.

## Verdict

**request-changes** — the fixes are individually correct and
several address real production bugs (image rendering, double
token counting). But shipping ~640 lines of new code with zero
tests on a security-and-billing-adjacent observability path is
not mergeable as-is. Rebase into 4-5 smaller PRs, each with at
least one test, each with a real "Fixes #" link in the body.

## What I learned

OTEL/openinference span-attribute conventions are *strict path
contracts*: writing `IMAGE_URL.0` instead of
`MESSAGE_CONTENTS.0.MESSAGE_CONTENT_IMAGE.IMAGE_URL` doesn't
break the span, it just makes the renderer silently skip it.
That makes "the image just doesn't show up" a specifically
hard-to-debug class of bug. The lesson is: any time you're
writing OTEL attribute keys, they should come from a typed
constants module (which this code does — it imports
`MessageContentAttributes`, `ImageAttributes` from
`open_inference`), and the renderer-side schema should be linked
in a comment near the writer.
