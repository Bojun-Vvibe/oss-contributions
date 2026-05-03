# sst/opencode PR #25633 — refactor(cli): effectify provider commands

- Link: https://github.com/sst/opencode/pull/25633
- SHA: `4d374c863dd104792b23a4ab58bbcac97846a9da`
- Author: kitlangton
- Stats: +158 / −142, 2 files (`packages/opencode/src/cli/cmd/providers.ts`, `packages/opencode/src/cli/effect/prompt.ts`)

## Summary

Migrates the providers CLI command (list/login/logout) off the legacy `AppRuntime.runPromise` + raw `@clack/prompts` cancel-checks pattern onto the project's growing `effectCmd` + Effect-based `Prompt` wrapper. Plugin auth flows are also rewritten as `Effect.gen` programs that yield from `Prompt.select`/`Prompt.text` instead of awaiting clack directly and manually calling `prompts.isCancel`. Net change is roughly even (+158/-142), so this is a structural refactor, not a feature.

## Specific references

- `providers.ts` L1–L17: imports drop `AppRuntime` and the bare `@clack/prompts` namespace, and add `CliError, effectCmd, fail` plus the local `Prompt` wrapper and `errorMessage` helper. This is the right surface — `AppRuntime.runPromise` was the leaky bottom of the old code.
- L29–L37: `put` becomes `Effect.fn("Cli.providers.put")` with `Effect.orDie(auth.set(key, info))`. Naming the fn for tracing is the same convention used elsewhere in the cli/effect layer; using `orDie` here is reasonable since a credential write failure is genuinely unrecoverable in this command, but it does mean a transient FS error becomes a panic instead of a friendly `CliError`. Worth a follow-up to map this to a `fail("could not save credentials: …")` for parity with other write paths.
- L42–L45: `promptValue` collapses `Option.None → Effect.die(new UI.CancelledError())`. Note this is `Effect.die`, not `Effect.fail` — meaning Ctrl-C shows up as a defect in traces rather than a typed cancellation. The rest of the codebase already treats `UI.CancelledError` as a sentinel that the top-level handler recognises, so this works, but `Effect.fail(new UI.CancelledError())` would be more consistent with how cancellation is modelled elsewhere.
- L52–L56: `cliTry` wraps a `PromiseLike<Value>` and produces a `CliError` with `message + errorMessage(error)`. Two nits: (1) string concatenation without a separator means callers must remember to put a trailing space/colon in `message`; a small `: ${errorMessage(error)}` template would be safer. (2) `errorMessage` from `util/error` typically already prefixes — easy to double-prefix accidentally.
- L58–L97: `handlePluginAuth` is the meatiest change — old code did `process.exit(1)` on unknown method which bypassed the Effect runtime. The new path correctly `yield* fail(...)` so the error propagates through the runtime and any finalizers run. Good fix.
- L99–L103: `Effect.sleep("10 millis")` replaces `new Promise(r => setTimeout(r, 10))`. This 10ms sleep is undocumented in both old and new code — it looks like a clack rendering hack. A one-line comment explaining why it's needed would prevent future removal.
- L109–L123 (the truncated select/text block): the per-prompt cancel handling is now uniform via `promptValue` instead of two near-identical `if (prompts.isCancel(value)) throw new UI.CancelledError()` branches. Real win.

## Verdict

verdict: merge-after-nits

## Reasoning

This is a clean continuation of the ongoing cli-effectification work — the structural diff is correct, the cancel/error model is more uniform, and the only behaviour change (replacing `process.exit(1)` on unknown method with a typed failure) is an improvement. Nits are stylistic: prefer `Effect.fail` over `Effect.die` for `UI.CancelledError`, add a `: ` separator in `cliTry`'s message composition, and either downgrade `Effect.orDie(auth.set …)` to a typed failure or document why a credential-save failure must be a defect. Tests already in place (`bun run test test/provider/provider.test.ts` per PR body) cover the primary list/login/logout flows.
