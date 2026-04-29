# google-gemini/gemini-cli#26204 — perf(core): eliminate redundant serialization in ToolOutputMaskingService

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26204
- **Head SHA**: `76844c9e5529462ff89bcd4418d3f5130badf4e6`
- **Size**: +295 / −185, 4 files
- **Files**: `packages/core/src/context/toolOutputMaskingService.{ts,test.ts}`, `__snapshots__/toolOutputMaskingService.test.ts.snap`, `packages/core/src/utils/tokenCalculation.ts`
- **Fixes**: #24028

## Context

`ToolOutputMaskingService.mask()` is on the hot path of every conversation turn — it scans recent tool outputs to decide which ones to disk-offload + summarize. Profiling indicated multiple `JSON.stringify` calls per part: one in `getToolOutputContent()` for the disk write, another inside `estimateTokenCountSync()` for the heuristic token estimate, plus a serialization-based check in `isAlreadyMasked()`. For long shell-output histories, this dominated the masking pass.

The PR makes three correctness/perf changes plus two security hardenings, packaged together.

## Why this exists / problem solved

Per the PR body and the diff, the four motivations are:

1. **Perf**: kill the 2nd and 3rd `JSON.stringify` per part by reusing the serialization done in `getToolOutputContent`, and replace the structural `isAlreadyMasked` check with an `isRecord` shape check that doesn't serialize at all.
2. **Correctness regression fix**: previously-masked tool responses were preserving unwanted metadata like `exitCode` in the conversation history; now the masked part is a clean `{output: <snippet>}` object.
3. **Security**: prompt-injection hardening — sanitize the preview snippet before embedding it in the masked output, and detect spoofed `<masking_indicator>` tags so an attacker who controls tool output can't fake a "this is already masked, skip me" signal.
4. **Type safety**: drop unsafe `as` casts in favor of `isRecord` guards from `markdownUtils.ts`, complying with project ESLint rules without `eslint-disable` lines.

This is a lot for one PR — it's really four PRs in a trench coat.

## Design analysis

### Perf change: single-pass serialization

