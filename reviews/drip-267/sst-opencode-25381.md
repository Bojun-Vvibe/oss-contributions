# sst/opencode #25381 — docs: add Manifest provider section

- **Head SHA:** `a4db79d152e9c198128bea568924a20a1b8c7f9e`
- **Files:** `packages/web/src/content/docs/providers.mdx` (+44 / -0)
- **Verdict:** `merge-after-nits`

## Rationale

Pure docs addition placing a new "Manifest" provider entry into `providers.mdx` between OpenCode Zen and LLM Gateway (line ~1642 onward). Content follows the established pattern in the file: marketing one-liner, `/connect` flow, API-key entry block, `/models` step, and a `:::note[Self-hosted]` callout with a JSON config snippet pointing to `http://localhost:2099/v1`. No code paths touched, no schema changes — risk is essentially zero.

Nits: (1) the description claims "16+ providers" and "smart routing" — these are unverifiable marketing claims; consider asking the submitter to soften to "multiple providers" or cite the source. (2) The API-key prefix `mnfst_` is asserted but not used anywhere by opencode validation — fine for docs but worth confirming with the Manifest team that the prefix is stable. (3) The localhost port `2099` should ideally link to Manifest's own self-host docs in case that default changes upstream.

Approve with the marketing-claim softening; the structural fit and formatting are correct and consistent with neighboring provider entries.

