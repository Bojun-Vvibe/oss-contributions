---
pr: 2568
repo: charmbracelet/crush
sha: e44f712255b45276e46d108879860258a96d61bf
verdict: merge-after-nits
date: 2026-04-26
---

# charmbracelet/crush #2568 — feat: support Bedrock cross-region inference model IDs and annotate 1M models

- **URL**: https://github.com/charmbracelet/crush/pull/2568
- **Author**: ahmadaccino
- **Head SHA**: e44f712255b45276e46d108879860258a96d61bf
- **Size**: +84/-1 across `internal/config/load.go` + `internal/config/load_test.go`

## Scope

Two related changes in `internal/config/load.go`:

1. **`isBedrockAnthropicModel(id string) bool`** (new, lines 119-122) — replaces the existing `strings.HasPrefix(model.ID, "anthropic.")` guard at the configure-providers callsite (line 299) with a helper that also accepts cross-region inference IDs like `us.anthropic.claude-opus-4-6-v1:1m`, `eu.anthropic.claude-sonnet-4-6:1m`, `ap.anthropic.claude-opus-4-6-v1:1m`. Implementation: `strings.HasPrefix(id, "anthropic.") || strings.Contains(id, ".anthropic.")`.

2. **`annotateBedrockModelNames(models []catwalk.Model) []catwalk.Model`** (new, lines 127-134) — appends `" (1M)"` to `model.Name` for any Bedrock model with `ContextWindow >= 1_000_000`, idempotent on re-call (`!strings.Contains(models[i].Name, "1M")`). Wired in at line 303 right after the validation loop.

Three new tests cover the matrix:
- `TestConfig_configureProvidersBedrockCrossRegionModel` — full integration through `configureProviders`, asserts a `us.anthropic.claude-opus-4-6-v1:1m` model is preserved.
- `TestAnnotateBedrockModelNames` — unit test on the annotator, including the 200K context model that should *not* get the `(1M)` tag.
- `TestIsBedrockAnthropicModel` — explicit truth table over `anthropic.*`, `us.anthropic.*`, `eu.anthropic.*`, `ap.anthropic.*`, `some-random-model`, `openai.gpt-4`.

## Specific findings

- **`strings.Contains(id, ".anthropic.")` is the right check, but slightly over-permissive.** It accepts any string of the form `<anything>.anthropic.<anything>` — so a hypothetical `mycompany.anthropic.foo` would pass. AWS's actual cross-region prefixes are a closed set: `us`, `eu`, `apac`, `ap`, etc. (publicly listed in the Bedrock cross-region inference docs). **Nit:** consider tightening to a regex like `^(us|eu|apac|ap)\.anthropic\.` if maintainer prefers strict, or leave as-is if maintainer prefers permissive (AWS adds new regions). Not a blocker — the loose check is forward-compatible with future AWS region prefixes the team won't have to chase.

- **The `(1M)` annotation is heuristic-based, which is correct.** `model.ContextWindow >= 1_000_000` captures the 1M-context variants without hard-coding model IDs. The idempotency guard `!strings.Contains(models[i].Name, "1M")` means re-calling the function won't append `(1M) (1M)` — useful if `configureProviders` is ever called twice, or if the catwalk catalog already labels some entries. Worth checking: if the user has *customized* `model.Name` in their crush config to include "1M context window" or similar, the substring guard might still match and skip annotation. Edge case, unlikely to bite.

- **Tests are thorough.** `TestIsBedrockAnthropicModel` (load_test.go:317-324) covers 4 region prefixes and 2 negative cases. `TestAnnotateBedrockModelNames` (load_test.go:296-315) covers two ≥1M models *and* a 200K model that should not be annotated — that negative assertion is the right check (catches a future bug where someone accidentally inverts the inequality).

- **The integration test at load_test.go:257-280** uses `us.anthropic.claude-opus-4-6-v1:1m` — confirming the previously-failing path now succeeds. Asserts the model survived configure with its original ID intact (no rename). 

- **`prepared.Models = annotateBedrockModelNames(p.Models)` at line 303** assigns to `prepared.Models` after the validation loop iterates over `p.Models`. The validation loop reads `model.ID` only (not `model.Name`), so the order is fine: validate IDs against the original list, then annotate names into the prepared output. If annotation were done *before* validation, an attacker-controlled `Name` couldn't break things either (it's not used in the guard), but the current ordering reads cleaner.

- **Backwards-compat check.** Existing users with `anthropic.claude-...` IDs still pass `isBedrockAnthropicModel` via `HasPrefix`. No config-file migration needed. Users with `us.anthropic.claude-...` IDs (who today would hit the old `"bedrock provider only supports anthropic models"` error) now succeed — that's the intended behavior change.

- **Author appears to be a drive-by but a careful one.** `ahmadaccino` doesn't appear in recent commits. PR follows the test-first style (3 new tests for 2 helpers + 1 callsite change), is exactly scope-bounded, and the helper extraction makes the diff easier to review than an inline check would have been.

## Risk

Low. The looser ID check is forward-compatible with AWS regions but does accept `<anything>.anthropic.<anything>` — only relevant if someone hand-rolls a non-AWS-format model ID with `.anthropic.` in it, which is implausible for the catwalk catalog source. The 1M annotation is purely cosmetic in the model-selection dropdown and only fires on `>= 1_000_000` context. Worst case: a user with a custom theme that strips parentheses sees an awkward " 1M" trailing space; trivial follow-up fix.

## Verdict

**merge-after-nits** — two optional nits:
1. Tighten `isBedrockAnthropicModel` to an allowlist regex (`^(us|eu|apac|ap)\.anthropic\.`) if maintainer prefers strict matching. Not required.
2. Consider whether the `(1M)` suffix should also surface in the `model.ID` comparison (e.g. `:1m` suffix detection) as a secondary signal — currently context-window is the only check. If catwalk ever emits a 1M model with `ContextWindow == 0` (unset), this annotation is silently skipped. Not blocking.

If neither nit lands, this is still a clean, well-tested ship.

## What I learned

Cross-region inference IDs are the kind of vendor-side schema change that breaks everyone's "just check the prefix" validation simultaneously. The pattern `HasPrefix(id, fixedPrefix) || Contains(id, "."+fixedPrefix+".")` is a reasonable forward-compat hedge for any regional-prefix dotted-namespace ID format (AWS Bedrock, GCP regions, Azure deployment IDs all have this shape). Worth adopting as a small idiom for any code that gates on namespace-rooted IDs.
