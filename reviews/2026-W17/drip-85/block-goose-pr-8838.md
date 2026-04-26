# block/goose PR #8838 — Refresh canonical model metadata from models.dev

- **Repo:** block/goose
- **PR:** [#8838](https://github.com/block/goose/pull/8838)
- **Head SHA:** `290cb03d6727a4805f5b722f11fe74fe07d9be3d`
- **Author:** angiejones (Angie Jones)
- **Files:** 3 files, +2080/−87
  - `crates/goose/src/providers/canonical/data/canonical_models.json` (bulk model metadata refresh)
  - `crates/goose/src/providers/canonical/data/provider_metadata.json`
  - `ui/desktop/tests/integration/test_providers_lib.ts` (single model id rename)

## Context

The goose UI was failing to render pricing for several recently-added
models (Kimi K2.6, DeepSeek V4 Flash/Pro, Claude Sonnet 4.6, etc.)
because the bundled `canonical_models.json` snapshot was stale. The PR
syncs the canonical metadata from upstream `models.dev` to pick up new
entries and fix pricing display. The body screenshots show the
"before" missing-pricing state and the "after" fixed-pricing state.

## What the diff actually does

**`canonical_models.json`** — adds many new model entries across at
least 20+ providers and renames a few existing ones. Sample of new
entries (each follows the same shape: `id`, `name`, `family`,
capability flags, `modalities`, `cost.{input,output}`,
`limit.{context,output}`, dates):

- `alibaba-cn/kimi-k2.6` — Moonshot K2.6 with `release_date: "2026-04-21"`,
  262144 context, `cost.input: 0.929`, `cost.output: 3.858`,
  multimodal text/image/video input.
- `amazon-bedrock/au.anthropic.claude-opus-4.6-v1` and
  `amazon-bedrock/au.anthropic.claude-sonnet-4.6` — AU regional
  Bedrock variants.
- `azure/claude-sonnet-4.6` — Azure Claude entry.
- `baseten/moonshotai/Kimi-K2.6`, `chutes/moonshotai/Kimi-K2.6-TEE`,
  `cortecs/kimi-k2.6`, `deepinfra/moonshotai/Kimi-K2.6`,
  `fireworks-ai/accounts/fireworks/models/kimi-k2p6`,
  `firmware/kimi-k2-6`, `kilo/moonshotai/kimi-k2.6` — same Kimi K2.6
  surfaced through 7+ different provider gateways with different
  id-encoding conventions.
- `deepseek/deepseek-v4-flash`, `deepseek/deepseek-v4-pro` plus
  `venice/deepseek-v4-flash`, `venice/deepseek-v4-pro`, etc.
- `deepinfra/Qwen/Qwen3.5-35B-A3B`, `deepinfra/Qwen/Qwen3.5-397B-A17B`,
  `deepinfra/Qwen/Qwen3.6-35B-A3B`,
  `siliconflow-cn/Qwen/Qwen3.6-35B-A3B`,
  `regolo-ai/qwen3.5-9b`, `regolo-ai/qwen3.5-122b` — Qwen 3.5 / 3.6
  family.

**Renames** (one row, semantic):
- `google/gemma-4-26b-it` → `google/gemma-4-26b-a4b-it` (the `-a4b-`
  active-parameter-count suffix added to disambiguate against the
  Gemma 4 dense variant).

**`provider_metadata.json`** — provider-list sync (not detailed in
the head of diff, presumably new provider entries to match the new
model entries).

**`ui/desktop/tests/integration/test_providers_lib.ts:61`** — single
test fixture rename:
```diff
-        { name: 'nvidia/nemotron-3-nano-30b-a3b', flaky: true },
+        { name: 'nvidia/nemotron-3-nano-30b-a3b:free', flaky: true },
```
NVIDIA Nemotron 3 Nano 30B-A3B is the same model with a `:free` tier
suffix added (probably because the gated variant is what's now
canonical).

## Strengths

- **Right shape of fix.** Pricing wasn't surfacing in the UI because
  the bundled snapshot was stale; refreshing from upstream
  `models.dev` is the canonical mechanism for this kind of metadata
  drift, and the screenshots in the PR body confirm the user-visible
  fix.
- **Each new entry follows the existing schema** — same field set,
  same naming conventions per provider (e.g.,
  `amazon-bedrock/<region>.<vendor>.<model>-v1`,
  `firmware/<model>-<version>`), no schema breakage.
- **Multi-modal flags filled in correctly** for K2.6
  (`modalities.input: ["text","image","video"]`) — these come from
  `models.dev` upstream and the PR doesn't second-guess them, which
  is the safe stance for a refresh.
- **The integration test fixture rename** at
  `test_providers_lib.ts:61` is mechanical and proportional —
  `nvidia/nemotron-3-nano-30b-a3b` → `:free` aligns with whatever
  upstream rename happened.

## Risks / nits

1. **No diff between "what models.dev shipped" and "what we copied"**
   — refresh PRs from upstream catalogs are notoriously easy to
   accept silently because each entry "looks like" the existing
   entries. The PR body should either link the upstream
   `models.dev` commit hash or attach a tool-generated diff
   showing exactly which entries are new vs renamed vs (silently)
   removed. Otherwise, a maintainer five months from now can't tell
   if a removed entry was intentional or lost in the sync.
2. **0 deletions despite 87 line removals** — the line-count delta
   is +2080/−87, which means 87 lines of existing entries got
   replaced or removed. Need to verify that no model id used by
   real goose users disappeared from the catalog. A `git diff
   --no-color HEAD~1 -- crates/goose/src/providers/canonical/data/canonical_models.json
   | grep '^-\\s\\+"id":'` should be inspected line-by-line to
   confirm every removed id was either renamed (with the new id
   visible in the diff) or known-deprecated.
3. **Gemma rename `-26b-it` → `-26b-a4b-it`** is a breaking change
   for any user who has the old id pinned in a config or session.
   Either keep both ids (alias the old one) or document the rename
   in a CHANGELOG entry so users hit a clear error rather than
   "model not found" silently.
4. **`fireworks-ai/accounts/fireworks/models/kimi-k2p6`** — the
   `k2p6` suffix is an upstream-Fireworks convention but reads as
   a typo to anyone scanning the catalog. A comment in the JSON
   isn't really an option (JSON has no comments) so this just has
   to be accepted, but verify it's the actual Fireworks id and
   not a transcription error from `k2.6`.
5. **No model-data validation in CI?** A schema-check that asserts
   every entry has the required fields (`id`, `name`, `cost.input`,
   `cost.output`, `limit.context`) and that no two entries share
   the same `id` would prevent sync-introduced regressions. If
   such a check exists, the test_providers_lib rename is the only
   visible CI signal in the diff — that's thin coverage for a
   2080-line metadata refresh.
6. **Multiple Kimi K2.6 entries with different cost values** — at
   least 8 provider gateways (alibaba-cn, baseten, chutes, cortecs,
   deepinfra, fireworks, firmware, kilo, siliconflow, togetherai,
   venice, zenmux, etc.) all carry their own Kimi K2.6 row with
   distinct pricing. That's correct behavior (different gateways
   charge differently) but if the upstream `models.dev` ever
   normalizes these to a single canonical row with a `gateways[]`
   array, the next refresh will be a destructive rewrite. Worth a
   doc-comment somewhere in the catalog README explaining the
   "one row per gateway" convention.
7. **Test fixture `:free` suffix** at `test_providers_lib.ts:61`
   may break the test if `OPENROUTER_API_KEY` actually has access
   to the non-`:free` tier and not the free tier (or vice versa).
   Worth a CI-only smoke run before tagging.

## Verdict

`merge-after-nits`

This is a routine catalog refresh that fixes a real user-visible
issue (missing pricing in the UI). The diff is mechanical and the
schema shape is preserved end-to-end. Two follow-up items before
merge: (a) attach a tool-generated summary of "X new ids, Y renamed
(old→new), Z removed" so the 87 line-deletions are auditable, and
(b) add a one-line CHANGELOG entry for the
`google/gemma-4-26b-it` → `google/gemma-4-26b-a4b-it` rename so
config-pinned users get a clear migration breadcrumb. The
`:free`-suffix test rename should be smoked once on CI with the
real `OPENROUTER_API_KEY`. The Kimi-K2.6-across-12-gateways
duplication is fine as-is given the existing per-gateway-row
convention.
