# google-gemini/gemini-cli #26158 — `feat(core): implement tool repair`

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26158
- **Head SHA:** `f4bc19915161c610af613be528694806b59b622a`
- **Size:** +308 / −3

## Summary
Adds two-stage tool-name repair to `Scheduler` so model-emitted hallucinated tool names get rewritten to a valid registry entry instead of failing outright. Stage 1 = mechanical normalization (`read-file` → `read_file` via `replaceAll('-', '_')`). Stage 2 = fuzzy match against the available-tools list using Levenshtein distance ≤ 2, with explicit ambiguity rejection (multiple matches at the same minimum distance → no repair). The original name is preserved on the request as `originalRequestName` for audit/replay.

## Specific observations

1. **Repair is two-pass with explicit failure semantics** at `packages/core/src/scheduler/scheduler.ts:166-207`. First `normalizeToolName(originalName)` only triggers a registry lookup if it changed the string (`if (normalizedName !== originalName)`). Second, if normalization didn't help, `getClosestMatch(originalName, toolRegistry.getAllToolNames(), 2)` runs and accepts the result *only if* `!fuzzyResult.isAmbiguous`. Both stages log via `debugLogger.log` so the repair trail is visible without polluting normal output.

2. **Ambiguity is the load-bearing decision** at `fuzzy-matcher.ts:357-376`. The matcher tracks `minDistance` and `matches: string[]`; on tie at the minimum distance the array grows and `matches.length > 1` returns `{ isAmbiguous: true, repairedName: undefined }`. Test at `fuzzy-matcher.test.ts:279-287` exercises exactly this with `['read_file_1', 'read_file_2']` against `'read_file'` (distance 2 to both). This is the right call — silently picking either match would let "I think I meant `read_file_1`" land on `read_file_2` ~50% of the time, which is worse than failing.

3. **Security limit** at `fuzzy-matcher.ts:351-354`: `if (hallucinatedName.length > 64) return { isAmbiguous: false, distance: Infinity }`. Levenshtein is O(n·m) and a sufficiently long adversarial name against many tools could be a CPU-exhaustion vector. 64 is a reasonable cap (matches typical max identifier length) — pinned by the test `it('returns no match if the name is too long (security limit)')` at line 295. Worth noting the limit applies only to the input name, not to the registry names — if someone registered a 1000-char tool name (legal but weird), the comparison still runs against it.

4. **The repair is also applied on the followup-tool-call rerun path** at `scheduler.ts:817-825`. This is subtle and important: when a tool call's "tail" (continuation) re-dispatches, the model may again emit the hallucinated name. Without applying repair on the rerun, you'd get a fix-once-fail-forever pattern. The PR threads `repairedTailName` through `newRequest.name = repairedTailName || tailRequest.name` at line 824, which is correct.

5. **`originalRequestName` preservation** at `scheduler.ts:325-329` only sets the field if it wasn't already set (`enrichedRequest.originalRequestName = enrichedRequest.originalRequestName || request.name`). Good — preserves the *first* hallucinated name across multiple repair passes, so audit logs show the model's original mistake, not the intermediate-state name.

6. **Levenshtein library choice (`fast-levenshtein`)** is fine — it's a 9-year-stable C-extension-optional pure-JS implementation. Worth noting it counts substitutions/insertions/deletions equally; the more sophisticated Damerau-Levenshtein (which handles transpositions like `raed_file` → `read_file` in distance 1 instead of 2) might be a better fit for typo-class hallucinations specifically. Future enhancement.

7. **Test coverage at `scheduler.test.ts:1418-1524` covers exactly the three branches:**
   - kebab-to-snake repair → succeeds, `originalRequestName: 'read-file'` preserved
   - fuzzy match (distance 2) → succeeds, `originalRequestName: 'rd_file'` preserved
   - ambiguous match → fails, status remains `Error`

   Plus 7 unit tests at `fuzzy-matcher.test.ts` covering the matcher itself in isolation. Solid layered coverage.

## Risks

- **Behavioral change for tool authors.** Anyone who *intentionally* relied on `read-file` failing (e.g. to detect a misconfigured client) now sees it succeed silently. Unlikely, but a one-line release note covers it.
- **Fuzzy match success rate of distance-2 across a large tool registry can produce surprising matches.** With ~30+ tools, the probability of an unrelated 5-char hallucinated name landing within distance 2 of an unrelated valid tool is non-negligible. The ambiguity check helps but doesn't eliminate single-best-match-by-coincidence. Worth monitoring telemetry on `Repaired tool name:` log lines once shipped.
- **`originalRequestName` is only logged via `debugLogger.log`** which most users won't see. If repair becomes the primary source of "wait, I asked for X but the agent ran Y" surprise, surfacing it as a once-per-call user-visible note (or a footer indicator) would help debuggability.
- **No telemetry counter** on repair-success vs repair-fail — would be useful to gauge real-world hallucination rates and tune `maxDistance` based on data.

## Suggestions

- **(Recommended)** Add a telemetry counter (`tool_call.repair.{normalized|fuzzy|ambiguous_reject|miss}`) so the team can tune `maxDistance` empirically.
- **(Recommended)** Consider surfacing the repair to the user (not just `debugLogger`) as a one-line `"(repaired from `read-file` → `read_file`)"` annotation on the tool-call card. Otherwise users get confused when they ask for the wrong name and get "the right" answer.
- **(Optional)** Switch to Damerau-Levenshtein for transposition-class typos.
- **(Optional)** Add a short doc comment on `_tryRepairToolName` describing the precedence (normalization first, fuzzy second, ambiguity rejected).
- **(Nit)** The 64-char cap is a magic number; consider hoisting to `const MAX_FUZZY_MATCH_INPUT_LENGTH = 64` with a brief justification comment.

## Verdict: `merge-after-nits`

Well-scoped feature with explicit failure semantics, layered tests, and a security cap. The two real concerns (telemetry visibility, user-facing repair indication) are improvements, not blockers.

## What I learned

The "repair under ambiguity" pattern — silently rewriting input requires explicit handling of the case where there's no unique correct rewrite. The PR's choice to track a `matches[]` array and check `length > 1` is the right shape; the subtler alternative ("pick the alphabetically-first match" or "pick the one that registered first") would feel deterministic but make hallucination-debugging much harder because the same hallucinated name produces different results depending on registration order.
