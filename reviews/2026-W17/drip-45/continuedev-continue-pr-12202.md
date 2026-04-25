# continuedev/continue #12202 — update broken documentation references

- **PR:** https://github.com/continuedev/continue/pull/12202
- **Head SHA:** `b3e1249a40796d77db8ad9d83ef42374833d9157`
- **Files changed:** 4 — `docs/guides/netlify-mcp-continuous-deployment.mdx`, `docs/guides/running-continue-without-internet.mdx`, `docs/reference.mdx`, `docs/reference/json-reference.mdx`

## Summary

Pure docs link-rot fixes. Updates four URLs that previously pointed at moved/renamed pages on `docs.continue.dev`, and reformats one inline JSON example for readability.

## Line-level call-outs

- `docs/guides/netlify-mcp-continuous-deployment.mdx:727` — replaces `https://docs.continue.dev/guides` (apparently a 404 or generic landing) with `https://docs.continue.dev/guides/configuring-models-rules-tools`. Confirm the new URL is actually a guides index and not just one specific guide — the surrounding context is "Continue Performance Guides" (plural), and the linked page is "configuring-models-rules-tools" (singular topic), so the link text and target may now disagree. Suggest either renaming the link text to "Continue: Configuring Models, Rules, Tools" or pointing at a true guides index.
- `docs/guides/running-continue-without-internet.mdx:7` — `reference/telemetry` → `customize/telemetry`. Sensible if the docs site moved telemetry under `/customize/`. Verify by hitting the URL.
- `docs/guides/running-continue-without-internet.mdx:8` — `reference/model-providers/ollama` → `customize/model-providers/top-level/ollama`. Same — verify; the `top-level/` segment looks like an internal information-architecture artifact that might surprise users. If the docs site has a redirect, leaving the shorter path may actually be more durable.
- `docs/reference.mdx:301` — example `startUrl` updated from `https://docs.continue.dev/intro` to `https://docs.continue.dev/index`. The new URL `.../index` is unusual for documentation sites — typically the index page has no path or is `/` or `/getting-started/...`. Worth checking whether `/index` is actually canonical or whether this is masking a redirect; if the site follows convention, `/getting-started/overview` (which the JSON-reference example uses two files later) would be more consistent.
- `docs/reference/json-reference.mdx:282-289` — two improvements bundled here: (a) reformats the single-line JSON sample into pretty-printed multi-line form (clear win for readability), and (b) updates `startUrl` to `https://docs.continue.dev/getting-started/overview` and `faviconUrl` to `https://www.continue.dev/favicon.png`. **Inconsistency:** the `reference.mdx` example just two files away uses `/index` while this one uses `/getting-started/overview` — pick one canonical example URL and use it in both places, otherwise readers comparing the two examples will be confused about which is "right".
- General: link-rot fixes ideally come with a one-line note on *why* (e.g. "site IA reorganized in #XXXX"). Without it, the next person doing a docs link audit can't tell whether these are the *current* canonical paths or yet another snapshot.

## Verdict

**merge-after-nits**

## Rationale

Pure-docs link fixes are almost always low-risk and welcome. The two things to address before merge: (1) the canonical "intro" example URL is inconsistent between `reference.mdx` (`/index`) and `json-reference.mdx` (`/getting-started/overview`) — pick one, (2) the link text in the Netlify guide may no longer match its target after the rewrite. Otherwise this is exactly the kind of small-bore maintenance the docs benefit from.
