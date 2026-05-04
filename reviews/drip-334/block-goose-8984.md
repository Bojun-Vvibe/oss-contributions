# block/goose#8984 — [blog] new post on findings with MiniMax and OfficeQA benchmark

- **Head SHA:** `877c78a273aeec34f92405832978d67dbefc8905`
- **Author:** internal contributor (mic)
- **Size:** +25 / −0, 1 file (`documentation/blog/2026-05-04-officeqa-minimax-m25/index.md`)

## Summary

Single new blog post under `documentation/blog/`. Highlights a third-party (Sentient AGI) result running goose with MiniMax M2.5 against the Databricks OfficeQA benchmark. Compares cost ($1.74/run vs $56.53 for Opus 4.5) and accuracy (~70% vs ~80% closed-source).

## Specific citations

- `documentation/blog/2026-05-04-officeqa-minimax-m25/index.md:1-5` — front matter: `title`, `description`, `authors: [mic]`. Title "Approaching frontier performance for 1/30th the cost" matches the body's "30× cheaper" claim.
- `index.md:7` — primary source link to `https://x.com/SentientAGI/status/2046967422004154739`. Single external link is the load-bearing citation; the rest of the post is quoted from it.
- `index.md:11-13` — quoted bullet about goose harness vs alternatives ("8× cheaper than the next option", "20x more token efficient than OpenHands", "more than 40x cheaper than OpenCode"). Comparative claims attributed to the linked article.
- `index.md:18-23` — `<head>` block with `og:title`, `og:type`, `og:url`, `og:description` for social sharing. URL is `https://goose-docs.ai/blog/2026/05/04/officeqa-minimax-m25` — should match the actual published path; worth a sanity check that the docs site routes here.

## Verdict: `merge-after-nits`

Marketing/blog content, low risk. Two suggestions:

1. **Single-source citation.** The entire post leans on one X (Twitter) post from `@SentientAGI`. Twitter posts are mutable/deletable; consider mirroring the screenshot/numbers into a static asset under `documentation/blog/2026-05-04-officeqa-minimax-m25/` so the post survives if the tweet is deleted. Even better: link to the underlying Databricks OfficeQA blog (`databricks.com/blog/officeqa`) for the benchmark methodology in addition to the tweet.
2. **Comparative claims about other tools** ("20x more token efficient than OpenHands", "40x cheaper than OpenCode") are reproduced from a third party. These are strong public claims about competitor products; consider either: (a) attributing more explicitly inline ("Per @SentientAGI's findings:"), or (b) noting the methodology limitations (single benchmark, single run config) so readers calibrate. Reputational risk if the numbers don't reproduce and goose appears to have endorsed them.
3. **`og:url` host** (`goose-docs.ai`) — verify this matches the actual deployed docs URL. If the canonical host is `block.github.io/goose` or similar, the OG metadata will produce wrong shares.

Content-wise, fine to merge once #1 (mirror the data locally) is addressed.
