# google-gemini/gemini-cli#26282 — fix(cli): skip redundant settings writes and preserve trailing newlines

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26282
- **Head SHA**: `02c25b7b284422432404dff3d2a63b8ec5bed131`
- **State**: CLOSED
- **Size**: +123 / -2, 2 files
- **Verdict**: **merge-after-nits** (despite CLOSED state — design-of-record review)

## Context

Fixes #18934. The `updateSettingsFilePreservingFormat(filePath, updates)` helper rewrites `.gemini/settings.json` whenever a settings update fires (including internal migrations that don't actually change anything). Two problems:

1. **No-op writes still happen.** A migration that updates a field to its existing value still rewrites the file, churning mtime, breaking file-watcher idempotence, and (more visibly) reformatting any `comment-json`-preserved formatting like single-line arrays.
2. **Trailing newline drift.** `JSON.stringify(parsed, null, 2)` does not emit a trailing newline. If the user's file had one (`.gemini/settings.json\n`), the rewrite drops it. POSIX-text-file linters and git complain.

## What's right

**`packages/cli/src/utils/commentJson.ts:24-30, 41-51` — short-circuit on logical equality.**

Two parallel parses now happen on read:
- `parsed = parse(originalContent)` — the existing `comment-json` parse that preserves comments via Symbol keys.
- `cleanParsed = JSON.parse(stripJsonComments(originalContent))` — a clean parse that drops comments and Symbol metadata.

Then at `:50-53`:

```ts
const updatedClean = applyUpdates(structuredClone(cleanParsed), updates);
if (deepEqual(cleanParsed, updatedClean)) {
  return;
}
```

The clean-parse is the **right surface for equality testing** because `comment-json`'s parsed shape carries Symbol-keyed metadata (comment text, position info) that would always make two re-parsed representations unequal even when logical content is identical. Doing the equality check on the clean parse avoids false-positive writes and avoids comparing comment-juxtaposition state.

`structuredClone` at `:50` is the right primitive for the comparison-only `applyUpdates` call — `applyUpdates` mutates its first argument, so we need a deep copy of `cleanParsed` to avoid corrupting it before the equality check.

**`commentJson.ts:38, 56-58` — trailing-newline preservation.**

```ts
const hasTrailingNewline = originalContent.endsWith('\n');
// ...
const finalContent = hasTrailingNewline
  ? updatedContent + '\n'
  : updatedContent;
```

Three behaviors pinned by tests:
- New file (`!fs.existsSync(filePath)`) at `:24-27`: now writes `JSON.stringify(updates, null, 2) + '\n'` (was no trailing `\n`). Sensible default.
- Existing file with `\n`: preserves it.
- Existing file without `\n`: preserves the *absence*. Test at `:425-431` explicitly pins this: "should NOT add trailing newline if original file did not have one."

The third behavior is the right respectful-of-user-format choice — if the user has a no-trailing-newline workflow (rare but real, e.g., concatenating with another file via `cat`), we don't fight it.

**`deepEqual` helper at `:73-104`** correctly:
- Short-circuits on reference equality.
- Handles arrays separately from objects (uses `Array.isArray` to distinguish, so `{0: 'a', 1: 'b', length: 2}` doesn't match `['a', 'b']`).
- Uses `Object.getOwnPropertyNames` to enumerate keys — this is the correct method to skip Symbol-keyed `comment-json` metadata while including all string-keyed enumerable+non-enumerable own properties. **The choice of `getOwnPropertyNames` over `Object.keys` is load-bearing**: `Object.keys` would skip non-enumerable properties, and while neither input should have non-enumerable string keys after a `JSON.parse(stripJsonComments(...))`, `getOwnPropertyNames` is the conservative-correct choice.
- Length check on arrays before recursing keeps O(n) on the unequal-length case.

**Test coverage at `commentJson.test.ts:369-435` — four tests, all targeted:**
1. `should skip write if logical content has not changed` — captures `mtimeMs`, waits 10ms, calls update with identical content, asserts mtime unchanged AND content byte-identical to original. Strong regression pin.
2. `should preserve trailing newline on legitimate update` — pre-existing `\n` survives a real model rename.
3. `should add trailing newline to new files` — confirms the new-file default.
4. `should NOT add trailing newline if original file did not have one` — pins the no-newline-preservation case.

## Risks / nits

- **`stripJsonComments` is a new dependency** added to the import chain. PR body doesn't mention adding it to `package.json`. If `strip-json-comments` is already a transitive dep via `comment-json`, the import works; if not, the build breaks. Worth confirming via `npm ls strip-json-comments` and adding to `package.json` if needed.
- **Two parses on every call.** For a small settings file this is microsecond-scale, but if `commentJson.ts` is called in a hot path (e.g., during settings.json re-render at every keystroke), the doubled parse cost could matter. Not a concern for the current "called on update" surface; flag if the call site widens.
- **`applyUpdates` is called twice** — once on the clone for comparison, once on the original `parsed` (with comments) for the write at `:55`. If `applyUpdates` has any side effect beyond mutation of its first argument (e.g., logging, event emission), it fires twice. Worth confirming `applyUpdates` is pure (it appears to be from the surrounding code, but the helper isn't fully visible in this diff).
- **`structuredClone` is Node 17+.** `gemini-cli` already requires Node 18+ (per its `package.json` `engines.node` per project convention) so this is fine, but a one-line comment justifying "structuredClone over JSON.parse(JSON.stringify(...))" would help — the difference matters for objects containing Maps/Sets/Dates.
- **Error path at `commentJson.ts:42-50`** silently emits feedback and returns when `parse()` or `stripJsonComments`+`JSON.parse()` fails. Both parses are inside the same `try`; if `parse(originalContent)` succeeds but `JSON.parse(stripJsonComments(originalContent))` fails (e.g., comments containing `*/` that confuse `strip-json-comments`), the error message is generic and may not point at the right cause. Worth splitting the two parses into separate try blocks with distinct error messages.
- **`deepEqual` returns `false` for `NaN === NaN`** because of the `a === b` check at `:75`. JSON doesn't allow `NaN` as a value, so this is fine in the settings-file context, but worth noting in a comment that this is an "JSON-equality" helper, not a "structural-equality" helper.
- **PR is CLOSED** without a stated reason in the visible body. If it was closed in favor of a different approach, this review serves as design-of-record for the rejected approach. If it was closed because the test infrastructure was found insufficient, that's worth understanding before resurrecting the same fix.

## Verdict

**merge-after-nits** (design-of-record). The double-parse (with-comments + clean) for equality-checking is exactly the right pattern for `comment-json`-managed files, and the trailing-newline preservation respects the three relevant cases (preserve-yes, preserve-no, default-yes-on-new). Test coverage is targeted and pins the regression. Address the `strip-json-comments` dependency check, split the parse error paths, and document the `getOwnPropertyNames` / `structuredClone` choices before resurrecting under a new PR.
