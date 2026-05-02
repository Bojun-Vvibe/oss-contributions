# Review: sst/opencode #25410

- **PR:** sst/opencode#25410 — `fix(cli): strip leading slash from --command argument`
- **Head SHA:** `b2221a093fbad12291ffff3bc5074552bc83f531`
- **Files:** 1 (+2/-2)
- **Verdict:** `request-changes`

## Observations

1. **Latent NPE if `--command` is omitted** — `run.ts:635` becomes `command: args.command.replace(/^\//, "")`. The `command` option is declared optional (`type: "string"`, no `demand`), so when the flag is absent `args.command` is `undefined` and `.replace` throws `TypeError: Cannot read properties of undefined (reading 'replace')`. Previously the line passed `args.command` through directly, so `undefined` was a valid (no-command) input. Fix: `command: args.command?.replace(/^\//, "")` or guard explicitly.

2. **Whitespace not trimmed** — Users who type `--command "/test"` from a shell history may accidentally include trailing space; `replace(/^\//, "")` only handles a single leading slash. A more forgiving normalization is `args.command?.trim().replace(/^\/+/, "")` (also handles `//test` typo). Minor but consistent with other CLI normalization in the same file.

3. **Help-text update is good** — `"the command to run (without leading slash), use message for args"` makes the contract explicit; combined with the strip this is friendlier than either alone.

4. **No regression test** — A 2-line fix with a documented bug (the NPE) really does want a unit test asserting both `/test` and `test` route to the same registry lookup, and that `undefined` is preserved. Worth adding.

## Recommendation

Trivial fix once the undefined guard is added; otherwise this regresses the no-`--command` invocation path.
