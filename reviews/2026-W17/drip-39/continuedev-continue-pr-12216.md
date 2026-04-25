# continuedev/continue #12216 — docs(openrouter): document automatic identification headers

- **Repo**: continuedev/continue
- **PR**: [#12216](https://github.com/continuedev/continue/pull/12216)
- **Head SHA**: `8077f4b2cc2eec721b686b5dc1b4c38e0d09760f`
- **Author**: mvanhorn
- **State**: OPEN (+48 / -0)
- **Verdict**: `merge-as-is`

## Context

Continue's OpenRouter provider already auto-sends three
identification headers (`HTTP-Referer`,
`X-OpenRouter-Title: Continue`,
`X-OpenRouter-Categories: ide-extension`). That fact wasn't
documented anywhere user-facing, so users running their own forks
or building on top didn't know to override them. Pure docs add.

## Design

Single file: `docs/customize/model-providers/top-level/openrouter.mdx`,
+48 / -0.

1. **New "App Identification Headers" section** inserted at
   `openrouter.mdx:92` (between the existing prompt-compression
   section and "Model Capabilities"). Lists the three default
   headers with their exact values.

2. **Override instructions** explain that user-provided headers
   in `requestOptions.headers` take precedence — important
   because attribution-shaped overrides are a legitimate fork /
   reseller use case.

3. **Two tabs** at lines 100-138: a YAML config example and a
   JSON config example. Both demonstrate setting `HTTP-Referer`
   and `X-OpenRouter-Title` to user-chosen values. JSON tab
   carries the existing `(Deprecated)` qualifier so the file's
   convention is preserved.

## Risks

- **Docs-only, no code change** — the only failure mode is the
  documented behaviour drifting from the implementation.
  Worth a one-line search of the OpenRouter provider source to
  confirm:
  - The header names are exactly `HTTP-Referer`,
    `X-OpenRouter-Title`, `X-OpenRouter-Categories` (not
    `X-Title` / `X-Continue-Categories` / etc).
  - The default values match (`https://www.continue.dev/`,
    `Continue`, `ide-extension`).
  - User overrides really do take precedence (the doc claims
    this; if the implementation merges in the opposite order,
    the doc is wrong).
- **No test, no codeowner ping for the OpenRouter handler** —
  if this docs PR lands and a future code change flips header
  names, the doc silently rots. Worth wiring the OpenRouter
  default headers into a constant the docs can reference, or
  at least adding a comment in the source pointing at this
  doc page.

## Suggestions

- (Optional) Sanity-check the three header names against the
  OpenRouter provider source before merge.
- (Optional) Add a one-line snippet showing how to *suppress*
  attribution entirely (some self-hosters want zero
  identification): set the header to an empty string, or
  document that this isn't supported.
- (Nit) The `(Deprecated)` JSON tab on a brand-new doc section
  reads odd. Consider dropping the JSON tab here and pointing
  at the JSON deprecation notice once at the top of the page,
  if that's the project direction.

## What I learned

Documenting automatic identification headers is small but
high-leverage. Anyone building a fork or a downstream product
needs to know what's being shipped on their behalf, both for
correct attribution and for any privacy / compliance review.
Three headers is small; a multi-page doc PR is large only in
that it surfaces something that was previously implicit.
