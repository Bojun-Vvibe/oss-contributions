# block/goose PR #8816 — chore: introduce DEFAULT_PROVIDER_TIMEOUT_SECS constant for providers

- **PR:** https://github.com/block/goose/pull/8816
- **Author:** r0x0d
- **Head SHA:** `33228947d614733e51c78bd379702dd9580fdd5a`
- **Files:** 11 (+57 / -19)
- **Verdict:** `merge-as-is`

## What it does

Pure consolidation refactor: ten provider files (`api_client`, `databricks`,
`gcpvertexai`, `gemini_oauth`, `gh-cp` (a vendor provider), `kimicode`, `litellm`,
`ollama`, `openai`, `toolshim`) each had their own hardcoded
`Duration::from_secs(600)` for the HTTP timeout. This PR defines
`pub const DEFAULT_PROVIDER_TIMEOUT_SECS: u64 = 600;` in
`providers/base.rs` with a doc comment, and routes every provider through
the constant.

## Specific reads

- `crates/goose/src/providers/base.rs:7-10` — the new constant with a
  three-line doc comment explaining the rationale ("Long-running model
  inference can take several minutes, so we allow up to 10 minutes
  before giving up. Individual providers may override this via their
  own config key."). The "individual providers may override" is the key
  caveat — Databricks, for instance, keeps its own
  `const DEFAULT_TIMEOUT_SECS: u64 = DEFAULT_PROVIDER_TIMEOUT_SECS;`
  alias, preserving the option to diverge later.
- `crates/goose/src/providers/api_client.rs:281-285` — `ApiClient::new`
  changed from `Duration::from_secs(600)` to
  `Duration::from_secs(DEFAULT_PROVIDER_TIMEOUT_SECS)`. The `with_timeout`
  signature unchanged. So all callers of `new` get the constant, all
  callers of `with_timeout` keep their explicit override. Right shape.
- `crates/goose/src/providers/databricks.rs:47` — pattern of
  `const DEFAULT_TIMEOUT_SECS: u64 = DEFAULT_PROVIDER_TIMEOUT_SECS;`
  preserves the per-provider name while wiring it to the central value.
  This is the "consolidation without lock-in" pattern: if Databricks
  later needs 1200s, you change one line in `databricks.rs` and the
  central constant is unaffected. Better than `Duration::from_secs(600)`
  (silent drift) *and* better than dropping the per-provider alias
  (forces every provider to use the same value).
- The diff stat (+57 / -19) is a +38 net-line change across 11 files —
  about 3-4 lines added per provider for `use super::base::{...,
  DEFAULT_PROVIDER_TIMEOUT_SECS}` plus the constant alias plus the
  call-site update. Reasonable bloat for a centralization win.
- No behavior change. Every timeout is still 600 seconds.

## Risk surface

**Effectively zero.** Replacement of a magic number with a named
constant whose value matches the original. The only ways this can
break:

1. Import-cycle introduced by `api_client.rs` now depending on
   `base.rs` — but `api_client` already used `base.rs` types, so this
   is a no-op on the dep graph.
2. A provider file *not* in the list still has `600` hardcoded — i.e.,
   the consolidation is incomplete. A `git grep '600'` across
   `crates/goose/src/providers/` should return only the constant
   definition itself plus any genuinely-different timeouts (e.g. a
   shorter timeout for OAuth token exchange). Worth a one-line check
   in CI: `! git grep -q 'Duration::from_secs(600)' crates/goose/src/providers/`.

## Suggestions before merge

- Add the `git grep` check to CI to prevent reintroduction of the
  literal.
- Consider whether the constant should be `pub(crate)` instead of
  `pub` — it's not obviously useful outside the providers module, and
  `pub(crate)` keeps the public surface smaller. Minor.
- The doc comment says "Individual providers may override this via
  their own config key" — but the provider-side aliases (e.g.
  Databricks's `DEFAULT_TIMEOUT_SECS`) are *compile-time* aliases, not
  config-key overrides. Either the comment is forward-looking ("we
  intend to add config-key support") or it's slightly inaccurate. Worth
  clarifying.

Verdict: merge-as-is — exemplary chore PR. Single concern, no behavior
change, eleven files touched in mechanically identical ways, with
preserved per-provider aliases that allow future divergence without
churn. The kind of refactor that prevents the next "we changed the
timeout to 1200 in 8 of 10 providers and forgot the other 2" bug.

## What I learned

The "named-constant + per-module alias" pattern is undervalued. The
naive consolidation is "every provider directly uses
`base::DEFAULT_PROVIDER_TIMEOUT_SECS`" — which works but **forces
every divergence to be a global discussion**. The pattern this PR
uses — central constant + per-provider alias re-export — gives each
provider its own knob to turn while keeping the default consistent.
The cost is one line of `const X: u64 = Y;` per file. The benefit is
that the next provider that needs a non-default timeout can change
one line in its own file, with code review limited to that
provider's owners. That's a real win on a multi-team codebase.
