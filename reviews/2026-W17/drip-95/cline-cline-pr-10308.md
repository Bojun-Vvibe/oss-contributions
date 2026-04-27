# cline/cline #10308 — docs: add QuickSilver Pro provider configuration page

- Author: raullenchai
- Head SHA: `1a0131574257d01e6c022e09cb461191482bf0cd`
- +39 / −0: new `docs/provider-config/quicksilverpro.mdx`
- PR link: <https://github.com/cline/cline/pull/10308>

## Specifics

- New file at `docs/provider-config/quicksilverpro.mdx:1-39` follows the docs-site frontmatter convention with `title` and `description` keys at `:7-10`. Title `"QuickSilver Pro"` and description scoped to "OpenAI-compatible access to DeepSeek V3, R1, and Qwen3.5-35B-A3B" — concise and accurate.
- Three sections in the standard order: "Getting an API Key" (`:17-21`), "Supported Models" (`:23-29`), "Configuration in Cline" (`:31-37`). Matches the structure used by neighboring provider pages in `docs/provider-config/`.
- The configuration steps at `:33-37` walk through the OpenAI-Compatible adapter path: select "OpenAI Compatible" provider, set `Base URL` to `https://api.quicksilverpro.io/v1`, paste the key, and enter one of three model IDs. This is the standard OpenAI-compat onboarding for any third-party inference provider — no custom adapter code is being claimed; the docs are pure user-facing.
- Notes section at `:39-45` flags two things worth knowing: (1) the response includes non-standard `usage.cost`/`usage.currency` fields alongside OpenAI's `usage` object — useful for users wanting per-request cost telemetry, and (2) the political context that "DeepSeek removed R1 from their direct API in December 2025" giving QuickSilver Pro a unique R1 access angle. Both notes add real value beyond the generic OpenAI-compat boilerplate.
- The model list at `:25-27` includes context-window sizes (`131K context` for v3/r1, `262K context, 3B active MoE` for qwen3.5-35b) — concrete enough that users can size requests without re-checking the provider site.

## Concerns

- The third model ID `qwen3.5-35b` at `:27,37` is unusual — Qwen's official naming uses `Qwen3-`/`Qwen2.5-` prefixes (no `Qwen3.5`), and a 35B-A3B (35B total, 3B active) MoE model isn't in Qwen's published lineup as of this PR's timeframe. The model could be a QuickSilver-rebrand of an upstream Qwen variant, but the docs should either cite the upstream model card or acknowledge that the identifier is QuickSilver-specific. As written, a user could reasonably search for "qwen3.5-35b" elsewhere and find nothing.
- All three of QuickSilver Pro's URLs (`quicksilverpro.io`, `quicksilverpro.io/dashboard`, `api.quicksilverpro.io/v1`) appear to be the legitimate public endpoints based on the doc's surface, but the cline reviewer should sanity-check the domain hasn't been recently registered as a typosquat — provider docs PRs from non-affiliated contributors are a common vector for fake-provider seeding. Author `raullenchai` has no disclosed affiliation in the diff. (The PR description and author profile would resolve this, not visible from the diff alone.)
- The "Cost visibility" note at `:42-44` claims `usage.cost` and `usage.currency` are returned alongside standard OpenAI usage fields. cline doesn't itself consume non-standard usage fields today, so this is informational-only and won't show up in the cline UI's cost tracker — worth a one-line caveat that "cline doesn't currently surface these fields in its cost UI" so users don't expect automatic cost tracking just by configuring the provider.
- "R1 availability" note at `:44-45` makes a factual claim about DeepSeek's API ("removed R1 from their direct API in December 2025") that's marketing-adjacent — true in the sense that DeepSeek did pull R1 access for direct users at one point, but framing it as "third-party providers like QuickSilver Pro are the only way to continue using it" is partially promotional. Could soften to "...are one way to continue accessing it" without losing information.

## Verdict

`needs-discussion` — the docs are well-structured and follow the convention of `docs/provider-config/`, but two questions need a maintainer call before merging: (1) verify the `qwen3.5-35b` model identifier corresponds to a real upstream model and isn't a re-label that needs a footnote, and (2) confirm the QuickSilver Pro provider relationship — if cline accepts third-party provider docs PRs from anyone who registers a domain, document that policy; if not, validate the author's affiliation. The "R1 availability" promotional framing is a smaller wording nit. The actual configuration mechanics (OpenAI-compat + base URL + model ID) are correct and don't need code-side changes.
