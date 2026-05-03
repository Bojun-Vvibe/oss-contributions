# BerriAI/litellm PR #27070 — fix(ui): omit unchanged allowed_routes on key update

- Author: fengfeng-zi
- Head SHA: `483c20ee865b7a1ced6a318b1df502f27f9d64d7`
- Diff: +94 / -0 across 2 files
- Files: `ui/litellm-dashboard/src/components/templates/key_info_view.test.tsx`, `ui/litellm-dashboard/src/components/templates/key_info_view.tsx`

## Observations

1. **The actual fix is `key_info_view.tsx:201-208`**: before calling `keyUpdateCall`, the form payload is normalized — if `formValues.allowed_routes` is order-and-content-equal to `currentKeyData.allowed_routes`, the field is `delete`d from the payload. Comment correctly explains the motivation: non-admin editors trip a backend "setting allowed_routes" permission check on every save, even when they didn't touch the field.
2. **`normalizeStringList` (`key_info_view.tsx:50-65`) is reasonable but slightly over-broad**: it accepts both arrays and comma-separated strings, trimming and dropping empties. This is fine for the comparator path, but be aware that `["a", "a"]` vs `["a"]` would *not* be considered equal (length differs) — and `[" a", "a "]` *would* be (after trim). That seems correct for route names, just worth noting.
3. **`areStringListsEqual` (`key_info_view.tsx:67-75`) is order-sensitive**: `["a","b"]` vs `["b","a"]` would be considered different and re-send the payload. For `allowed_routes` this is probably correct (the backend stores order), but if the backend treats them as a set the diff will produce a false-negative "changed" signal that re-trips the same permission check. Worth a quick check on the API contract — if order is irrelevant, sort both before comparing.
4. **Test `key_info_view.test.tsx:631-649` covers the no-op-with-existing-routes path**: asserts `expect.not.objectContaining({ allowed_routes: expect.anything() })`. Clean.
5. **Test at `key_info_view.test.tsx:651-669` covers `[] === []`** (both empty), and the third test at `:671-689` covers the *real* change path (existing `["management_routes"]` cleared to `[]`) — proving the delete-on-no-op logic doesn't suppress legitimate clears. This is the test I'd write to feel safe shipping. Good.
6. **No backend change**: this is purely a client-side workaround for the over-aggressive permission check. The cleaner long-term fix is on the proxy side (don't gate on payload presence, gate on actual mutation), but as a low-risk UI patch this is fine.

## Verdict: `merge-after-nits`

Tight, well-tested, motivation is clearly documented inline. Confirm whether `allowed_routes` order matters server-side — if not, `areStringListsEqual` should sort both lists first to avoid false "changed" signals on cosmetic reorderings.
