# BerriAI/litellm#26744 — fix(azure_ai): use correct 'input_image' key for FLUX.2-pro image editing

- URL: https://github.com/BerriAI/litellm/pull/26744
- Head SHA: `ca6a34033459014cf3468ee3e203dc07c5913271`
- Size: +1 / −1, 1 file (`litellm/llms/azure_ai/image_edit/flux2_transformation.py`)
- Verdict: **merge-after-nits**

## Summary

One-line key rename in the Azure AI FLUX.2-pro image-edit request body: `"image"` → `"input_image"`. The Black Forest Labs FLUX.2 API silently ignores unknown keys and returns a successful response that just *omits the input image* from the edit, so the bug is high-severity user-impact (paid call returns wrong result) but invisible to error-monitoring.

## Specific issue / line

The change is at `litellm/llms/azure_ai/image_edit/flux2_transformation.py:115`:

```python
# Before
request_body: Dict[str, Any] = {
    "prompt": prompt,
    "image": image_b64,
    "model": model,
}
# After
request_body: Dict[str, Any] = {
    "prompt": prompt,
    "input_image": image_b64,
    "model": model,
}
```

## Specific issues flagged

### 1. **Missing test** — and this is exactly the bug class that needs a test

This is a `+1/-1` PR. It looks "obviously safe" but the bug being fixed is *itself* a regression that shipped because the request-body shape was never asserted. The fix without a test means the next refactor (someone consolidating `image_edit` payload builders, or a typo in a future provider) re-introduces the bug undetected. A 10-line test in `tests/test_litellm/llms/azure_ai/test_flux2_transformation.py` that calls `transform_image_edit_request(...)` and asserts `body["input_image"] == <expected_b64>` and `"image" not in body` is the load-bearing artifact, not the one-character rename. The project template explicitly says "at least 1 test is a hard requirement" — this PR should not skip that.

### 2. Source-of-truth link not in the PR description

PR body says "the Black Forest Labs API actually expects the key `input_image`" but doesn't link the BFL API reference. Add a permalink to the FLUX.2-pro `/edit` endpoint docs (or the Azure AI Foundry model card that documents the schema) so the next person triaging a regression here can verify the contract without re-reverse-engineering it from a 401/empty-edit failure.

### 3. Audit other Azure AI image transformations for the same bug class

If `flux2_transformation.py` got `image` vs `input_image` wrong, sibling transformations under `litellm/llms/azure_ai/image_edit/` and `litellm/llms/azure_ai/image_generation/` may have analogous schema-mismatch bugs (e.g., `"images"` vs `"input_images"`, `"reference_image"`, etc., which differ across BFL/SDXL/DALL-E variants). A 5-minute grep of sibling `transform_*_request` functions to confirm key names match the upstream provider docs would be a high-value follow-up. Worth at least a "checked, looks ok" note in the PR.

### 4. Confirm cassette / VCR fixtures aren't pinning the broken key

If there are recorded HTTP fixtures (cassettes / VCR YAML) anywhere under `tests/` that capture the `flux2` request, they're now stale (will record the new key on next re-record but assert against the old `"image"` key on replay). Worth a quick `rg '"image":' tests/ | rg -i flux` to confirm.

## Why merge-after-nits

The fix is correct and minimal for a real high-impact silent-failure bug. Item 1 (test) should land in the same PR not a follow-up — the bug class makes a regression test mandatory. Items 2-4 are low-effort due-diligence.
