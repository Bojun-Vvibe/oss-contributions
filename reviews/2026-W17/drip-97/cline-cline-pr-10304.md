# cline/cline#10304: feat: Vertex AI Architecture Overhaul (Region Decoupling and 1M context routing)

- **Author:** TAMdrew
- **HEAD SHA:** 2465853f
- **Verdict:** request-changes

## Summary

A 5212/-4984 PR billed as a Vertex AI architectural overhaul that
claims four wins: (1) decouples `vertexRegion` so plan-mode and
act-mode can target different regions independently
(`planVertexRegion` / `actVertexRegion` added to proto field
indices 242 and 243); (2) adds Opus-4.7-style `task_budget` (soft
cap) schema with new `adaptive` thinking type; (3) strips
`temperature` and `top_p` when reasoning is enabled to avoid
strict-mode 400s; and (4) adds Gemma 4 MaaS, Haiku 4.5, and
Gemini 3.1 Flash-Lite to the model registry.

The first claim (region decoupling) is the actual substantive
change and is implemented cleanly in `src/core/api/index.ts:130-138`
and `:172-180` with a sensible mode-aware fallback chain
(`planVertexRegion || vertexRegion` and `actVertexRegion ||
vertexRegion`). However, the diff shape makes this very hard to
land as-is: `src/shared/api.ts` shows +4862/-4727 against a file
where the underlying intent is "add three model entries", which
strongly suggests the entire file has been reformatted (e.g., a
prettier/biome config drift, line-ending change, or quote-style
toggle), drowning the actual model additions in noise.
`src/core/api/providers/vertex.ts` (+329/-254) shows the same
pattern — every line in the diff hunks visible has gained a
trailing semicolon and double-quote style switches, indicating
a formatter mismatch with the repo standard.

## Specific feedback

- **Blocker — formatter drift:** `src/shared/api.ts:1-end` shows
  near-total reformat (+4862/-4727 on a file where the substantive
  change is adding ~3 model entries). The visible
  `src/core/api/providers/vertex.ts` hunk shows lines like
  `import { buildExternalBasicHeaders } from "@/services/EnvUtils";`
  *with* a trailing semicolon while the original was unsemicoloned;
  same for double-quote vs the repo's apparent existing style.
  Run the repo's formatter (`npm run format`) against `main` and
  rebase so the diff isolates the actual logic changes. Without
  this, the diff is unreviewable.
- **Blocker — placeholder in PR template:** `Issue: #XXXX`
  literal placeholder in the PR body's "Related Issue" field
  (cline contribution guidelines explicitly require a linked
  issue/discussion for non-trivial changes). The body references
  "#10295 (Follow-up to partial Vertex AI implementation)" but
  the template field itself is unfilled.
- `proto/cline/models.proto:691-693` — `plan_vertex_region` and
  `act_vertex_region` added at field indices 242, 243. Indices
  match the existing pattern (the prior added field at 241 is
  `act_mode_cline_model_info`). Forward-compat OK; reverse-compat
  unaffected because these are `optional`.
- `src/core/api/index.ts:130-138` (Vertex handler creation) and
  `:172-180` (Gemini handler creation) — region-mode plumbing is
  symmetric across both providers, which is correct since both
  hit Vertex region endpoints. Fallback to legacy `vertexRegion`
  when the mode-specific value is unset is the right
  backward-compat posture.
- `src/core/api/providers/gemini.ts:36-45` —
  `mapReasoningEffortToGeminiThinkingLevel` change splits `medium`
  out of the `low`-merged branch and returns `"MEDIUM" as
  ThinkingLevel`. The `as ThinkingLevel` cast is suspicious — if
  `MEDIUM` is a real enum value upstream the import should be
  used; if it isn't, this cast lies to the type system and the
  runtime value may not be honored by `@google/genai`. Verify
  against the SDK's actual `ThinkingLevel` enum.
- `src/shared/storage/state-keys.ts +4/-0` — new state keys for
  the decoupled regions; confirm the corresponding read/write
  sites in the configuration store are also updated (the file
  list does not show a settings-store touch, which is suspicious
  for a state-keys addition).
- Test plan in the PR body is entirely manual ("Verified via
  terminal logs that separate regional endpoints were called
  correctly without state leakage") — a 5k-line provider overhaul
  needs at least one regression test asserting the region pickup
  in plan vs. act mode. No test files appear in the file list.
- The PR description claims support for Opus 4.7 and a
  `task-budgets-2026-03-13` header but `vertex.ts` is the only
  provider file changed (no Anthropic-direct provider equivalent),
  which means cross-provider parity for the same reasoning model
  family is not addressed in this PR.

## Risks / questions

- The combination of (a) a near-complete file reformat in
  `src/shared/api.ts`, (b) provider-file reformat in `vertex.ts`,
  and (c) zero added tests for a 5k-line behavioral PR makes
  this hard to merge confidently. Splitting into:
  (1) formatter-conformance commit landed first against `main`,
  (2) region-decoupling proto+plumbing PR with tests,
  (3) model-registry-additions PR for the three new entries,
  (4) reasoning-payload-cleanup PR with tests
  would each be reviewable on their own merits.
- `temperature`/`top_p` strip-when-reasoning logic is a strong
  semantic change to outbound payloads. It needs a unit test
  pinning down the exact set of {model × reasoning-flag} cases
  that strip vs. preserve, otherwise a future model addition
  silently regresses behavior.
- Vertex region availability is model-dependent (e.g., Claude
  models are not available in every region) — the decoupled-region
  feature should have a validation step that warns when the user
  selects a {model, region} combination that Vertex doesn't
  actually serve, otherwise the user gets a 404 from the upstream
  with no actionable hint.
