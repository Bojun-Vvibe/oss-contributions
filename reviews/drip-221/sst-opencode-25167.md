---
pr-url: https://github.com/sst/opencode/pull/25167
sha: 560baae15d80
verdict: merge-as-is
---

# fix: ensure user config takes precedence over plugin hooks for model resolution

Surgical reordering inside the `provider/provider.ts` layer-build generator: the 27-line plugin-`models()`-hook block at `provider.ts:1140-1166` is moved from *after* the `configProviders` extension loop to *before* it. The diff is a pure cut-and-paste — same 27 lines deleted at the old site (`:1326-1352` in the pre-image) and re-inserted at the new site, with the only line-level change being the `provider` lookup base going from `providers[providerID]` (the post-config-extension snapshot) to `database[providerID]` (the pre-config-extension base). That single identifier change is the load-bearing fix: at the new earlier insertion point the merged `providers` map doesn't yet exist, so the hook has to read from `database` (the raw provider database before user config has been folded in). The downstream `for (const [providerID, provider] of configProviders)` loop at `:1166+` then sees the plugin-supplied `models` map already on `provider.models` and the standard "user config wins" merge in that loop overrides whatever the plugin-hook produced — restoring the documented precedence (user config > plugin hook > built-in) that had silently inverted to (plugin hook > user config) when the hook ran last.

The shape of the bug is the textbook ordering hazard for "last write wins" extension chains: the plugin hook was *correct* in isolation (it produced the right model list for the plugin-aware case) but the chain's *order* made it unstoppable — a user who'd explicitly set `model: { my-provider: { ... } }` in `opencode.json` to override a plugin-injected entry would see the plugin value re-stomp it on every layer rebuild. The fix is the right kind: don't add a precedence-flag parameter to the hook, don't add a "user-pinned" sentinel field on `Provider`, just run the steps in the order whose last-write-wins semantics produce the documented precedence.

The one nit small enough not to block: there's no test added, and the new ordering is the kind of thing a future "let me clean up this layer-build generator" refactor will silently re-invert. A 30-line `it("user config beats plugin models hook")` test pinning `database` → plugin-`models()` → `configProviders` extension → `model[user-key]` survives would make the contract explicit. But as a one-character semantic fix paired with a clean cut-and-paste, this lands cleanly.

## what I learned
"Plugin hook ran in the wrong order so user config got silently overridden" is one of the most-reported and least-debugged failure modes in extensible config systems — the user sees their config *appears* to take effect on first render (because the layer hasn't rebuilt yet), then mysteriously reverts on the next interaction. The fix is almost always a re-ordering rather than a new abstraction: the precedence rules were already documented, just not enforced by execution order.