Old flow per part: `getToolOutputContent(part)` (stringify #1) → `estimateTokenCountSync([part])` (stringify #2 internally) → if masked, build `maskedPart` → `estimateTokenCountSync([maskedPart])` (stringify #3).

New flow at `toolOutputMaskingService.ts:131-138`: `getToolOutputContent(part)` returns the serialized string, then `partTokens = Math.floor(estimateTextTokens(toolOutputContent) + (toolName?.length ?? 0) / DEFAULT_CHARS_PER_TOKEN)`. The masked-part estimate at `:251-254` follows the same shape on the masked snippet string.

This is the right collapse. `estimateTextTokens` is the underlying char-count-based heuristic that `estimateTokenCountSync` was wrapping; calling it directly on an already-stringified payload skips the redundant `JSON.stringify` inside `estimateTokenCountSync`. The `+ toolName.length / DEFAULT_CHARS_PER_TOKEN` at `:135` is the small accounting for the `name` field that `estimateTokenCountSync` would have included via the part's outer JSON shape — a reasonable approximation, though it ignores the JSON braces/quotes/`functionResponse` wrapper which is ~30 chars (~7-8 tokens) of overhead per part. For per-part token budgets in the tens-of-thousands range this is noise; for per-part budgets in the hundreds it could be 5-10% off. The PR doesn't quantify the heuristic divergence — would be good to know.

`estimateTextTokens` and `DEFAULT_CHARS_PER_TOKEN` are now exported from `tokenCalculation.ts:5-6` per the diff (`+5 / -2`). The export shape is fine.

### Correctness change: isAlreadyMasked now takes the response object, not the string

At `toolOutputMaskingService.ts:309-321`, `isAlreadyMasked(response: unknown)` now:

```python
if isRecord(response):
    keys = Object.keys(response)
    if keys.length !== 1 || keys[0] !== 'output':
        return false
    output = response['output']
    return ...  # check for masking indicator tag
```

This is the *spoof-prevention* hardening. The old `isAlreadyMasked(content: string)` was a substring match on `<masking_indicator` — meaning a tool whose output literally contained the string `<masking_indicator>I'm pretending to be masked</masking_indicator>` would silently get skipped over by the masker, even though it wasn't a real mask. The new check requires the response to be a `{output: <string-with-tag>}` object with **no other keys** — a real mask is a single-key object, so a spoof attempt with extra keys (`{output: "<masking_indicator>...", malicious: "..."}`) is correctly identified as *not* already masked. The new test at test.ts:351-396 explicitly covers this (`it('should prevent spoofing of the masking indicator tag')`).

This is a real security improvement, not just a refactor. A tool-output prompt injection where the attacker tries to exempt their content from masking by spoofing the indicator now fails. Worth calling out in the CHANGELOG explicitly because it's a behavior change attackers will care about.

### Correctness change: clean masked response

At `toolOutputMaskingService.ts:206-211`, the masked part's `originalResponse` is built via `isRecord(part.functionResponse.response) ? part.functionResponse.response : {}` instead of the old `as Record<string, unknown>` cast with `|| {}` fallback. Practically equivalent for happy paths, but the `isRecord` guard is honest about the type narrowing and the empty-object fallback is the same.

The bigger fix described in the PR body — "preserved unwanted metadata like `exitCode`" — is NOT visible at the lines I read but the snapshot diff confirms behavior change. Look at `__snapshots__/toolOutputMaskingService.test.ts.snap`:

```
- Line\nLine\n... (one per line, with "[6 lines omitted]" middle marker)
+ Line Line Line Line Line Line ... [9981 lines omitted ... Line Line Line Line
```

The new snapshot collapses the per-line content into a *space-separated* preview rather than preserving newlines. **This is a meaningful UX regression**: shell-output previews lose their line structure, and a user reading the masked snippet in their conversation history can no longer tell that the original output was 10000 lines of "Line\n" — they see "Line Line Line ... [9981 lines omitted] ... Line Line Line" with no newline-shape preserved. For shell tools specifically, line-breaks are *the* semantic structure. The PR body doesn't acknowledge this snapshot change. **This needs to either be reverted or explicitly justified.**

### Security change: preview sanitization

The preview is now sanitized via `this.sanitizePreview(preview)` at `:236` before being embedded in the masked snippet. The test at test.ts:587-632 (`should sanitize tool output to prevent prompt injection`) covers this — `</INJECTION>` in the original tool output becomes `&lt;/INJECTION&gt;` in the snippet, while the legitimate wrapper `<masking_indicator>` tags are preserved.

This is exactly the right hardening. A tool that returns text containing `</INJECTION> <new_system_prompt>...</new_system_prompt>` could otherwise inject directives into the model's context via the masked-output preview. HTML-escaping the preview while preserving the wrapper tags closes that vector.

### Cosmetic change: home-dir replacement

At `:234`, `filePath: filePath.replace(os.homedir(), '~')` is applied before embedding the path in the masked snippet. Reasonable cosmetic — the user sees `~/...tool-outputs/...` instead of `/Users/bojun/...tool-outputs/...`. Two notes:

1. `os.homedir()` is called per-mask. For a long history this is N calls; cache once at construction time for a small win, but probably not worth a separate change.
2. Single-shot `String.prototype.replace(string, string)` only replaces the first occurrence. If the file path is somehow under `~/x/~/y` (unusual but possible on user-controlled directory naming), only the first `~` is folded. Use a regex-anchored replace or `replaceAll`, but again this is a minor nit.

### Test refactor

The tests now spy on `service.getToolOutputContent` directly instead of mocking the module-level `estimateTokenCountSync`. This is a cleaner test seam — the masking service's behavior is now tested in terms of "given content of this size, what does it do" rather than "given a synthetic token count, what does it do." The thresholds in the test cases are also bumped (60000 → 240000 chars, etc.) to compensate for the heuristic-vs-mocked-count change. The `mockedEstimateTokenCountSync.mockImplementation` calls are gone; in their place are `vi.spyOn(service, 'getToolOutputContent').mockImplementation(...)`.

This is genuinely a better test design. The downside: the test cases now exercise the *real* `estimateTextTokens` heuristic, so if anyone tunes that heuristic, several test thresholds may need readjusting. Worth a comment at the top of the test file noting "thresholds depend on `estimateTextTokens` heuristic; adjust together."

### `getToolOutputContent` is now `/** @internal */`

At `:294`, the visibility is widened from `private` to public-but-internal so the spy can attach. This is a common test-seam pattern in TypeScript projects without proper DI. The `@internal` JSDoc tag is the right marker. Worth checking the project's `tsconfig.json` has `"stripInternal": true` for consumer-facing builds so this doesn't leak into the public API surface.

## Risks

- **Snapshot regression in masked-preview line structure** is the biggest concern. The snapshot diff at `__snapshots__/toolOutputMaskingService.test.ts.snap:6-26` shows a real UX change (newlines collapsed to spaces in the preview). Either revert the preview formatter to preserve `\n`, or document this as an intentional density tradeoff. As-written, this is a regression for shell tools.
- **Heuristic divergence**. `estimateTextTokens(content) + name.length/CHARS_PER_TOKEN` is not equivalent to `estimateTokenCountSync([part])` — the latter accounts for the `functionResponse` JSON wrapper (~30 chars). For small parts this 5-10% undercount could cause a part to slip below the masking threshold that previously would have been masked. The user impact is small (one fewer mask per turn in pathological cases) but worth a regression test asserting equivalent behavior at the boundary.
- **Spoof-detection narrowness**. The new `isAlreadyMasked` requires `keys.length === 1 && keys[0] === 'output'`. If the masking format ever evolves to add a `metadata` key (e.g., `{output: "<...>", chunkCount: 3}`), this check will start treating real masks as not-yet-masked, causing double-masking. Add a test that pins the current canonical-mask shape and a `// when changing this, update isAlreadyMasked` comment.
- **Test threshold fragility** as noted above. The 240000-char thresholds are heuristic-dependent.
- **PR scope**: four motivations in one PR. Perf change, two security changes, and a correctness fix bundled. The security changes (anti-spoof + preview sanitization) deserve their own PR with explicit security-focused review and CHANGELOG callouts.

## Suggestions

1. **Address the snapshot newline-collapse before merge.** Either revert `'Line\n'.repeat(10000)` collapsing to space-joined in the preview (preserve `\n` in the formatter), or add a PR-body explanation of why density-over-structure is the right tradeoff for masked previews. As-currently-written, shell-tool previews lose their semantic structure and this is the *only* signal a user has about what the masked output was.
2. **Split the PR.** Land (a) the perf single-pass + heuristic in one PR, (b) the spoof-prevention `isAlreadyMasked` rewrite + preview sanitization in a security-tagged PR, (c) the `os.homedir()` cosmetic and test-refactor as a third. Each is reviewable in isolation; the bundled diff is 295 lines and asks the reviewer to context-switch four times.
3. **CHANGELOG entries** for the two security hardenings — anti-spoof of masking indicator and preview HTML-escaping. Both are user-visible behavior changes that security-conscious users (especially anyone running goose/gemini-cli against untrusted tool servers) need to know about.
4. **Add an equivalence test** that pins `estimateTextTokens(content) + name.length/CHARS_PER_TOKEN` ≈ `estimateTokenCountSync([part])` within a small tolerance, so future drift in either function is caught.
5. **Pin the canonical-mask shape** in a constant or test fixture and `// HOLD-OPEN: when changing the canonical mask shape, update `isAlreadyMasked` key check at toolOutputMaskingService.ts:312`.
6. **Cache `os.homedir()` once** at service construction to avoid the per-mask call. Negligible perf, but cleaner.

## Verdict

**request-changes** — the perf and security wins are real and the test refactor is genuinely better, but the snapshot diff at `__snapshots__/toolOutputMaskingService.test.ts.snap` shows masked shell-output previews lose newline structure (`Line\nLine\n...` becomes `Line Line ...`), which is a UX regression the PR doesn't acknowledge. Either restore newline preservation in the preview or justify the density tradeoff in writing. Also strongly recommend splitting into three PRs (perf / security / cosmetic+test-refactor) so the security changes get the focused review they deserve. Once the snapshot is addressed, this is a `merge-after-nits`.

## What I learned

Bundling perf, security, and cosmetic changes in one PR is a recurring anti-pattern in mature OSS repos because each motivation has a different review-attention cost: perf changes need benchmarks, security changes need threat-model thinking, and cosmetic changes are basically taste. When they're packaged together, the reviewer's attention budget gets split and the security change in particular tends to get under-scrutinized — which is precisely the wrong outcome. The right practice for a 4-motivation refactor is "land the security pieces in their own PRs first, with explicit security-tag and CHANGELOG callouts, then the perf piece, then the cosmetic." It costs the contributor more PRs but it pays back in reviewer signal-to-noise and post-hoc auditability — when someone in 2027 runs `git log --grep="security"` looking for "when did we add anti-spoof to masked-output detection," they want to find a single focused commit, not "perf(core): eliminate redundant serialization in ToolOutputMaskingService" with the hardening buried in line 312.
