# Review: QwenLM/qwen-code #3769 — fix(core): isolate fast model side queries

- Repo: QwenLM/qwen-code
- PR: #3769
- Head SHA: `6371959e30571e74cff29da0cea778d99eeec61a`
- Author: LaZzyMan
- Size: +168 / -58 across 18 files

## What it does
Introduces per-model `ContentGenerator` caching in `BaseLlmClient` so that
"side queries" (rename, session title, recap, tool-use summary,
next-speaker check, side-query utility, subagent generator,
relevance selector) routed to a distinct fast model (`getFastModel()`)
get their own configured generator instead of reusing the main model's
generator with just the `model` field overridden. Also forces
`thinkingConfig: { includeThoughts: false }` on the rename command's
title generation.

## File-level notes

**`packages/core/src/core/baseLlmClient.ts` @ L66–105 (head `6371959`)**
```ts
private readonly contentGeneratorsByModel = new Map<string, ContentGenerator>();

private async getContentGeneratorForModel(model: string): Promise<ContentGenerator> {
  if (model === this.config.getModel()) return this.contentGenerator;
  const cached = this.contentGeneratorsByModel.get(model);
  if (cached) return cached;
  const generatorConfig = buildAgentContentGeneratorConfig(
    this.config, model,
    { authType: this.config.getContentGeneratorConfig().authType! },
  );
  const generator = await createContentGenerator(generatorConfig, this.config);
  this.contentGeneratorsByModel.set(model, generator);
  return generator;
}
```
- The non-null assertion `authType!` is risky — `getContentGeneratorConfig()`
  may legitimately return a partial config in some auth flows. A guarded
  `if (!authType) throw new Error(...)` with a clear message would be
  safer than crashing in `buildAgentContentGeneratorConfig`.
- The cache has unbounded growth keyed by model name. In practice the
  set of models is tiny (main + fast = 2), so this is fine, but a
  one-line comment noting the bound would prevent future confusion.
- No invalidation: if the user runs `/auth` mid-session and the auth
  type changes, cached generators may hold stale credentials. Consider
  invalidating on a config event, or document that re-auth requires a
  restart.

**`packages/cli/src/ui/commands/renameCommand.ts` @ L82**
```ts
+ thinkingConfig: { includeThoughts: false },
```
- Correct: titles don't need a "thinking" preamble and excluding it
  reduces tokens/latency. Test at `renameCommand.test.ts` L223 asserts
  this is set.

**`packages/core/src/utils/sideQuery.ts` (+5/-1)** + new tests (+27)
- Centralizes side-query routing through a chokepoint, which is the
  whole point of the PR. Good architectural move.

**`packages/core/src/services/sessionRecap.ts`,
`sessionTitle.ts`, `toolUseSummary.ts`,
`memory/relevanceSelector.ts`, `core/geminiChat.ts`, etc.**
- One-line additions each; from the diff hunks they appear to thread
  `model` through to the new `getContentGeneratorForModel` path. Test
  files updated symmetrically (each `.test.ts` adds 1–2 lines for the
  new mock surface, e.g. `getFastModel: vi.fn().mockReturnValue(undefined)`).

## Risks
- The `authType!` non-null assertion is the main correctness risk.
- Caching without invalidation is a latent issue but unlikely to bite
  users in normal flows.
- 18 files touched but most are 1–3 line edits; the change is broader
  than it is deep.

## Verdict: `merge-after-nits`
Replace the `authType!` non-null assertion with a guarded throw, add a
brief comment on the cache bound, and this is good. The architectural
move (chokepoint + per-model generator cache) is the right shape.
