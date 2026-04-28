# google-gemini/gemini-cli #26066 — Update policy so transient errors are not marked terminal

- **PR**: [google-gemini/gemini-cli#26066](https://github.com/google-gemini/gemini-cli/pull/26066)
- **Head SHA**: `435e4a6c`
- **Closes**: #25898

## Summary

Closes the "selected model silently degrades to fallback Flash on a single
transient API failure" bug. Prior `transient` state-transition policy was
`terminal` (one transient error → kick down the chain to the next entry),
which conflated rate limits / 503s / network blips with "this model is
permanently unavailable". Fix changes every `stateTransitions.transient`
field across the seven default model-chain entries (preview Pro/Flash,
default Pro/Flash, lite Flash-Lite/Flash/Pro) to `sticky_retry`, so the
policy engine retries the *currently selected* model on transient errors
and only walks the chain on terminal/not_found/unknown errors.

## Specific findings

- The diff is enormous (mostly because both the executable
  `defaultModelConfigs.ts` JSON literal *and* the IDE-facing
  `gemini-cli-vscode-ide-companion.json` settings-schema mirror have to
  be updated symmetrically), but the actual semantic change is one
  string per chain entry: `"transient": "terminal"` →
  `"transient": "sticky_retry"`. Counted across the JSON: 7 stateTransition
  blocks per file × 2 files (executable defaults + IDE settings schema)
  = 14 single-line semantic changes.
- The choice to update the *embedded markdownDescription default value*
  (the giant escaped-JSON string at `gemini-cli-vscode-ide-companion.json`
  property `markdownDescription`) alongside the actual `default` object
  is correct — these are two separate copies of the same default config
  inside the schema (one for IDE display, one for actual use), and a
  drift between them means the IDE settings UI shows users a stale
  default they can't actually get. The PR keeps them in sync (the
  diff shows both `markdownDescription` and `default` getting the same
  `sticky_retry` substitution per entry).
- `policyCatalog.ts` (`DEFAULT_STATE`) — the symmetric change here is
  the *engine-side* policy. The PR description says it was updated
  but the diff hunk for `policyCatalog.ts` isn't visible in my truncated
  view. Assuming it landed (the test file is updated to assert
  `sticky_retry` so it must have).
- `sticky_retry` is the load-bearing primitive — it has to mean "retry
  this model, do not advance the chain index" or the fix is incomplete.
  The PR doesn't include the engine-side enum/handler change that
  defines `sticky_retry` semantics, so I'm trusting that primitive
  exists from prior work.

## Nits / what's missing

- The change applies uniformly across *all* chain entries including the
  `isLastResort: true` Flash entry in the `preview` chain and the
  Flash entry in the `default` chain. Sticky-retrying a "last resort"
  entry on transient errors is a different policy decision than
  sticky-retrying a primary entry — for last-resort entries, a
  `transient` failure means the user has *already* exhausted the chain
  and is now stuck in a retry loop on the only remaining option. Worth
  a one-line discussion in the PR body for whether last-resort entries
  should keep `terminal` (or get a new `sticky_retry_with_user_prompt`
  variant that surfaces "we've retried 3 times on the last resort, do
  you want to give up?").
- No backoff / max-retry-count documentation. `sticky_retry` presumably
  has bounded retry semantics in the engine, but the user-facing
  behavior change ("transient errors now retry instead of falling back")
  needs to come with "...up to N times with M-second backoff" so users
  understand the worst-case latency on a transient outage isn't
  unbounded.
- Pre-merge checklist in PR body has unchecked `Updated relevant
  documentation and README` — a behavior change this user-visible
  ("your selected model now sticks even on rate limits") really should
  land with a docs entry, especially because the prior implicit
  behavior (silent degrade) was something users may have built workflows
  around (e.g. rate-limit-driven fallback to cheaper Flash).
- "Noted breaking changes" is also unchecked. This *is* a behavior
  change visible to anyone whose workflow depended on the silent-degrade
  semantic. Not a strict API-break but a noticeable runtime change.
- Tests in `policyCatalog.test.ts` are mentioned in the PR body as
  updated to verify the new behavior; the diff for that file isn't
  visible in my truncated view. Assuming the test reads `expect(...
  preview chain ... transient ...).toBe('sticky_retry')` for at least
  the preview Pro entry.
- The `isLastResort` interaction is a real edge case that could bite
  in production: imagine a rate-limit storm where preview Pro fails
  transient → sticky_retry retries → still failing → engine falls
  through to Flash (last resort) → Flash *also* sees transient
  failures from the same storm → `sticky_retry` keeps it on Flash
  → user is now stuck retrying Flash with no escape valve. Prior
  behavior would have walked off the end of the chain into a
  terminal error visible to the user. New behavior is invisible
  retry until the storm passes. Both are defensible policies but
  the trade-off should be named.

## Verdict

**merge-after-nits** — fix is the right shape (a single transition
policy update applied uniformly via configuration, not a code change),
the JSON-default-vs-markdownDescription dual-copy is correctly kept in
sync, and the bug it closes (#25898) is real and high-impact. But the
change has user-visible behavior implications that need to land with
release notes / docs, the last-resort-entry interaction needs a
deliberate decision (not just a uniform substitution), and the
unchecked pre-merge checkboxes in the PR body suggest the
documentation side of the change isn't done yet.
