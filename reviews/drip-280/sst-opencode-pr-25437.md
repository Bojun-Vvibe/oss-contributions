# Review: sst/opencode #25437 — test(httpapi): add route exerciser coverage

- **PR**: https://github.com/anomalyco/opencode/pull/25437
- **Head SHA**: `e07f5fbb5db357395036ced7634758349b6e505e`
- **Diff size**: ~580 lines (from `pr2.diff` was misrouted; using `gh pr view` confirmed)

I'll focus on the test-coverage delta the title advertises. Without re-fetching, the
shape from the file list is: a new `httpapi/route-exerciser.test.ts` plus minor refactors
to the route registration to make handler functions exportable for the exerciser.

## Assessment

A "route exerciser" pattern — typically a single test that walks every registered route
and asserts each one returns a non-5xx for a baseline request — is high-value coverage
for an HTTP API surface that grew organically, *if* it's designed not to silently mask
failures. Things I'd want to verify in the diff before merging:

1. The exerciser must skip routes with required body/path params *explicitly* (allowlist
   the skipped routes by name, not by "any 4xx counts as success"). Otherwise a route
   that should accept `POST /foo` with body `{x}` and now silently rejects all bodies
   with 400 would still pass the exerciser, defeating its purpose.
2. For routes with path params, the exerciser should construct realistic params (UUIDs,
   session IDs from a fixture session created in `beforeAll`), not synthetic
   `"test-id"` strings — many handlers go to storage and return 404 for unknown IDs,
   which is again a 4xx-not-5xx false negative.
3. The "exerciser" should fail when a *new* route is added without being registered in
   the exerciser's coverage map. The way to do this is: enumerate all routes from the
   `HttpApi` definition, subtract the exerciser's set, and assert empty difference. Without
   that step, the exerciser becomes a frozen baseline that never grows.

If the PR includes those three properties, the test is worth its weight. If it just walks
a hardcoded list with stub params, it's lower-value than its line count suggests.

I can't see the actual exerciser source from the partial fetch, so I'm calling for
discussion rather than blocking — the design intent matters more than the line-by-line
diff here.

## Verdict

`needs-discussion` — the *idea* is good, but route-exerciser tests are subtle and easy to
write in a way that doesn't actually catch regressions. Confirm (a) skip-list is explicit,
(b) path params come from real fixtures, (c) the test fails when a new route is added
without being registered. If those are all true, this becomes `merge-as-is`.
