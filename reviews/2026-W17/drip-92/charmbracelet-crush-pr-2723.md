---
pr: 2723
repo: charmbracelet/crush
sha: f183594bf5b9f39baea7f77202fe82d8b8a6a8a2
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2723 — Kagi search

- **Author**: taoeffect (PR description: "written under my supervision and testing by GPT-5.5")
- **Head SHA**: f183594bf5b9f39baea7f77202fe82d8b8a6a8a2
- **Size**: +1066/-18 across 18 files. Adds Kagi as a second search engine alongside DuckDuckGo, plus a config schema, two new TUI dialogs (engine picker + Kagi-key prompt), and tests.

## Scope

Closes #2664. New `tools.web_search.{search_engine, kagi_api_key}` config keys, a `SearchEngine` typed enum, a `kagi` HTTP path against `https://kagi.com/api/v0/search`, an optional `search_engine` parameter on the `web_search` tool for per-call override, and TUI plumbing (`select_search_engine` command, `SearchEngines` dialog, `KagiAPIKeyInput` dialog). DuckDuckGo path unchanged.

## Specific findings

- `internal/agent/tools/search.go:94-127` (`searchKagi`) — clean: typed status-code mapping for 401/403/`402 PaymentRequired` ("balance exhausted"). The `parseKagiSearchResults` walker at `:135-160` correctly filters `t != 0` (Kagi returns mixed result types — `t:0` is web result, `t:1` is "related queries" lists), trims snippets, appends the `published` date in parens, and respects `maxResults`. Good.
- `internal/agent/tools/search.go:295-308` (`maybeDelaySearch`) — the rate-limit refactor is correct: previously held the mutex across `time.Sleep`, now snapshots the delay under lock then sleeps unlocked then takes the lock again to update `lastSearchTime`. **Bug**: this only delays DuckDuckGo (caller in `internal/agent/tools/search.go:~460`); Kagi requests skip `maybeDelaySearch()` entirely. Probably intentional (paid API, no scraping etiquette), but worth a one-line comment.
- `internal/config/config.go:349-396` (`SearchEngine`, `ToolWebSearch`, `Engine()`, `ResolvedKagiAPIKey()`) — `Engine()` silently coerces invalid values to DuckDuckGo at `:381-385`. Reasonable for forward-compat, but means a typo like `"search_engine": "duckduck"` ships zero diagnostic. A `slog.Warn` on unknown values would help.
- `internal/agent/tools/search.go:~445-450` (web_search dispatch) — the `params.SearchEngine` per-call override goes through `config.SearchEngine(params.SearchEngine).Valid()`, returning `unsupported search_engine: ...` on invalid. But there's no path validating that the user has actually configured a `kagi_api_key` before the model picks `search_engine: "kagi"`. The downstream `searchKagi` does return "kagi API key is required" (`search.go:96-98`) which surfaces as a tool error — acceptable but the model gets one round-trip to discover this.
- `internal/ui/model/ui.go:1525-1540` (`ActionSelectSearchEngine`) — nice UX: selecting Kagi without a key automatically opens the `KagiAPIKeyInput` dialog. After saving the key (`:1551-1561`), it auto-applies the engine. Two dialog `CloseDialog` calls in the success branch are correct (close both the key prompt and the engine picker).
- `internal/ui/dialog/kagi_api_key_input.go:148` and `internal/ui/dialog/search_engines.go:267` — full new dialog files with Bubble Tea key bindings, fuzzy filtering, and tests. The pattern mirrors `internal/ui/dialog/reasoning_effort.go` style.
- `schema.json:769-815` — JSON schema regeneration includes `web_search` in the required-fields list of `Tools`. This is a **breaking change** for anyone validating their existing config against the schema, since their config currently has no `web_search` block. Should be `omitzero` / non-required to preserve back-compat.
- Tests: `internal/agent/tools/search_test.go` (3 tests covering parser, max-results clamp, schema-contains-`search_engine`), `internal/config/search_test.go` (defaults + resolver + nil-resolver paths), `internal/ui/dialog/kagi_api_key_input_test.go` (empty-key rejection), `internal/ui/model/search_engine_test.go` (save + invalid-engine + display name). Solid table coverage of the new surface.

## Risk

Low for the search code itself (additive, opt-in, gracefully degrades when key is missing). Medium for the schema change at `schema.json:813` requiring `web_search`, which would break strict config validators on existing user configs.

## Verdict

**merge-after-nits** — drop `web_search` from the required list in `schema.json` (or set the field to `omitzero` everywhere), add a `slog.Warn` on unknown `search_engine` values in `Engine()`, and add a one-line comment explaining why Kagi skips `maybeDelaySearch()`. Code quality is otherwise solid and the UX of "pick Kagi → auto-prompt for key → auto-apply" is well-thought-out.
