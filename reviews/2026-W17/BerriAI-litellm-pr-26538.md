# BerriAI/litellm #26538 — fix(fireworks_ai): modernize chat transforms, add Messages + Responses API

- **Repo**: BerriAI/litellm
- **PR**: #26538
- **Author**: frdeng
- **Head SHA**: 3e9cf5d5d137ad5101672528db35632b7f2f031d
- **Link**: https://github.com/BerriAI/litellm/pull/26538
- **Size**: ~1467 diff lines across `litellm/__init__.py`, `_lazy_imports_registry.py`, `llms/fireworks_ai/{chat,messages,responses,cost_calculator}/...`, `utils.py`, plus tests.

## What it changes

Three things bundled in one PR (claims "1 specific problem", but it's
really three):

1. **Chat transform refactor**: `FireworksAIConfig.map_openai_params`
   in `litellm/llms/fireworks_ai/chat/transformation.py:174-189`
   becomes much shorter — it drops the `tool_choice="required" → "any"`
   remap (#4416 workaround), drops the `response_format` + `tools`
   coexistence carve-out (the `_add_response_format_to_tools`
   fallback), drops the `json_schema` → `json_object` schema unwrap,
   and drops the `max_completion_tokens → max_tokens` rewrite. New
   pass-through is essentially `optional_params[param] = value`.
2. **`get_models` rewrite** (`:419-485`): now paginates the management
   API with `pageSize=200 / pageToken`, adds `supports_serverless=true`
   filter for the public `accounts/fireworks/models` namespace, and
   only adds the user's `FIREWORKS_ACCOUNT_ID` namespace if it's set
   *and* not equal to `"fireworks"`. `FIREWORKS_ACCOUNT_ID` becomes
   optional (was required); base URL stripping handles the
   `/inference/v1` suffix variant.
3. **Two new API surfaces**: `messages/transformation.py` (62 lines,
   subclasses `AnthropicMessagesConfig` to expose Fireworks's
   Anthropic-compatible `/v1/messages` endpoint) and
   `responses/transformation.py` (54 lines, subclasses
   `OpenAILikeResponsesConfig` for `/v1/responses`). Both with API-key
   resolution via the same four env-var fallback chain
   (`FIREWORKS_API_KEY` / `FIREWORKS_AI_API_KEY` /
   `FIREWORKSAI_API_KEY` / `FIREWORKS_AI_TOKEN`).

Also adds `custom_llm_provider` properties on all three configs and
removes the `supports_reasoning(...)` gate on `reasoning_effort`
(`:148-167`) — the docstring at `:39-90` says Fireworks "accepts it on
all models and handles unsupported cases itself". Cost calculator
gets cached-token tracking and a separate cached-input price knob
(`cost_calculator.py:58-89`).

## Strengths

- The new `get_models` implementation in
  `chat/transformation.py:419-485` is genuinely better: it paginates
  (the old one didn't, so any account with > default page size was
  silently truncated), it doesn't require `FIREWORKS_ACCOUNT_ID` for
  the common case (you're querying the *public* Fireworks catalog),
  and it gracefully degrades on per-account 4xx by warning and
  continuing instead of raising. The dedupe via `seen: set` (`:455`)
  is correct — when both targets are queried, the same model name
  could appear twice.
- Tests cover the new shapes: 246 lines added to
  `test_fireworks_ai_chat_transformation.py:145-390` exercise the
  paginate-and-merge path, the URL-suffix-stripping for
  `/inference/v1`, and the env-var fallback chain.
- Splitting Messages and Responses into their own subdirectories
  matches the existing layout convention (`embed/`, `chat/`).

## Concerns / asks

- **`FIXME` / `verbose_logger.warning` on every page failure
  (`:474-479`) is the right call, but `break` on the first failed
  page silently truncates the model list mid-pagination.** A user
  whose first page succeeds and second page 503s will see a
  partial list with no error to surface upstream. Consider
  retry-with-backoff for transient 5xx, or at minimum bubble a
  `Warning` to the caller via the return value (e.g. include a
  `partial=True` flag) so the proxy admin UI can show "list may be
  incomplete".
- **`map_openai_params` regression risk is high.** The deletions at
  `:175-205` remove three documented Fireworks quirks:
  - `tool_choice == "required"` → `"any"` rewrite came from #4416.
    Fireworks's docs may have caught up, but the PR body doesn't
    cite a Fireworks API changelog confirming "required" is now
    accepted. If it isn't, every caller using strict tool-choice
    will start getting 400s.
  - The `response_format` + `tools` coexistence ban was a real
    Fireworks server-side restriction. Now both flow through
    untouched. Same risk: if the server still rejects the combo,
    callers see a 400 they used to be insulated from.
  - `max_completion_tokens` → `max_tokens` rename: if Fireworks's
    `/chat/completions` still only accepts `max_tokens`, requests
    that previously worked will now ship the wrong field.
  All three should either be (a) cited as no-longer-needed with a
  link to the Fireworks API doc that confirms the change, or
  (b) preserved with deprecation comments. The PR body doesn't do
  either.
- **`reasoning_effort` is now always passed through** (`:151-167`,
  `supports_reasoning` import deleted at `:30`). Comment says
  Fireworks "handles unsupported cases itself", but "handles"
  could mean "ignores", "rejects with 400", or "interprets as
  high". No test asserts the behavior. Worth one round-trip test
  against a non-reasoning model in CI.
- **Three-features-in-one-PR violates the PR template's own
  "isolated as possible, only solves 1 specific problem" checkbox**
  (which is *checked* in the body — `[x] My PR's scope is as
  isolated as possible`). Should be three PRs: get_models pagination
  fix; chat-transform deletions (with justification per quirk);
  Messages + Responses surfaces. Each is independently reviewable,
  this combined diff is not.
- **`FireworksAIMessagesConfig.get_complete_url`
  (`messages/transformation.py:54-62`) appends `/v1/messages` to
  any base URL that doesn't already end with it.** If a user passes
  `api_base="https://example.com/inference"` (no `/v1`), the
  fall-through branch produces `.../inference/v1/messages` — fine.
  But if they pass `https://example.com/v1/foo`, you get
  `.../v1/foo/v1/messages`. Add a sanity check that the final URL
  is one of the two expected suffixes, or document the constraint.
- The new `_fetch_models_from_account` (`:455-485`) infinite loop
  trusts the API to eventually return no `nextPageToken`. A bug on
  the Fireworks side (token never empty, same page returned) would
  hang the proxy thread. Add a `MAX_PAGES = 50` safety bound.
- No tests for the new `messages/` or `responses/` configs — only
  the chat transform got coverage. For new files claiming Anthropic
  Messages and OpenAI Responses parity, that's a gap.

## Verdict

**request-changes** — the `get_models` improvement and the new API
surfaces are valuable, but bundling them with three undocumented
deletions of known-Fireworks-quirks workarounds in
`map_openai_params` is too risky to merge as one. Split into
get-models-pagination (mergeable), api-surface-additions (mergeable
with tests), and chat-transform-cleanup (needs per-quirk justification
linking to a Fireworks API change confirming the workaround is no
longer needed). Without that justification, this PR will silently
regress every caller that depends on the dropped param mappings.

## What I learned

When a PR description checks "scope is as isolated as possible" but
the diff touches an `__init__.py`, a lazy-imports registry, four
provider files, the global `utils.py`, and adds two new API
surfaces, the checkbox is aspirational. Reviewers should treat it
as advisory and ask for a split. The harder lesson is that
"modernize" in a PR title is almost always a red flag — it bundles
"new feature" (the API surfaces) with "remove old workarounds"
(the param-mapping deletions), and the workaround removals are the
ones that silently break production.
