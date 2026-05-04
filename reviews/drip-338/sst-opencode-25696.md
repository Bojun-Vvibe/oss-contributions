# Review: sst/opencode #25696

- **Title:** docs(es): update providers documentation
- **Head SHA:** `2015f070e578ccfcb37a24b80de9835ecba190b8`
- **Scope:** +241 / -34, single file `packages/web/src/content/docs/es/providers.mdx`
- **Drip:** drip-338

## What changed

Refresh of the Spanish providers docs to bring it back into sync with the
English source. Three substantive areas:

1. Connect-flow corrections and section dividers (`***`) inserted across the
   Bedrock auth section.
2. Anthropic auth section rewritten to drop the Pro/Max OAuth flow
   (now plugin-only since 1.3.0) and direct readers to "Manually enter API
   Key" instead.
3. Cloudflare AI Gateway flow rewritten from env-var-first to the in-CLI
   `/connect` prompt-driven flow with separate Account/Gateway/Token steps.

## Specific observations

- `providers.mdx` line ~60: changing `seleccione opencode` →
  `seleccione \`OpenCode Zen\`` matches the actual TUI label and reduces
  ambiguity vs. the generic "opencode" provider entry. Good fix.
- `providers.mdx` Bedrock section (lines ~170–260): the `***` dividers are
  inserted only between subsection H4 headings; rendering should still be
  clean in Starlight. Verify the divider doesn't introduce an unwanted gap
  before the "Métodos de autenticación" list — worth a local preview.
- `providers.mdx` Anthropic block (lines ~302–335): the rewritten `:::info`
  block ends with `:::` glued to the last bullet (`- Gitlab Duo\n  :::`).
  That's likely to break the admonition close on Starlight — the `:::` must
  be on its own line. Nit before merge.
- `providers.mdx` Cloudflare flow (lines ~525–570): new step ordering is
  consistent with the EN docs reorganization and removes the pre-export
  env-var requirement. The "Anota tu Account ID y Gateway ID" inline note
  is helpful.
- Translation register stays in formal `usted` form throughout — consistent
  with the rest of the ES docs.

## Risks

- The `:::info` close issue noted above is the only render risk.
- Pure docs change, no code paths touched, no tests needed.

## Verdict

**merge-after-nits** — fix the dangling `:::` close in the Anthropic info
block (move `:::` to its own line), then ship.
