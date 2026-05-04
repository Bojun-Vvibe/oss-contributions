# Review: QwenLM/qwen-code #3671

- **Title:** docs(design): banner customization design (#3005)
- **Head SHA:** `43c41314d027369d70fd6312460634a167abd8b1`
- **Scope:** +1146 / -0 across 2 files (EN + zh-CN design docs)
- **Drip:** drip-338

## What changed

Adds a design document under
`docs/design/customize-banner-area/` (English plus Chinese translation)
proposing three new `ui.*` settings — `hideBanner`, `customBannerTitle`,
`customAsciiArt` — for Qwen Code's startup banner. The doc enumerates
which banner regions are replaceable (logo column, brand title) vs.
locked (version suffix, auth/model line, working directory line) and
spells out the three accepted shapes for `customAsciiArt`: inline string,
`{ path }`, and width-tiered `{ small, large }`. References upstream
issue #3005 and explicitly out-of-scopes runtime text-to-art rendering.

## Specific observations

- `docs/design/customize-banner-area/customize-banner-area.md:1-60` — the
  framing "brand chrome is replaceable; operational data is not" is
  stated cleanly up front and then drives the lock/replace matrix
  consistently. Good design discipline.
- `docs/design/customize-banner-area/customize-banner-area.md:90-140` —
  the customization rules table is the load-bearing section. It maps
  every banner region to a customization category and a rationale.
  Locking the version suffix on the grounds of "support tractability"
  is the right call; many CLIs eventually regret making the version
  hideable.
- `docs/design/customize-banner-area/customize-banner-area.md:150-180` —
  explicit "what is NOT offered" list is unusually disciplined for a
  design doc and should reduce future feature creep PRs that try to
  hide the auth line or path line piecewise. Worth keeping verbatim.
- `docs/design/customize-banner-area/customize-banner-area.md:200-260` —
  path resolution rules ("workspace settings → `.qwen/`, user settings
  → `~/.qwen/`") are precise and answer the obvious "relative to what?"
  question. Good. One nit: the "file is read once at startup … editing
  the file mid-session does not re-render" rule should also note the
  failure mode if the file is moved or deleted between reads — does
  the next startup fall back to default art silently, or surface an
  error?
- `docs/design/customize-banner-area/customize-banner-area.zh-CN.md` —
  Chinese translation tracks the EN doc structurally; quick-spot check
  that the lock/replace matrix is preserved with the same column
  semantics. Translator should confirm "screen-reader gate" wording
  matches the existing zh-CN docs convention for accessibility terms.
- File path uses `customize-banner-area/` plural-less convention,
  consistent with sibling design docs in the repo's `docs/design/`
  tree (worth a quick `ls` to confirm the convention).

## Risks

- Pure docs / design-only PR. No code paths affected. Risk is bounded
  to the design's downstream implementation PRs adopting (or quietly
  drifting from) these rules.
- The doc is large (1146 lines across two languages). Reviewers should
  be aware that approving it locks in the lock/replace matrix as a
  contract.

## Verdict

**merge-after-nits** — design is disciplined and well-justified;
clarify the "missing custom art file" failure mode, and have a
zh-CN reviewer confirm the translation register before merging.
