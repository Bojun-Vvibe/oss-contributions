# Review: sst/opencode#25453

- **PR:** sst/opencode#25453
- **Head SHA:** `b7b8b06011ff58f7d52dc0f73df67b459d9f5dce`
- **Title:** chore(docs): rename firmware provider to frogbot
- **Author:** cgilly2fast

## Files touched (high-level)

- `packages/web/src/content/docs/<lang>/providers.mdx` — pure docs rename across many locale files (ar, bs, da, de, es, fr, it, and presumably the rest), changing every "Firmware" provider section to "FrogBot", including the `app.firmware.ai` → `app.frogbot.ai` URL in the signup link.

## Specific observations

- `packages/web/src/content/docs/ar/providers.mdx:651` (and the parallel hunk in every other locale file): heading `### Firmware` → `### FrogBot`, link text + URL changed in lock-step. Mechanical, consistent.
- `packages/web/src/content/docs/da/providers.mdx:660`: in Danish, the body text "Indtast firmware API-nøglen" became "Indtast frogbot API-nøglen". Lowercase "frogbot" inside the sentence reads slightly odd next to a brand name — most other locales preserved capitalization ("FrogBot API-Schlüssel" in de, "FrogBot API key" pattern). Minor consistency nit; could be normalized to `FrogBot` everywhere, but not blocking.
- Same minor capitalization inconsistency in `es/providers.mdx` ("clave de frogbot API" vs. the German `FrogBot API-Schlüssel`) and `fr/providers.mdx` ("Tableau de bord du micrologiciel" was kept as "micrologiciel" — French translation of "firmware" — even though the brand was renamed; the literal translation should also be updated to a transliteration of "FrogBot" or just "FrogBot dashboard").
- No code-side changes appear in the diff (no provider id renames in `packages/opencode`), so this is docs-only and shouldn't affect runtime provider resolution. Worth confirming there isn't a corresponding `models.dev` / provider id change required to actually wire `frogbot` — if the upstream provider was actually renamed, there should be a non-docs PR, and this docs PR is racing it. If the upstream is _still_ called `firmware`, this PR is wrong.
- Anti-dup risk: many translated mdx files all touch the same hunk; if the upstream provider id is unchanged, this PR could silently drift docs from the actual `/connect` menu where the user still sees "Firmware". A short note in the PR body confirming the source-of-truth rename would help.

## Verdict

**needs-discussion**

## Reasoning

The docs change is mechanically clean and broadly consistent across locales, but it documents a provider rename without a corresponding source-of-truth change in the diff. Before merging, the maintainer should confirm: (1) the underlying provider/registry id was actually renamed `firmware` → `frogbot` (and where), (2) the `/connect` UI label matches the new docs, and (3) whether the French "micrologiciel" / Danish lowercase "frogbot" body text should also be normalized. If the upstream id is unchanged, this PR will leave the docs lying to users.
