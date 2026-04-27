# sst/opencode#24618 — fix: guard against undefined MCP tool output causing output.split crash

- PR: https://github.com/sst/opencode/pull/24618
- Head SHA: `c11a6d47`
- Diff: +3 / -1 across `session/message-v2.ts` and `tool/truncate.ts`
- Closes #24402

## What it does
Three layered guards against an MCP tool returning a result with no text content (e.g. only images), which previously crashed the app with `TypeError: undefined is not an object (evaluating 'output.split')`:

1. `message-v2.ts:321` — `truncateToolOutput` early-returns `""` when text is falsy.
2. `message-v2.ts:857` — call site uses `?? ""` defensively when forwarding `part.state.output`.
3. `tool/truncate.ts:87` — `Truncate.output` early-returns `{ content: "", truncated: false }` when `text == null`.

## Strengths
- Defense in depth: the call-site `?? ""` plus the function-internal guards mean any future caller that forgets the coalesce won't crash. That's the right tradeoff for hot-path code that ingests untrusted MCP outputs.
- `text == null` (loose equality) in `truncate.ts:87` correctly catches both `null` and `undefined`, which matches the actual failure mode (TS type says `string` but MCP response makes it `undefined` at runtime).

## Concerns
- The schema in `message-v2.ts` still types `part.state.output` as a non-optional `string`. With this fix the runtime is safe, but the schema lies — a follow-up that loosens the schema to `Schema.optional(Schema.String)` (or coerces in the decoder) would prevent the same class of bug from recurring elsewhere. As written, future code that destructures `output` will hit the same trap.
- `if (!text) return ""` in `truncateToolOutput` collapses the empty-string case too. That's almost certainly fine (truncating `""` returns `""` anyway), but it's a behavior change worth a one-line comment so a future reader doesn't "fix" it back.
- No regression test. An MCP tool fixture that returns `{ content: [{ type: "image", ... }] }` and asserts no crash would cost ~10 lines and lock in the contract.

## Verdict
**merge-after-nits** — ship the guards immediately (they unblock real users); file a follow-up to fix the schema and add a fixture-based regression test.
