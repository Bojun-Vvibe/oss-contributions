# google-gemini/gemini-cli#26147 — test(evals): add EvalMetadata JSDoc annotations to older tests

- PR: https://github.com/google-gemini/gemini-cli/pull/26147
- Head SHA: `4412fbdf84872b0c1fd3a05da5c0504469c70648`
- Author: akh64bit
- Diff: +85/-0 across `evals/ask_user.eval.ts`, `evals/automated-tool-use.eval.ts`, `evals/frugalReads.eval.ts`, `evals/plan_mode.eval.ts` (and likely siblings in the same `evals/` dir)

## What changed

Pure test-metadata annotation pass: adds JSDoc `@group`/`@scenario`/`@maintainer` tags to existing eval test cases that pre-date the `EvalMetadata` convention. No functional change — every hunk inserts a comment block immediately above an existing `evalTest('USUALLY_PASSES', { ... })` or `askUserEvalTest('USUALLY_PASSES', { ... })` call. Examples:

- `evals/ask_user.eval.ts:34-39` — annotates four `askUserEvalTest` cases with `@group Communication / @scenario ask-user / @maintainer agent-team`.
- `evals/automated-tool-use.eval.ts:14-19, 106-111` — extends two existing JSDoc blocks (the `eslint --fix` and `prettier --write` cases) with `@group Core / @scenario tool-use / @maintainer agent-team`, *appending* to the prose comment rather than replacing it.
- `evals/frugalReads.eval.ts:16-21, 142-147, 217-222` — three cases get `@group Context / @scenario frugal-reads / @maintainer agent-team`.
- `evals/plan_mode.eval.ts:35-40, 56-61, 81-86, 130-135, 199-204, 238-243` — six cases get `@group Planning / @scenario plan-mode / @maintainer agent-team`.

## Observations

The change is mechanical and consistent: every block uses the same three-tag schema, the `@group` value matches the suite domain (Communication / Core / Context / Planning), and the `@scenario` value is the dash-cased file basename. That consistency is the whole load-bearing property of an annotation backfill — it keeps the metadata grep-able and parseable by whatever downstream `EvalMetadata` collector exists in the repo.

Two observations worth a one-line response from the author before merge:

1. **JSDoc block placement vs. existing prose.** `automated-tool-use.eval.ts` correctly *extends* an existing JSDoc block by appending the tags after the prose, preserving readability. The other three files always insert a *new* JSDoc block — even for cases that previously had no comment, which is fine — but for the first `askUserEvalTest` (`ask_user.eval.ts:34`) the block sits immediately under `function askUserEvalTest(...)` and could be misread as documenting the helper rather than the call. JSDoc-on-statement is a known soft spot of TS tooling. If `EvalMetadata` is collected statically by AST walk over the call expression, this is fine; if it's collected by JSDoc-symbol association, the helper-vs-call binding may be wrong. Worth a quick check that the consumer correctly attaches.

2. **`@maintainer agent-team` for all 13+ blocks.** That's a uniform value, which suggests the annotation is currently more of a placeholder than a routing signal. Fine for now, but if the schema is supposed to support multiple maintainers (`@maintainer foo, bar`) or per-file overrides, that should be documented in the `EvalMetadata` source so the next backfill PR doesn't diverge.

No code paths exercised; CI risk is essentially zero — eval tests still run with the same `'USUALLY_PASSES'` / `'ALWAYS_PASSES'` reliability tier. No banned content. The +85/-0 (deletion-free, comment-only) shape is exactly what makes this kind of cleanup a safe drip.

## Verdict

`merge-as-is`
