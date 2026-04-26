# BerriAI/litellm #26536 — fix(memory): jsonify metadata before Prisma writes on /v1/memory

- **Repo**: BerriAI/litellm
- **PR**: #26536
- **Author**: krrish-berri-2 (Krrish Dholakia)
- **Head SHA**: 9ce2176b2621ec5330352930a29208df02f27ab8
- **Base**: main
- **Size**: +265 / −21 across `memory_endpoints.py` (+25/−16) and the
  test file (+240/−5). Five new test cases + one rewrite of the
  prior explicit-null behavior.

## What it changes

Adds `_serialize_metadata_for_prisma` in
`litellm/proxy/memory/memory_endpoints.py:43-55` — a one-liner that
unconditionally `json.dumps`-encodes the `metadata` payload before
handing it to prisma-client-python. Three call sites
(`create_memory:304`, `upsert_memory` create-branch `:503`,
`upsert_memory` update-branch `:454`) now route through it. The mock
Prisma layer in the test file
(`tests/test_litellm/proxy/memory/test_memory_endpoints.py:91, 140`)
mirrors real Prisma's read-side behavior by `json.loads`-ing string
inputs back into Python values.

The semantics of `PUT /v1/memory/{key}` with `metadata: null` are
also changed: the previous "clear the column to SQL NULL" path is
replaced with "ignore the field entirely", and a `metadata: null`
payload with no other fields now returns 400 instead of an empty 200.

## Strengths

- Real bug fix: `prisma-client-python` rejects raw Python dicts/lists
  on a `Json?` column with `DataError`. The 5 regression tests
  (`test_create_memory_with_metadata_jsonifies_for_prisma`,
  `..._list_metadata...`, `test_put_memory_with_list_metadata...`,
  `test_put_memory_update_with_list_metadata...`,
  `..._string_metadata...`) each spy on `table.create`/`update` and
  assert `isinstance(sent_metadata, str)` — that's the right
  contract-level assertion (the bug was at the Prisma boundary, so
  the test asserts what's sent across the boundary).
- Bare-string case (`test_create_memory_with_string_metadata...`,
  `:404-435`) is the most subtle — Postgres `jsonb` rejects
  `hello` as invalid JSON but accepts `"hello"`. The PR catches it,
  the test catches it, and the docstring spells out exactly why.
- `model_fields_set` distinction (`upsert_memory:441`) is preserved:
  the endpoint still differentiates "field omitted" from "field sent
  as null". Only the *interpretation* of "field sent as null" changes
  (was: clear column; now: no-op).
- The new 400 in `test_put_memory_null_metadata_alone_returns_400` is
  a good UX detail — without it a `PUT {"metadata": null}` would
  silently succeed with no state change, which is a confusing API.

## Concerns / asks

- **Behavior change is documented only in a docstring**, not in a
  changelog. The prior `test_put_memory_explicit_null_metadata_clears_field`
  is renamed to `..._is_noop` and rewritten — that's a public-API
  semantics change for any caller that was using `metadata: null` to
  intentionally clear metadata. There is no migration note,
  `OldBehavior`/`NewBehavior` block in the PR description, or
  deprecation cycle. The justification (referencing
  `RobertCraigie/prisma-client-py#714`) is sound, but consumers who
  *were* successfully clearing metadata via the old code path
  (which would have been crashing with `DataError` for non-null,
  but might have worked for null in some Prisma versions) won't
  notice until they read the diff. A `## Breaking change` section
  in the PR description and a `WARN` log on `metadata: None` paths
  would help.
- The test mock `_filter`/`create`/`update` at
  `test_memory_endpoints.py:88-152` now does `json.loads` on
  string inputs — but only inside `create` and `update`. If
  `find_many` ever surfaces metadata that the mock stored as a
  string (e.g. via a code path that bypasses create/update),
  it'll come back as a string. Minor since today no such path
  exists, but worth a centralized helper in the mock so future
  CRUD methods don't drift.
- Three call sites repeat the same pattern
  (`if body.metadata is not None: data["metadata"] = _serialize...`).
  Consider folding the conditional into the helper itself
  (`_serialize_metadata_for_prisma_or_omit(data, body.metadata)`)
  so a future fourth call site can't forget the `is not None` guard.
- No test for `PUT /v1/memory/{key}` create-branch with `metadata`
  as a JSON scalar (number/bool). Lists, dicts, strings, and None
  are covered; `metadata: 42` and `metadata: true` are not, and
  the fix's `json.dumps` should handle them correctly — adding
  one parametrized case would close the gap cheaply.

## Verdict

**merge-after-nits** — fix is correct, regression coverage is solid
and the right shape. Asks: explicit breaking-change note for the
`metadata: null` semantics flip, parametrize the metadata-type tests
to cover scalars, and consider centralizing the `is not None` guard
in the helper.

## What I learned

prisma-client-python's `Json?` columns are a known sharp edge: there's
no `JsonNull`/`DbNull` sentinel
(`RobertCraigie/prisma-client-py#714`), so you can't write a true SQL
NULL through the typed client. The pragmatic workaround that the rest
of the litellm proxy uses — `json.dumps` everything for write, omit
the field to get SQL NULL, treat user-supplied null as no-op — is
ugly but consistent. Adopting that consistency here closes the bug
and keeps the proxy's behavior uniform across all `Json?` columns.
