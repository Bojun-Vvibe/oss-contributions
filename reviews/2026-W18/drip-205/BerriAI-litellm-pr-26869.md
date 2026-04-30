# Review: BerriAI/litellm#26869 — feat: implement Litellm-pgvector OpenAI-specific RAG ingestion + public registry

- PR: https://github.com/BerriAI/litellm/pull/26869
- Author: sujal011 (Sujal Bhavsar)
- headRefOid: `eb2270e8db4c17447a831cc2474e3b0a993a0dcb`
- Files: `litellm/rag/ingestion/openai_ingestion.py` (+5/-5),
  `litellm/rag/main.py` (+1/-0),
  `litellm/types/utils.py` (+1/-0),
  `litellm/utils.py` (+1/-1),
  three new tests (+120) replacing one deleted test (-297)
- Verdict: **request-changes**

## Analysis

The functional intent is fine: register `pg_vector` as an alias for `OpenAIRAGIngestion`
(rag/main.py:55), parameterize the previously-hard-coded `custom_llm_provider="openai"` calls in
`OpenAIRAGIngestion.store()` (openai_ingestion.py:91, 109, 120) so the same code path can talk to
an OpenAI-compatible pgvector backend, and extend the file/vector-store provider gates to accept
`PG_VECTOR` (`types/utils.py:3343` adds it to `OPENAI_COMPATIBLE_BATCH_AND_FILES_PROVIDERS`,
`utils.py:8864` extends the vector-store-files config gate). That's all reasonable plumbing for a
new OpenAI-compatible backend.

**Blocking concern 1 — large unrelated test deletion.** The diff deletes
`tests/test_litellm/llms/pg_vector/vector_stores/test_pg_vector_transformation.py` (297 lines)
and adds `tests/test_litellm/vector_stores/test_pg_vector_transformation.py` (54 lines). Even
allowing that the test was relocated to align with the rest of the `vector_stores/` test layout,
the new file is materially smaller — the original covered `validate_environment` happy/error paths,
URL formatting with/without trailing slashes, and presumably more. The PR description doesn't
explain what coverage was intentionally dropped vs. what got moved. Either restore parity or
explicitly call out which assertions were redundant.

**Blocking concern 2 — provider field plumbing.** `OpenAIRAGIngestion.store()` now reads
`self.custom_llm_provider` (openai_ingestion.py:92, 110, 122) but the diff does not show
`OpenAIRAGIngestion.__init__` (or its base class) being updated to *set* that attribute when
constructed via the registry as `pg_vector`. If `RAG_INGESTION_REGISTRY["pg_vector"]` resolves to
the same `OpenAIRAGIngestion` class with no override, then `self.custom_llm_provider` will likely
default to `"openai"` regardless of the registry key, defeating the entire point of the change.
Need to see the class init or factory wiring before this can merge — either the diff is incomplete
or the new test files are not actually exercising the pg_vector path end-to-end.

**Blocking concern 3 — test naming vs. intent.** The two new test files
(`test_vector_store_create_provider_logic.py`, `test_vector_store_registry.py`) plus the slimmer
transformation test add 120 lines but the PR body only says "Added testing"; there's no description
of what they assert. For a feature that adds a new provider to a public registry, I'd expect at
minimum: (a) a test that creating an ingestion job with `pg_vector` actually selects the pg_vector
endpoint/api_base, (b) a test that the provider field flows from registry resolution into the
underlying `vector_store_acreate` / `acreate_file` calls.

The change conceptually belongs in the codebase but as written it has too many unanswered questions
about provider-field threading and test coverage to land. Request changes; ask the author to (a)
explain the test deletion, (b) show where `self.custom_llm_provider` is set for the pg_vector
registry path, and (c) add an end-to-end registry → store call assertion.
