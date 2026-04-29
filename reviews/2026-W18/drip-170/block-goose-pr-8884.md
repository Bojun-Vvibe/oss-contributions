# Review: block/goose#8884 — blog: goose with peekaboo

- **PR:** https://github.com/block/goose/pull/8884
- **Head SHA:** `4a33e2e61e28ed20a568fd0abb5d830494d903b6`
- **Author:** acekyd
- **Diff size:** +133 / -0 across 2 files (one new markdown post + one binary PNG)
- **Verdict:** `merge-after-nits`

## What it does

Adds a single new blog post under `documentation/blog/2026-04-29-computer-controller-peekaboo/index.md` (133 lines) plus a hero image at `documentation/static/img/blog/computer-controller-peekaboo.png`. The post announces v1.26's Computer Controller extension rebuild on top of [Peekaboo](https://github.com/steipete/Peekaboo), a macOS CLI for screen capture + accessibility-driven GUI automation. Walks through the see → click → type loop, four use-case categories, the command toolkit (`see`, `click`, `type`, `scroll`, `drag`, `menu`, `dialog`), tips, and the macOS-only limitation.

## Specific citations

- `documentation/blog/2026-04-29-computer-controller-peekaboo/index.md:1-9` — frontmatter (title/description/authors/image). Author handle `adewale` should resolve in `documentation/authors.yml`; needs verification.
- `:62-66` — describes a four-line `goose configure` flow:
  ```
  goose configure
  # → Toggle Extensions → enable computercontroller
  ```
  Confirm the extension id is exactly `computercontroller` (one word, no hyphen) and matches the registered name in `crates/goose-server/src/extensions/registry.rs` or wherever extensions are listed.
- `:97-104` — "Current limitations" section says "On Windows and Linux, the Computer Controller falls back to shell-based automation (PowerShell scripts on Windows, xdotool/wmctrl on Linux)." Verify this matches the actual fallback in the rebuilt extension; if the rebuild dropped the cross-platform fallback in favor of macOS-only, this paragraph is misleading.
- `:107` — the "Take a screenshot of my current screen and describe what you see" example assumes the extension exposes `see` to the model under that exact natural-language interface. Worth a one-line confirmation it works as written.
- `:128-132` — OG meta `og:url` hard-codes `2026/04/29/computer-controller-peekaboo` — confirm Docusaurus's URL slug matches (the date prefix in the directory name is `2026-04-29`, so this should resolve correctly under Docusaurus's default blog routing).
- The PNG at `documentation/static/img/blog/computer-controller-peekaboo.png` is added as a binary blob — no diff content shown. Confirm: file size reasonable (target < 500KB for a hero image), correct dimensions for the blog OG card (1200×630 is the standard), and no embedded EXIF/screenshot metadata that leaks paths or hostnames.

## Nits to address before merge

1. **Verify the v1.26 release tag exists** at the publication date (2026-04-29). The post says "With v1.26, goose broke out of the terminal." If v1.26 hasn't shipped yet, either gate the post on the release or soften to "the upcoming Computer Controller rebuild".
2. **Add a Peekaboo install/dependency note.** The post says "If you're on macOS, Peekaboo will handle the rest" at `:73`, implying the binary is bundled or auto-installed. Be explicit: is Peekaboo bundled in the goose distribution, installed on first use, or does the user need `brew install steipete/peekaboo`? This is the #1 question a reader will have.
3. **Accessibility permissions caveat is missing.** Peekaboo + macOS GUI automation requires Accessibility permission grants in System Settings → Privacy & Security → Accessibility for the goose binary. First-time users will hit a stuck screenshot and not know why. One sentence: "On first use, macOS will prompt you to grant Accessibility access to goose; this is required for Peekaboo to read UI element labels."
4. **Cross-platform paragraph at `:97-99` is squishy.** Either drop "On Windows and Linux, the Computer Controller falls back to shell-based automation" if it's no longer true post-rebuild, or qualify it as "limited shell-based automation only — no see/click/type loop". Avoid implying parity that doesn't exist.
5. **The post links to https://github.com/steipete/Peekaboo** at `:9` and `:69` — confirm this is the canonical upstream URL and not a temporary fork. Broken external links in long-lived blog content are a chronic site-rot source.
6. **OG card dimensions and alt text.** `:6` `image: /img/blog/computer-controller-peekaboo.png` is good; the `<head>` block at `:121-132` repeats `og:image` implicitly via the blog plugin. Confirm the rendered OG card on Twitter/Slack doesn't crop the annotated screenshot's UI labels.
7. **The "Standard UI elements work best" limitation at `:101`** is correct but could mention specific known-bad cases (Electron apps with custom-rendered widgets, some Figma surfaces) so readers calibrate expectations.

## Rationale

Pure documentation addition, no production code path touched. The post is well-structured (see/click/type loop is the right framing, four use-case categories are concrete, limitations section is honest about macOS-only and standard-UI scope). The gaps are factual-verification and discoverability (Peekaboo install path, Accessibility permission grant, v1.26 release-gate). All fixable in-PR with text edits before merge.