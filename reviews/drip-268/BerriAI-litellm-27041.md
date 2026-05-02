# BerriAI/litellm #27041 — fix(vertex-ai): set response=null on batch error entries per OpenAI spec

- **Head SHA:** `946dfb63c7896c317b99cfc4760877460f7d2cff`
- **Files:** `litellm/llms/vertex_ai/files/transformation.py`, `tests/test_litellm/llms/vertex_ai/files/test_vertex_ai_files_transformation.py` (+8/-30)
- **Verdict:** `merge-as-is`

## Rationale

Tightens the Vertex-AI batch output transform to match OpenAI's batch output schema, which requires `response: null` on error entries (the error detail lives in the top-level `error` object). Before this PR, `_transform_single_vertex_batch_output_to_openai` (`transformation.py:731`) was building a fake `response.body.error` envelope with `status_code: 400` for upstream Vertex errors and a parallel one with `status_code: 500` in the transformation-failure `except` branch — which caused downstream tooling that expects OpenAI's exact shape (e.g. clients that key off `response is None` to route into the error path) to misroute errors as successful 4xx responses.

The cleanup is correct and minimal: both branches now emit `"response": None` and a top-level `error` dict with just `code` and `message`, dropping the redundant `type` key (`error.code` already carries the discriminator). The test (`test_vertex_ai_files_transformation.py:434`) is updated in lockstep — it asserts `result["response"] is None` and `result["error"]["code"] == "vertex_ai_error"`, which is exactly the shape clients should branch on.

Two minor observations, neither blocking: the success path (lines 754–772 in the original file, untouched here) still emits a populated `response` block, so the null-on-error contract is consistent. And dropping `error.type` is an OpenAI-spec match — OpenAI's batch output uses `error.code` + `error.message`, not `error.type`. Risk is low because this only changes the error-entry shape and the existing test suite gates the new contract.
