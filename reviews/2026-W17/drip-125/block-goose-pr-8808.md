# block/goose #8808 — blog: Built-in Local Inference blogpost

- **PR**: [block/goose#8808](https://github.com/block/goose/pull/8808)
- **Head SHA**: `fe8cc369`

## Summary

Single-file documentation contribution: adds
`documentation/blog/2026-04-24-use-goose-with-built-in-local-inference/index.md`
(95 lines) plus a binary cover image
(`documentation/static/img/blog/goose-built-in-inference.png`). The post
introduces goose's new built-in local-inference path (llama.cpp embedded
in-process, GGUF model download, no Ollama dependency), positions it
against the existing Ollama provider via a feature comparison table,
lists the curated featured-models set, and documents the realistic
trade-offs (slower first request, smaller context windows, gap in
emulated-mode tool calling).

## Specific findings

- `documentation/blog/2026-04-24-use-goose-with-built-in-local-inference/index.md:1-7`
  — frontmatter is well-formed (title, description, authors, image
  fields all present and well-typed for the Docusaurus blog plugin).
  Image path `/img/blog/goose-built-in-inference.png` resolves
  correctly to the new binary at `documentation/static/img/...`
  per the standard Docusaurus static-asset routing.
- `:9-14` — opening hook is correctly framed in user-impact terms ("no
  Ollama, no Docker, no external server") rather than implementation
  ("we integrated llama.cpp via FFI"). Good editorial choice for a
  user-facing blog post.
- `:18-28` — the "How it works" section names the actual file system
  side effect (`~/.local/share/goose/models/` as the model storage
  location). This is the kind of thing users will grep for when a
  download appears to "go missing" or they want to clean up disk —
  having it in the blog post means it's discoverable.
- `:34-42` — the comparison table is the most load-bearing piece of
  the post. Ten rows × 3 columns markdown table comparing built-in
  llama.cpp vs Ollama on Setup, Server, Model format, Model management,
  Tool calling, Vision, and Config. Honest framing — explicitly notes
  that built-in uses GGUF from HuggingFace (Ollama uses its own
  registry), and that tool calling is "Native (Gemma 4) or emulated"
  for built-in vs "Via toolshim interpreter" for Ollama. Honest
  framing is the right call because users who adopted Ollama for
  specific reasons (e.g. they already have models pulled) shouldn't
  be misled into switching.
- `:46-52` — featured-models bullet list is correct format for
  scannable reference. One nit: "As at the time of this writing"
  is a real-world maintenance hazard for a dated blog post. Either
  link to a live page that's the source of truth ("see [the model
  catalog](url)"), or commit to maintaining the list in this post
  (and document where else the list lives so they stay in sync).
- `:56-66` — the "What to expect" section is the single most valuable
  part for users evaluating the feature. Calling out concrete
  numbers ("first request takes 30–120 seconds", "4K–8K tokens of
  context vs 128K+ for Claude/GPT") sets honest expectations. The
  "It's slower" framing is the right pre-positioning for users used
  to cloud latency, and the "Apple Silicon (especially M2/M3/M4) but
  noticeably slower on CPU-only machines" note is the kind of thing
  that should be in the docs not buried in a forum thread.
- `:68-72` — privacy section ("Nothing leaves your machine. No
  telemetry, no API calls, no data sharing.") is a strong claim. If
  goose has any background telemetry (e.g. crash reporting, update
  checks) this needs an asterisk — even with local inference, the
  goose binary itself may phone home for product analytics. The
  blanket "No telemetry" claim should be precisely scoped to the
  inference path.
- `:78-90` — the tips section is well-targeted. "Don't fight emulated
  mode" with the concrete shell-command-workflow example is the kind
  of practical guidance that makes the difference between "users
  bounce off and go back to Claude" and "users adapt and stick".
- `:92-99` — `<head>` block at the bottom adds OG/Twitter card metadata
  with hardcoded URL `https://goose-docs.ai/blog/2026/04/24/use-goose-with-built-in-local-inference`.
  This URL is constructed from the slug-derived path; if the
  Docusaurus blog plugin is configured to use the date prefix in the
  URL (which it usually is by default) this is correct. Worth a
  quick verification by previewing the build.

## Nits / what's missing

- "As at the time of this writing" (`:46`) is a maintenance-hazard
  phrase. Either link to a canonical model catalog or commit the
  list to be evergreen-maintained.
- Privacy claim ("No telemetry, no API calls, no data sharing") on
  `:68-70` should be scoped to the inference path specifically, not
  blanket the whole product, unless goose desktop genuinely makes
  zero outbound calls of any kind.
- Featured-models list is bulleted-prose but the formatting is
  inconsistent: each item starts with the model name followed by an
  em-dash and a description, but the last entry ("Gemma 4 26B-A4B
  (GGUF) — a 26 billion parameter variant of Gemma 4 with native
  tool calling and vision support, for users") is truncated mid-
  sentence ("for users"). Editorial cleanup.
- The "What makes this different from Ollama" section links to
  `/blog/2025/03/14/goose-ollama` for the prior Ollama announcement.
  Good cross-reference. Should also link forward from the prior
  Ollama blog to this one (separate PR, but worth noting).
- The "Code Mode" reference at `:65` links to
  `/blog/2026/02/06/8-things-you-didnt-know-about-code-mode` — good.
- No mention of model licensing. Llama 3.2, Gemma 4, Mistral, Hermes
  all have different license terms and goose users downloading
  GGUFs from HuggingFace should at least see a one-liner pointing
  them to the model card on HuggingFace for license review,
  especially for any commercial use case.
- Cover image is a binary blob with no alt text in the `![blog
  cover]` markdown. Should be `![Cover image showing goose with
  llama.cpp local inference]` or similar for accessibility.
- "1. Open **Settings → Local Inference**" — assumes the desktop app
  flow. The intro says "available today in the desktop app" so this
  is consistent, but readers using the CLI will hit a dead-end at
  this step. Worth a one-line "(desktop only for now; CLI support
  coming)" or similar.

## Verdict

**merge-after-nits** — solid honest blog post that sets realistic
expectations for a feature whose performance is highly hardware-dependent,
the comparison-with-Ollama table is the right way to position this for
existing local-inference users, and the file-system / privacy / tips
sections are concretely actionable. Nits are all editorial: scope the
privacy claim, fix the truncated featured-model bullet, drop or
re-anchor "as at the time of this writing", add image alt text, and
note model licensing.
