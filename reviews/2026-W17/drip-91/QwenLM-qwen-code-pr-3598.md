---
pr: 3598
repo: QwenLM/qwen-code
sha: 2f6b406b9d99332f8165733ce5c89116f70b675f
verdict: merge-after-nits
date: 2026-04-27
---

# QwenLM/qwen-code #3598 — feat(cli): add --json-schema for structured output in headless mode

- **Head SHA**: `2f6b406b9d99332f8165733ce5c89116f70b675f`
- **URL**: https://github.com/QwenLM/qwen-code/pull/3598
- **Size**: medium (739/2 across 13 files)

## Summary
Adds a `--json-schema '<json>'` / `--json-schema @path/to/schema.json` flag to the headless mode (`qwen -p`). The user-supplied JSON Schema is registered as the parameter schema of a synthetic `structured_output` tool that the model must call to deliver its final answer. The session terminates on the first successful call; the validated payload surfaces on the result message as both a JSON-stringified `result` and a structured `structured_result`.

## Specific findings
- `packages/cli/src/config/config.ts:172-203` — `schemaRootAcceptsObject()` walks the schema root looking for `type: "object"` (or `type: ["object", ...]`), and recurses into root-level `anyOf`/`oneOf`. Correctly leaves `allOf` alone with a comment saying so. The "no narrowing → lenient default of `true`" at `:202` is the right call: a bare `{}` schema accepts anything, and rejecting it would block legitimate "give me back any object" use cases.
- `config.ts:208-269` — `resolveJsonSchemaArg()` does the right error funnel: read failure, JSON parse failure, non-object root, and root-doesn't-accept-objects all become `FatalConfigError` at CLI parse time. This is the **critical correctness property of the PR** — the body explicitly notes that the lenient runtime `SchemaValidator.validate` silently skips unsupported schemas for MCP compat, so without `compileStrict` here a malformed schema would no-op and quietly accept anything. Good design.
- `config.ts:251-262` — the rationale comment for why root must accept objects ("All function-calling APIs … require tool arguments to be a JSON object") is exactly the right level of detail to leave in the codebase. Future readers will not have to rediscover this.
- The "rejected when combined with `-i` / `--prompt-interactive`" check is mentioned in the PR body but I'd want to verify the actual yargs `.check()` is wired in — wasn't in the visible portion of the diff. Worth a reviewer pin.
- The PR body claims **23 unit tests** across `SyntheticOutputTool`, `resolveJsonSchemaArg`, `SchemaValidator.compileStrict`, and `BaseJsonOutputAdapter.buildResultMessage`. That coverage shape (cli-side schema parsing + core-side strict compile + adapter emission) is the right partition. The "12160/12164 core, 8848/8862 cli" raw counts in the PR body are not very useful for reviewers; what would be useful is a list of any tests that had to be updated for this change (none claimed, which is a positive signal).
- The "schema compatibility varies per provider" caveat in the unchecked manual test plan is real — Gemini's JSON-Schema dialect, OpenAI's, and Anthropic's tool-use schema all differ in supported keywords. The synthetic-tool path delegates that to the existing tool-call adapter so it should mostly work, but worth a `--json-schema --strict-providers` follow-up flag (or a runtime warning when the schema uses keywords known to be lossy on the active provider) — not blocking for this PR.
- Bundling 13 files for a CLI flag is on the larger side; that's because the synthetic tool, the Ajv strict-compile helper, the result-envelope plumbing, and three separate test files all need to change in lockstep. The shape is justifiable.

## Risk
Medium. The code path is well-tested and fails fast on bad schemas, but provider-side schema compatibility is the unmodeled risk: a schema that compiles fine in Ajv may be silently truncated when sent to a provider that doesn't support `oneOf` in tool params. The "session terminates on the first successful call" semantics also need a clear error path when the model never calls the tool — the body says "fails with a clear error and non-zero exit code", verify that's not just a generic stack trace.

## Verdict
**merge-after-nits** — confirm the yargs-side rejection of `--json-schema` + `-i` is actually wired in (not just claimed), and add one end-to-end test showing the "model never calls structured_output → exit code N" path with a real exit-code assertion.
