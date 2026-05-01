# sst/opencode#25214 — fix(httpapi): omit absent optional response fields

- Link: https://github.com/sst/opencode/pull/25214
- Head SHA: `5afc301ddd7985d54c76111dcdb03c13db08d203`
- Files: `project/project.ts` (+10/−10), `provider/auth.ts` (+21/−19), `provider/provider.ts` (+6/−6), `test/server/httpapi-json-parity.test.ts` (+81/−0)

## Notes
- Replaces `Schema.optional(...)` with `optionalOmitUndefined(...)` across every public response schema field that the legacy server omits when absent — `ProjectIcon.{url,override,color}`, `ProjectCommands.start`, `ProjectTime.initialized`, `Project.Info.{vcs,name,icon,commands}` at `project/project.ts:26-50`, the auth `TextPrompt.{placeholder,when}`/`SelectOption.hint`/`SelectPrompt.when`/`Method.prompts` at `provider/auth.ts:18-45`, and `ProviderCost.experimentalOver200K` / `ProviderLimit.input` / `Model.{family,variants}` / `Info.key` at `provider/provider.ts:875-918`. The fix is exactly at the schema-encode boundary so every endpoint that serializes via these schemas inherits the legacy-parity shape — one place, every consumer benefits.
- The hand-rolled adapter at `provider/auth.ts:138-159` flips from "always set the key, possibly to `undefined`" to spread-conditional `...(method.prompts && { prompts: ... })` / `...(prompt.when && { when: prompt.when })` / `...(prompt.placeholder && { placeholder: prompt.placeholder })`, which matches what `optionalOmitUndefined` produces at the schema layer — the two layers now agree on "absent means key not present" rather than disagreeing across the seam.
- The 81-line test arm `matches legacy JSON shape for safe GET endpoints` at `httpapi-json-parity.test.ts:93+` exercises five endpoints (`global.health`, `instance.path`, `instance.vcs`, `instance.vcsDiff?mode=git`, `instance.command`) under both legacy + HttpApi backends with a mocked `mcp.demo` (disabled) so the snapshot covers a realistic config shape, not just a trivial empty case — this is the right anti-behavior test ("legacy and HttpApi must produce byte-equal JSON for these GETs") that pins the contract so a future schema-shape refactor breaks the test rather than silently breaking SDK consumers.

## Nits
- The spread-conditional pattern at `auth.ts:139-159` repeats the `...(x && { x })` shape four times in 20 lines — extracting an `omitIfNullish<T>(key, value)` helper would dedupe and prevent a future contributor from adding a fifth field with `key: prompt.foo` (always-set) shape and silently breaking parity.
- The new `optionalOmitUndefined` import is added to three files but the `Schema.optional` call sites were all converted in lockstep — worth a one-liner repo-wide grep assertion in CI ("no `Schema.optional` in any file ending in `.ts` under `src/{project,provider,server}`") to lock the convention so the next added field doesn't drift.
- The test exercises only GET endpoints; the body-bearing POST/PATCH endpoints that consume these schemas (e.g. `PATCH /global/config`) are not covered for the inverse direction (request decode-then-encode round-trip). The schema is bidirectional so a `pickFirstThenEncode` round-trip arm would catch a future asymmetric `optionalOmitUndefined` decode behavior.
- Truthiness guard `prompt.placeholder && { placeholder: prompt.placeholder }` at `:155` will drop empty-string placeholders silently; if `""` is ever a meaningful "no-hint placeholder" sentinel for a select prompt, prefer `prompt.placeholder !== undefined` to match the schema's omit-only-undefined semantics exactly.

**Verdict: merge-after-nits**
