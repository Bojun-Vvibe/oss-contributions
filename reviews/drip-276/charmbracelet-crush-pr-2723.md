# Review — charmbracelet/crush#2723

- PR: https://github.com/charmbracelet/crush/pull/2723
- Title: feat(search): kagi search
- Head SHA: `17cf64bf9a1f6ef67e040693757c4aab032ba736`
- Size: +1415 / −22 across 22 files
- Verdict: **merge-after-nits**

## Summary

Adds Kagi as a selectable web-search backend alongside DuckDuckGo. Users
can pick the engine via the command menu (new `search_engines.go` and
`kagi_api_key_input.go` dialogs) or set `tools.web_search.search_engine
= "kagi"` plus `kagi_api_key` in `crush.json`. The web-search tool now
accepts an optional per-call `search_engine` override.

## Evidence / specific spots

- `internal/agent/tools/search.go`:
  - New `kagiSearchResponse`/`kagiSearchResult` structs (lines 49-60).
  - `searchKagi` (lines 94-167): builds a `GET https://kagi.com/api/v0/search?q=...&limit=...`
    with `Authorization: Bot <apiKey>` header. Returns `[]SearchResult`.
- `internal/agent/tools/web_search.go` (+31) — adds `WebSearchOptions`
  struct (`DefaultEngine`, `KagiAPIKey`) and dispatches per request.
- `internal/agent/tools/fetch_types.go` — `WebSearchParams` gains
  `SearchEngine string` (omitempty).
- `internal/config/config.go` (+55) — config schema with `Engine()`
  and `ResolvedKagiAPIKey(resolver)` methods (the resolver lets users
  use `$KAGI_API_KEY` env-var refs).
- `internal/ui/dialog/kagi_api_key_input.go` (new, +327) and
  `internal/ui/dialog/search_engines.go` (new, +293) — TUI flows.
- `schema.json` (+26) — JSON-schema entries for the new config keys.

## Notes / nits

- `searchKagi` returns `fmt.Errorf("kagi API key is required")` on empty
  key but doesn't differentiate "user never configured it" from
  "configured but env-var unset after `ResolvedKagiAPIKey` returned
  empty". A clearer error like "set tools.web_search.kagi_api_key or
  $KAGI_API_KEY" would save support time.
- The `Authorization: Bot <apiKey>` header is correct per Kagi docs,
  but consider documenting in `web_search.md` that the key is sent on
  every search and is logged nowhere (vs DuckDuckGo which is keyless).
- `kagiSearchResult.Type int` is decoded but not acted on. Kagi uses
  `t=0` for search results, `t=1` for related searches, `t=2` for
  "wikipedia-like" knowledge cards. The current code seems to consume
  all entries indiscriminately — if `searchKagi` is meant to return
  organic results only, it should filter `r.Type == 0`. Otherwise the
  agent will get noise. Worth a unit test in `search_test.go`.
- 1.4k-line PR with three new dialog files — make sure
  `search_engines_test.go` and `kagi` test files cover the
  "switch engine mid-session" path; otherwise that's a likely
  regression target.
- Tip: capture the Kagi API rate limit / 401 response shape in
  `searchKagi` so users get an actionable error instead of the raw HTTP
  body.

Net: clean addition of a second search backend. Merge once the type-0
filter and error messages are tightened.
