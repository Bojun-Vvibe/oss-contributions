# QwenLM/qwen-code #3598 — feat(cli): add `--json-schema` for structured output in headless mode

- Head SHA: `41bb7b3478b7d8b8267e173455bcff3a9166de9d`
- Author: wenshao
- Diff: +1580 / −15 across 16 files
- Author's test result: 12160/12164 core + 8848/8862 CLI passing (pre-existing skips), full cross-provider smoke matrix in PR body

## Verdict: `merge-after-nits`

## What it does

Adds a `--json-schema '<json>'` / `--json-schema @path/to/schema.json` flag to the headless `qwen -p` mode. The flag registers a synthetic `structured_output` tool whose parameter schema *is* the user-supplied JSON Schema; the model must call that tool to deliver the final result, and the session terminates on the first successful (schema-valid) call.

The `--json-schema` value flows through three filters before becoming a tool:
1. **Yargs check** rejects combinations that don't make sense (`-i` / `--prompt-interactive` and `--input-format stream-json`).
2. **`resolveJsonSchemaArg`** parses inline JSON or reads `@file`, then validates the parsed value as a satisfiable-by-object JSON Schema using a strict Ajv compile.
3. **`SchemaValidator.compileStrict`** (new) compiles the schema with Ajv strict mode and returns the first error string, in contrast to the existing lenient `SchemaValidator.validate` which silently no-ops on compile failure (kept that way for MCP compat).

Validated payloads land on the result message as both `result` (JSON-stringified, for `--output-format text` piping) and `structured_result` (parsed, for `--output-format json` / `stream-json`).

## What's genuinely well-thought-through

### 1. The `schemaRootAcceptsObject` check (`packages/cli/src/config/config.ts:175-265`) does the right thing for the right reason

JSON Schema applies sibling keywords conjunctively. The author's comment at lines 175-189 explicitly walks through *why* a schema like `{type:"object", anyOf:[{type:"string"}]}` is unsatisfiable: `type` requires object AND `anyOf` requires string — and so the check walks all four of `type`, `anyOf`, `oneOf`, `allOf` rather than returning on the first hit. That's the kind of "show your work" comment that makes the difference between "looks reasonable" and "is reasonable" for a non-trivial schema-walker.

