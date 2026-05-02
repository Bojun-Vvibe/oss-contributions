# charmbracelet/crush PR #2759 — fix(bedrock): accept cross-region inference profile model IDs

- PR: https://github.com/charmbracelet/crush/pull/2759
- Head SHA: `ade7292361655ba96fa3f2c9d76b639366e8067e`
- Author: @carlosgrillet (Carlos Grillet)
- Size: +9 / -1

## Summary

The Bedrock provider validator at `internal/config/load.go:300` was rejecting any model ID not starting with `anthropic.`. AWS cross-region inference profile IDs (`us.anthropic.*`, `eu.anthropic.*`, `apac.anthropic.*`, `us-gov.anthropic.*`) are the *only* way to reach Claude 4.x on Bedrock — on-demand throughput is not offered for those models — so this check made crush effectively unusable with current Anthropic models on Bedrock.

Fix strips the optional regional prefix before the family check via a small loop over the four known prefixes.

## Specific references from the diff

- `internal/config/load.go:301-309` — adds a `for _, prefix := range []string{"us.", "eu.", "apac.", "us-gov."}` loop that calls `strings.TrimPrefix(id, prefix)` on a local copy of `model.ID`, then runs the original `strings.HasPrefix(id, "anthropic.")` check against the stripped value. Error message at line 309 still reports the *original* `model.ID`, which is correct (the user's input is what they need to fix or confirm).

## Verdict: `merge-after-nits`

The fix solves a real and easily-reproduced configuration error against Bedrock + Claude 4. The validation evidence (live `crush run` round-trip with `us.anthropic.claude-opus-4-7`) is exactly what you want on this kind of fix. Two minor things keep it off `merge-as-is`.

## Nits / concerns

1. **Prefix list is hardcoded and will drift.** AWS adds new regional inference profile prefixes occasionally (`apne1.`, etc., have been mentioned in Bedrock release notes). The four-prefix list at line 305 will need maintenance. Two cleaner options:
   - Use a regex like `^[a-z0-9-]+\.anthropic\.` (matches any region-shaped prefix followed by `anthropic.`).
   - Detect by counting dot-separated segments: if `id` starts with anything ending in `.anthropic.`, accept.
   The regex form is the standard AWS-tooling pattern; SDK code in `aws-sdk-go-v2` does the same thing for ARN parsing.
2. **Loop calls `TrimPrefix` four times even after a match.** `TrimPrefix` is cheap and idempotent (no-op on miss), so functionally fine, but a `break` after a successful trim would make intent clearer:
   ```go
   for _, prefix := range []string{"us.", "eu.", "apac.", "us-gov."} {
       if rest, ok := strings.CutPrefix(id, prefix); ok {
           id = rest
           break
       }
   }
   ```
   `strings.CutPrefix` (Go 1.20+) is the right primitive here.
3. **Comment is good but slightly misleading.** "Claude 4.x on Bedrock is only available through these inference profiles" is accurate today but will age; consider "...is only available through these regional inference profiles at time of writing (2026-05); see AWS docs for current support".
4. **No test added.** `internal/config/load_test.go` (or wherever the existing provider validation is tested) should get one new case asserting `us.anthropic.claude-opus-4-7` validates and `us.openai.gpt-foo` still errors. One-liner table-test entry.
