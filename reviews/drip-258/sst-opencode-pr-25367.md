# sst/opencode PR #25367 — fix(session): cache messages across prompt loop to preserve prompt cache byte-identity

- PR: https://github.com/sst/opencode/pull/25367
- Head SHA: `6f1e554fc2e55b1fa50e9169f1a1261ed1a51872`
- Author: @BYK (Burak Yigit Kaya)
- Closes: #25366

## Summary

OpenCode's prompt loop calls `MessageV2.filterCompactedEffect(sessionID)` on every iteration, which re-reads messages from the DB. Between tool-call steps, tool parts transition `pending → completed` with new output text, so the same assistant message serializes to **different bytes** on consecutive API calls — busting Anthropic's prompt cache from that point forward. This PR caches the conversation array (`msgs`) across loop iterations and only appends *genuinely new* messages on tool-call continuations. Full reloads still happen on the first iteration, after compaction, and after subtask/overflow recovery (i.e. cases where the conversation structure actually changes). Reported impact: $2,264 in cache writes vs $1,234 in cache reads on a single account on Apr 21.

## Specific references from the diff

- `packages/opencode/src/session/prompt.ts:1280-1299` — introduces `let msgs: MessageV2.WithParts[] | undefined` and `let needsFullReload = true`; on `needsFullReload || !msgs` it does the old full reload, otherwise it loads fresh, computes `knownIDs`, and `msgs.push(m)` for any IDs not already cached.
- `:1348` (subtask), `:1361` (overflow recovery `result === "continue"`), `:1371-1373` (auto-compaction on overflow), `:1505-1507` (auto-compaction inside tool-call handler) — each of these structurally-changing branches sets `needsFullReload = true` before `continue`.
- `packages/app/vite.js:13-19` — unrelated drive-by adds `"@opencode-ai/core"` alias and re-indents `resolve:` (see nit 2).

## Verdict: `merge-after-nits`

The diagnosis is exactly right and the fix is minimally invasive. Only thing keeping this off `merge-as-is` is the unrelated `vite.js` change and the mutation-by-`push` style that's slightly fragile in this otherwise-pure-Effect file.

## Nits / concerns

1. **Cache correctness depends on `filterCompactedEffect` being a superset.** The `knownIDs.has(m.info.id)` merge assumes that any message in the cached `msgs` is also in the freshly-loaded `fresh` array — i.e. messages are never removed mid-loop except by compaction (which forces a full reload). Worth an inline comment at `:1289-1294` stating that invariant explicitly, since the next person to touch this will not see why we're confident a non-reload path is safe.
2. **`packages/app/vite.js` change is unrelated.** Lines 13-19 add a `"@opencode-ai/core"` alias and re-indent `resolve:` inside `config()`. This has nothing to do with prompt-cache byte-identity. Either split it into its own PR or call it out in the description — right now it'll silently land under a `fix(session)` commit.
3. **Mutating push vs immutable rebuild.** `msgs.push(m)` mutates the array the caller still holds. In this Effect-heavy file the rest of the data is treated immutably; consider `msgs = [...msgs, ...fresh.filter(...)]` for consistency. Functionally identical, but matches the surrounding style and is easier to reason about under structured concurrency.
