# google-gemini/gemini-cli #26277 — docs(sdk): add JSDoc to all exported interfaces and types

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26277
- **Head SHA**: `22001650ede2ec1ded343d62e0c34ec88a8146dc`
- **Author**: fauzan171
- **Size**: +343 / −0, 7 files

## Files changed

- `packages/sdk/src/agent.ts` (+38)
- `packages/sdk/src/fs.ts` (+7)
- `packages/sdk/src/session.ts` (+40)
- `packages/sdk/src/shell.ts` (+12)
- `packages/sdk/src/skills.ts` (+6)
- `packages/sdk/src/tool.ts` (+80)
- `packages/sdk/src/types.ts` (+160)

All adds, no deletes — this is a pure documentation PR.

## Specific-line review

- `agent.ts` — class-level JSDoc on `GeminiCliAgent` includes a runnable
  `@example` block (lines 19-32 in diff) showing the full
  `new → session() → initialize() → sendStream()` lifecycle. This is the
  highest-leverage doc add in the PR; SDK consumers landing in
  `agent.ts` first now have an end-to-end working snippet without
  leaving the file.
- `agent.ts:48-54` — `session()` doc clarifies the `{ sessionId }`
  override semantics and that one is auto-generated otherwise. Matches
  the implementation (`options?.sessionId || createSessionId()`).
- `agent.ts:59-67` — `resumeSession()` doc correctly calls out the
  `Error` thrown when no sessions exist or the ID is missing.
- `fs.ts:11-17` — `SdkAgentFilesystem` JSDoc states the read/write
  policy contract: reads return `null` on deny, writes throw. This is
  exactly the kind of "asymmetric error model" doc that SDK users
  routinely get wrong; documenting it in the type itself is the right
  place.
- `types.ts` is +160 lines on its own and is presumably where the bulk
  of the public-surface JSDoc lives. Pure additive comments in a
  `.d`-shape file have zero runtime risk.

## Risk

Effectively zero. Pure doc-comment additions to a TypeScript SDK.
No runtime behavior change; CI just needs `tsc` to still pass (JSDoc
syntax errors would surface as TS errors).

## Things to consider (non-blocking nits)

- The `@example` in `agent.ts` does not show the `await session.close()`
  / cleanup pattern. If sessions hold handles (file watchers, child
  processes) this would be a footgun for a copy-paster. Worth one
  sentence after the `for await` block.
- 343 lines of pure JSDoc in one PR is a lot to spot-check; a maintainer
  should at least scan `types.ts` to confirm none of the comments
  contradict the actual types (e.g. nullable vs. required fields).

## Verdict

**merge-after-nits** — high-value documentation PR, no behavior risk,
but the maintainer should skim `types.ts` for accuracy and ideally ask
for a cleanup snippet in the `@example`.
