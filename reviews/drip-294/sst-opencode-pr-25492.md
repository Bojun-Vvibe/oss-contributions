# sst/opencode PR #25492 — fix(provider): limit JSON schema depth for Moonshot/Kimi

- **Repo:** sst/opencode
- **PR:** #25492
- **Head SHA:** `2e1517dfccf8adf64c0b544c97f76469b3e01449`
- **Author:** nateGeorge
- **Title:** fix(provider): limit JSON schema depth for Moonshot/Kimi
- **Diff size:** +162 / -3 across 2 files
- **Drip:** drip-294

## Files changed

- `packages/opencode/src/provider/transform.ts` (+35/-3) — adds `MAX_DEPTH = 9`, `isComplex(node)` and `flatten(node)` helpers, and threads a `depth` parameter through `sanitizeMoonshot`. When `depth >= 9` and the node is complex (`object` / `array` with `properties` / `anyOf` / `items` / `$defs` etc.), it flattens to `{ type: "object", additionalProperties: true }` (or array-of-object).
- `packages/opencode/test/provider/transform.test.ts` (+127/-0) — two regression tests: a synthetic 12-level deep object schema, and a real-world deeply-nested `anyOf` chain (Klaviyo-shaped campaign-message schema).

## Specific observations

- `transform.ts:1104` — `MAX_DEPTH = 9 // Kimi K2.6 rejects schemas with nesting > 10; leave headroom`. Good that the headroom rationale is in the comment. Worth noting the limit applies to *every* call site that runs the moonshotai branch, including non-K2.6 Moonshot models — if K2.0/K2.5 happily accept depth-12, those callers now silently lose schema fidelity. Consider gating on `model.api.id` for Kimi-2.6 specifically, or at minimum documenting that depth-9 is the new floor for all Moonshot/Kimi models.
- `transform.ts:1106-1116` — `isComplex` checks `node.type && !["object", "array"].includes(node.type)` to early-return false. Good. But the OR chain (`properties || anyOf || ...`) treats a node with *only* `$defs` / `definitions` as complex; flattening one of those drops the entire definitions subtree and any sibling `$ref` resolution will fail. Probably rare since `$ref` is short-circuited at line 1133, but worth a sanity check.
- `transform.ts:1117-1122` — `flatten()` returns `additionalProperties: true` which is what Moonshot accepts but is also schema-blind: the model loses every field name/description below the cutoff. This is the right trade-off for "don't crash" but the flattening is silent — at minimum `verbose_proxy_logger` / equivalent debug log would help when a user wonders "why does the model never fill in nested field X?".
- `transform.ts:1131` — array recursion increments `depth + 1`, which is correct (an array is a depth level by itself in schema-tree terms).
- `transform.test.ts:1024-1058` — the depth assertion `expect(depthReached).toBeGreaterThanOrEqual(4).toBeLessThan(12)` is loose — pin it to `expect(depthReached).toBe(9)` (or whatever MAX_DEPTH evaluates to) so a future tweak of MAX_DEPTH actually fails the test instead of silently passing.
- `transform.test.ts:1060+` — the Klaviyo-shaped test is excellent context, but the only assertion (`definition.type === "object"`) doesn't actually verify flattening happened — it just verifies the schema didn't crash. Add an `additionalProperties === true` check at the cutoff depth to make the regression precise.

## Verdict: `merge-after-nits`

Correct fix, well-tested in spirit. Tighten the depth assertion, log when flattening triggers, and consider scoping to K2.6 specifically rather than all Moonshot models.
