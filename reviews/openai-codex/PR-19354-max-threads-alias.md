# PR #19354 — chore: alias max_concurrent_threads_per_session

**Repo:** openai/codex  
**Surface:** `codex-rs/config/src/key_aliases.rs`  
**Severity:** Low (compat shim, but watch the precedence)

## What changed

Adds a second entry to the static `CONFIG_KEY_ALIASES` table:

```rust
ConfigKeyAlias {
    table_path: &["agents"],
    legacy_key: "max_concurrent_threads_per_session",
    canonical_key: "max_threads",
},
```

The pattern is identical to the existing `memories.no_memories_if_mcp_or_web_search` →
`disable_on_external_context` alias. `normalize_key_aliases()` walks
the loaded TOML and rewrites legacy keys to canonical keys before the
config is parsed by Serde.

## What's risky / wrong / missing

1. **No collision policy in view.** The PR doesn't show the body of
   `normalize_key_aliases`, but the obvious failure mode is: what happens
   if a user's `config.toml` has *both* `agents.max_concurrent_threads_per_session = 4`
   *and* `agents.max_threads = 8`? Possible behaviors:
   - Legacy wins (silently overwrites new) — bad, surprising.
   - Canonical wins (silently drops legacy) — better, but still silent.
   - Hard error — best, but breaks rolling upgrades.

   The PR description should call out which one this is, and ideally add a
   test for the collision case. If the existing alias never had to deal
   with this because the legacy and canonical names lived in different
   semantic spaces, this new alias *does* reuse the same `agents.` table,
   raising the collision probability.

2. **Semantic shift, not just rename.** "max concurrent threads per
   session" is more specific than "max threads." If the canonical setting
   now governs a broader scope (e.g. global threads, or threads per agent),
   silently aliasing the per-session setting onto it could cause users to
   exceed the intended cap when the new global semantics multiply over
   multiple sessions. The aliasing assumes 1:1 semantic equivalence.

3. **No deprecation surface.** The user gets no warning that they're using
   a legacy key. Cf. typical Rust ecosystem practice of emitting a
   `tracing::warn!` from the alias rewriter so the next config edit is
   informed. Two entries today, ten in a year — without a deprecation
   warning, this table just grows.

4. **Test coverage in the PR is zero** (additions: 12, deletions: 5, all
   in the table). The existing `normalize_key_aliases` presumably has a
   test for the first entry; this should at minimum extend it to cover
   the new pair.

## Suggested fix

- Add a test asserting the `agents.max_concurrent_threads_per_session = 4`
  TOML normalizes to `agents.max_threads = 4`.
- Add a test for the collision case and document the chosen policy
  (recommend: canonical wins, warn on legacy presence).
- Add a `tracing::warn!` (or whatever the project uses) in the rewriter
  for any alias hit, so users see a one-time deprecation hint.

## Severity

Low. It's a compat shim. The risk is purely "silent semantic drift if the
new key isn't a strict superset of the old one."
