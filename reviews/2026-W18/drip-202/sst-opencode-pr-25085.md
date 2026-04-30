# sst/opencode #25085 — feat(cursor): add Cursor provider integration

- **Author:** Knifelf
- **SHA:** `a33fdbb`
- **State:** OPEN
- **Size:** +161 / -9 across 6 files (provider.ts +109, providers.ts +26, llm.ts +21, +.gitmodules / submodule / popular list)
- **Verdict:** `request-changes`

## Summary

Adds a `cursor` provider to the OSS surface — wires `CURSOR_API_KEY` into a new
`provider.cursor` Effect arm at `packages/opencode/src/provider/provider.ts:825-905`
that constructs a `CursorCloudAgentLanguageModel`, lazily discovers model IDs via
`fetchCursorModelIds`, falls back to a hard-coded `defaultCursorModelsDevProvider()`
synthesized provider record at `:1170-1174` if `models.dev` doesn't carry an
entry, and disables the OpenCode local tool surface for Cursor sessions at
`packages/opencode/src/session/llm.ts:200-206` (`tools = {}` plus a system-prompt
note that "Cursor Cloud runs in Cursor's VM with Cursor-side tools").

## Reasoning

Three concrete blockers before this can land as drawn:

1. **`.gitmodules` adds a git submodule pinning `https://github.com/cursor/cookbook.git`
   at `.cursor/cookbook` that is not used at build, runtime, or test time** — it's
   purely vendor docs. A submodule on a high-traffic OSS repo is a permanent
   user-facing tax (every clone adds latency and a possibly-stale pin) for zero
   functional value. Drop both `.gitmodules` and `.cursor/cookbook` and link the
   cookbook from the README/docs page instead.

2. **The `s.models.set(key, language)` per-session cache is bypassed only for the
   cursor provider** at `provider.ts:1681,1695-1697` (`if (model.providerID !==
   "cursor")` guards both read and write). The PR body doesn't say why. If the
   reason is "the Cursor SDK keeps stateful per-call session info on the language
   model object," that reason needs a comment at the bypass site — otherwise this
   reads as a workaround for a bug that will silently regress the moment someone
   refactors the cache.

3. **The capabilities matrix at `provider.ts:866-878` hard-codes `toolcall: false,
   reasoning: true, attachment: true, image: true, pdf: true`** for every
   discovered Cursor model regardless of what `fetchCursorModelIds` returns. This
   is the same shape `defaultCursorModelsDevProvider()` would presumably already
   express. Either feed capabilities through `models.dev` (preferred) or read them
   from the Cursor `/models` response if the API exposes them; "every Cursor model
   supports image+pdf input but no tool calls" is almost certainly false on at
   least one model id.

Smaller nits: the `apiKey` lookup uses `??` between authRecord and env var
(`provider.ts:828`) so a saved-but-empty-string auth entry will swallow the env
var; `cost: { input: 0, output: 0, ... }` on every discovered model means the
spend-tracking surface will show $0 for every Cursor turn (better to mark
`undefined` so downstream tells the user "unknown" rather than "free"); no
regression test for the tools-disabled gate at `llm.ts:200-206` (a future
provider id check refactor will silently re-enable tool dispatch into the Cursor
VM and the model will produce tool calls that go nowhere).