The handling is also correct on subtleties most implementations miss:
- `anyOf` / `oneOf` empty arrays return `false` (they're unsatisfiable per spec, lines 234-236), not "no constraint".
- `true` / `false` subschemas (draft-06+) are honoured (lines 220-228).
- Bare root `$ref` without sibling `type:"object"` is rejected because the resolver can't follow remote/recursive refs (lines 188-196). The author explicitly chose "reject" over "half-resolve" with the reasoning: *"getting it half-right is worse than rejecting"* — that's the right call for a security-adjacent input.
- `not.type` containing `"object"` is rejected as best-effort (lines 263-274) with explicit acknowledgment that arbitrary `not` schemas (e.g. `not:{anyOf:[...]}`) fall through to Ajv at runtime.

### 2. Strict-vs-lenient compile is a deliberate split, not an accident

`SchemaValidator.compileStrict` is new and is **only** called from `resolveJsonSchemaArg`. The lenient `SchemaValidator.validate` continues to no-op on compile failure for MCP tool schemas (which can be exotic / vendor-specific). The PR description calls this out explicitly: *"`--json-schema` is explicit user intent, so surface a bad schema here rather than letting it silently no-op later."* Right tradeoff.

### 3. Yargs-level rejection of incompatible combinations with explicit reasoning

`packages/cli/src/config/config.ts:817-832` rejects `--json-schema` + `-i` *and* `--json-schema` + `--input-format stream-json`. The `stream-json` rejection comment is the load-bearing one:

> The "first valid structured_output call ends the session" contract assumes a single one-shot prompt. Stream-json input keeps the process open waiting for more protocol messages, so terminating on the first call would silently drop subsequent prompts.

That's the correct contract analysis — without it the run would race to whichever message wins, and the silent-drop failure mode would be invisible to the operator. The explicit "we don't reject the no-prompt case here because stdin pipe is still valid" comment at lines 833-840 is also exactly right and prevents a future contributor from "tightening" the validation in a way that breaks `cat prompt.txt | qwen --json-schema '...'`.

### 4. Real cross-provider validation, including per-model reliability table

The PR body includes a 7-row reliability matrix (model × provider path × neutral-prompt success rate). The author noticed that smaller models (`qwen3.5-flash`) need an explicit "you MUST call the structured_output tool" directive to be dependable and documents this as a known caveat. That's the difference between "I tested it" and "I understand the failure modes."

## Nits worth raising before merge

### 1. The 280-line `resolveJsonSchemaArg.test.ts` is good but `schemaRootAcceptsObject` itself isn't directly unit-tested

The test file at `packages/cli/src/config/jsonSchemaArg.test.ts` (head visible in the diff) covers the public `resolveJsonSchemaArg` surface — but it tests the schema-root check *transitively* via the `--json-schema root must accept object-typed values...` error string. The 9-branch logic inside `schemaRootAcceptsObject` (type / $ref / const / enum / anyOf / oneOf / allOf / not / true-false subschema) deserves direct table-driven tests. A regression that, say, breaks the `not.type` branch would only be caught if someone happens to write a `--json-schema` test with a `not:{type:"object"}` shape.

### 2. `schemaRootAcceptsObject` doesn't handle root `$ref` with `type:"object"` *as a fragment*

At line 195: `if (typeof schema['$ref'] === 'string' && !typeIncludesObject) return false;`. The fix is correct for the bare-`$ref` case, but a schema like `{$ref:"#/$defs/MyObject", type:"object"}` is valid JSON Schema (sibling keywords) and the check passes (because `typeIncludesObject` is true) — but the actual `$ref` target could still be a non-object. This is documented as a known limitation in the comment at lines 192-196, so it's a deliberate tradeoff, but the comment should also explicitly note the `type:"object"` opt-out: *"if you sibling `type:'object'` next to your `$ref`, we trust you and skip the resolution; if the ref turns out to point at a string, Ajv will catch it at validation time but the synthetic tool will already be registered."*

### 3. The new `_litellm_batch_s3_bucket_overrides`-style "private control field on user-facing TypedDict" pattern is **not** what this PR does — but the synthetic-tool registration deserves a similar audit

I want to confirm: is the synthetic `structured_output` tool registered on a Tools registry that's also reachable from non-headless flows (e.g. interactive `/tools` listing)? If so, a user running an interactive session right after a headless `--json-schema` run might see a stale `structured_output` tool. The Yargs guard rejects the *combination* but not the *cross-session leak*. Worth a quick code-search before merge — this is the kind of thing that's easy to miss because the tool registration code may live in a separate module than the diff touched.

### 4. `resolveJsonSchemaArg` reads files with `fs.readFileSync` (`config.ts` line 290 area) — no size cap

A `--json-schema @/dev/zero` or `--json-schema @some-1GB-file.json` will OOM the process before the JSON.parse error fires. A 1 MiB cap would cover any realistic schema (the spec doesn't have a meaningful upper bound but real schemas are kilobytes) and would prevent a silly local DoS.

### 5. `payload = trimmed` at the inline-JSON branch (around line 297)

The trimmed string can still contain trailing junk after a valid JSON object (e.g. `--json-schema '{"x":1} extra'`). `JSON.parse` will reject this with a useful error, so this is fine — just noting it for completeness. `JSON.parse` is the right validator here; no need to add a separate "is fully consumed" check.

### 6. PR description mentions `kimi-for-coding` provider in the validation matrix

That's a legitimate provider name (`api.kimi.com/coding/`) — flagged purely so future readers don't mistake it for one of the assistant-product names that get filtered. No action.

## Why not request-changes

The architecture is right (synthetic tool, terminate-on-first-call, parse-time vs. runtime validation split, Yargs-level combination guards), the schema-root check has the rare quality of being *both* spec-correct *and* commented with the spec reasoning, and the cross-provider validation matrix shows the author understood the failure modes. The five nits above are coverage gaps and one OOM hardening — none change the design.

## Citations

- `packages/cli/src/config/config.ts:175-265` — `schemaRootAcceptsObject` walker with sibling-keyword reasoning
- `packages/cli/src/config/config.ts:280-365` (approx) — `resolveJsonSchemaArg` parse + strict-compile path
- `packages/cli/src/config/config.ts:676-688` — `--json-schema` Yargs option declaration
- `packages/cli/src/config/config.ts:817-832` — combination guards (`-i`, `stream-json`)
- `packages/cli/src/config/jsonSchemaArg.test.ts:1-280` (head shown in diff) — public-surface tests for `resolveJsonSchemaArg`
