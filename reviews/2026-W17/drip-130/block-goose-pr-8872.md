# block/goose #8872 — chore(release): bump version to 1.33.0 (minor)

- URL: https://github.com/block/goose/pull/8872
- Head SHA: `087d65a6ef67000976e2bcd0c7cb7d75138ce39c`
- Files: 6 (Cargo.lock, Cargo.toml, canonical_models.json, plus 3 others)
- Size: +2207 / −1259

## Summary

Workspace version bump from `1.32.0 → 1.33.0` across all crates plus a
canonical-models data refresh. The version-bump portion is pure
mechanical churn (every `goose-*` crate version in Cargo.lock and the
single workspace `version` field in Cargo.toml). The bulk of the diff
(~3000 lines) is `crates/goose/src/providers/canonical/data/canonical_models.json`
churn — additions, name normalizations, and date corrections — which
is what you'd expect a minor-version release to ship.

## Specific references

- `Cargo.toml:12` flips `version = "1.32.0"` → `"1.33.0"` (single
  source of truth via workspace inheritance — this is the right shape;
  no per-crate version drift possible).
- `Cargo.lock:4332-4618` mechanically updates the seven workspace
  crates (`goose`, `goose-acp-macros`, `goose-cli`, `goose-mcp`,
  `goose-sdk`, `goose-server`, `goose-test`, `goose-test-support`) to
  `1.33.0`. No external dep version changes visible — pure version
  bump, not a dep refresh.
- `canonical_models.json` changes split into three classes:
  1. **Name corrections** at `:200`, `:318`, `:1915`, `:1944` — strip
     date suffixes from canonical model names (e.g., `claude-haiku-4-5-20251001`
     → `claude-haiku-4-5`) which is the right canonicalization for
     date-versioned upstream IDs that shouldn't leak into the human-
     facing name. The `claude-opus-4.5` case at `:318` goes the *other*
     direction (name → `claude-opus-4-5-20251101`), which is suspicious
     — either upstream changed convention or this is accidentally
     inconsistent.
  2. **Date corrections** at `:205` (knowledge `2025-03` → `2025-02`),
     `:6175-6179` (DeepSeek R1 `2025-05-28` → `2025-01-01`). The R1
     date rollback is a strong signal — the prior `0528` annotation
     was specific and informative; demoting to `2025-01-01` loses
     fidelity. Likely a data-source change but worth confirming.
  3. **Net-new model entries** at `:4501-4528` (`abliteration-ai/abliterated-model`),
     `:6453-6515` (`alibaba-cn/deepseek-v4-flash`) and presumably more
     past the truncated diff. The `abliteration-ai/abliterated-model`
     entry is concerning — "abliterated" is the term-of-art for
     models with safety-training removed, and the entry has
     `"open_weights": true` with image-input multimodal support and
     a $3/$3 cost line. Whether or not this belongs in a *canonical*
     models registry is a policy question: is the registry "every
     model goose can route to" (then yes, list it) or "every model
     goose recommends" (then no)?

## Risk

The Cargo version bump is zero-risk. The canonical_models.json changes
carry product-shape risk in three places:

1. **Name canonicalization inconsistency** — some entries strip date
   suffixes, one adds them. Pick a rule and apply uniformly, or
   document why specific models opt out.
2. **Knowledge-cutoff date downgrades** — losing `2025-05-28` →
   `2025-01-01` precision is a regression in metadata fidelity that
   downstream cost/capability comparisons may depend on.
3. **Inclusion of safety-stripped models** — `abliterated-model` is
   not a model name, it's a *category* of model produced by removing
   refusal training. Listing it under a single `id` with a single
   cost line implies a single specific model, which doesn't match the
   reality (there are dozens of "abliterated" models from different
   authors). Either drop the entry or qualify it (`abliteration-ai/<author>-<base>-abliterated`).

## Nits (non-blocking)

- Release PRs typically include a `CHANGELOG.md` entry summarizing
  what's in 1.33.0 (PRs merged since 1.32.0). Not visible in this
  diff — if it lives elsewhere (`docs/`?), a link in the PR body
  would help.
- The 2207/1259 line count is dominated by `canonical_models.json`
  churn. Splitting "version bump (Cargo.toml + Cargo.lock)" from
  "models data refresh" into two PRs would let release-cutting
  happen on a clean diff without entangling model-data review with
  release timing. Especially relevant given the
  `abliterated-model` policy question above — a release shouldn't
  block on resolving it.
- No verification that downstream consumers of `canonical_models.json`
  (the canonical-model resolver, cost calculator, capability checks)
  handle the renamed IDs gracefully. A test loading the JSON and
  asserting all `id` keys are unique and all `name` values are
  non-empty would be cheap.

## Verdict

`needs-discussion` — the version bump itself is fine but the
co-shipped canonical models data has three concrete questions
(date-suffix policy, knowledge-date precision regression, abliterated
model inclusion) that should be answered before this lands as a
versioned release. Easiest path: split into two PRs.
