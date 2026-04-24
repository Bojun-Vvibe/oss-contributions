# PR #24118 — fix(config): avoid parseManagedPlist crash on non-object JSON

**Repo:** anomalyco/opencode  
**Surface:** `packages/opencode/src/config/managed.ts`,
`packages/opencode/test/config/config.test.ts`  
**Severity:** Low (defensive null-guard, but right call)

## What changed

`parseManagedPlist()` previously called `Object.keys(JSON.parse(json))`
unconditionally. If the managed plist payload deserialized to `null`, an
array, a number, or a string, the call threw — propagating up through the
managed-config loader and (per the linked issue #22318) crashing startup
on machines with malformed MDM payloads.

The fix:

```ts
const raw = JSON.parse(json)
if (!raw || typeof raw !== "object" || Array.isArray(raw)) {
  return JSON.stringify({})
}
```

Tests cover `null`, `[]`, `1`, `"test"` returning `"{}"`.

## What's risky / wrong / missing

1. **Silent fallback can hide deployment misconfiguration.** Returning
   `"{}"` for `null` is fine (nothing to merge). Returning `"{}"` for an
   *array* or a *string* is masking what is almost certainly an admin
   error in the plist authoring pipeline. There is no log line, no
   warning, no metric. The user gets default config and the IT admin
   never finds out their managed payload is malformed.

   Suggest at minimum a `console.warn` (or whatever the codebase's
   structured logger surface is) noting the rejected JSON shape:

   ```ts
   console.warn("managed-config: ignoring non-object JSON payload",
                { type: typeof raw, isArray: Array.isArray(raw) })
   ```

2. **Test coverage is shape-only, not integration.** The tests assert
   `parseManagedPlist` returns `"{}"`, but they don't assert that the
   surrounding loader (`ConfigManaged.load()` or whatever the caller is)
   then merges that `{}` cleanly without re-throwing on a
   downstream `JSON.parse`. The fix is right; the test arena is too narrow
   to catch a regression that puts the throw back one layer up.

3. **`JSON.parse` itself can still throw** on malformed JSON (not non-object
   JSON — actually un-parseable input). The PR title says "avoid crash on
   non-object JSON" but doesn't address the syntax-error case. If MDM
   delivers literally garbage bytes, this still bubbles. Probably fine
   if that's a documented separate path, but worth confirming.

4. **`return JSON.stringify({})` is just `"{}"`.** Minor: the literal
   string is one allocation cheaper and easier to grep for. Not worth
   blocking on.

## Severity

Low. It's a startup-resilience fix that's directionally correct. The
risk is silently hiding the underlying admin-side bug rather than the
crash itself.

## Suggested next step

Land as-is, then file a follow-up to add a warn-level log on the reject
path and a separate handler for `SyntaxError` from `JSON.parse`.
