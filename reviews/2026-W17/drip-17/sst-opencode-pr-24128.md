# sst/opencode PR #24128 — feat: optimize media attachments on paste in TUI

- **Author:** alohaninja
- **Head SHA:** 0ac3d297d45bd1beac09453b0df97c9e9c640c35
- **Files:** `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` (+27 / −13),
  `packages/opencode/src/cli/cmd/tui/util/media-optimize.ts` (+515),
  `packages/opencode/src/server/routes/instance/session.ts` (+2),
  `packages/opencode/src/session/prompt.ts` (+14),
  `packages/opencode/src/util/media.ts` (+16 / −2),
  `packages/opencode/test/cli/cmd/tui/media-optimize.test.ts` (+243),
  `packages/opencode/test/plugin/message-parts-before.test.ts` (+271),
  `packages/opencode/test/util/media.test.ts` (+129),
  `packages/plugin/src/index.ts` (+23)
- **Verdict:** `needs-discussion`

## What the diff does

Introduces a TUI-side media optimization layer that runs **before**
parts are sent to the server, and a plugin extension point so users
can register custom optimizers per modality.

Key pieces:
- `media-optimize.ts` (new, 515 lines): lazy-detects `sips`,
  `magick`/`convert`, `ffmpeg`/`ffprobe`, and a `shift-ai` binary,
  then exposes `optimizeParts(parts)` and `isAttachable(mime)`.
- `prompt/index.tsx` lines 813–840: wraps the assembled parts list
  with `await optimizeParts(assembledParts)` before calling
  `sdk.client.session.prompt`.
- `prompt/index.tsx` lines 901–915: extends the on-paste handler so
  `[Audio N]` / `[Video N]` virtual labels are produced alongside
  `[PDF N]` / `[Image N]`, with the matching `count` derived from
  same-mime parts already in the prompt.
- `prompt/index.tsx` line 1194: replaces the inline
  `mime.startsWith("image/") || mime === "application/pdf"` check
  with the new `isAttachable(mime)` helper.

## Review notes

- **Scope is much larger than the title.** "optimize media on paste"
  is sold as an ergonomic perf change, but the diff also (a) adds a
  new plugin surface (`registerOptimizer`), (b) introduces external
  binary dependencies (`ffmpeg`, `sips`, `magick`, `shift-ai`), and
  (c) silently mutates user input before send. Each of those is
  reviewable in its own right.
- **Tool detection is silent.** If `ffmpeg` isn't installed, video
  parts… do what? The shipped behavior should be documented in the
  PR body and surfaced to the user (status-line note "video sent
  unoptimized: ffmpeg not found"). Right now it looks like the
  fallback is "send as-is", which is fine, but users should know.
- **`shift-ai` binary detection** (line ~36 of `media-optimize.ts`)
  introduces a dependency on a third-party tool by name — that
  needs justification in the PR description and probably a config
  flag, not implicit autodetection. If it isn't a shipped tool
  somewhere documented, it shouldn't be probed.
- **`optimizeParts` blocks the send.** Wrapping the call site in
  `await` (line 825) means a paste of a 50MB video stalls the
  prompt UI until ffmpeg returns. Consider streaming a "optimizing
  N attachments…" status and allowing cancel.
- **Plugin contract.** `registerOptimizer` in
  `packages/plugin/src/index.ts` (+23) is a new public API. Worth
  pulling out into its own PR once the in-tree behavior settles —
  shipping a plugin API in the same change as its only consumer
  freezes the surface prematurely.
- **Test coverage is excellent for size.** 643 lines of new tests
  across `media-optimize`, `media`, and `message-parts-before` is
  the right ratio. Worth keeping the bar this high if the PR is
  split.

Recommend splitting into: (1) audio/video paste labels + isAttachable
refactor (mergeable on its own), (2) optimizer infrastructure +
plugin API behind a flag, (3) shift-ai integration as a separate
plugin out of tree.

## What I learned

Optimizing user content "on the way out" is one of those features
that looks self-contained but is really a policy choice — every
silent transformation is a potential surprise (loss, latency,
"why is my audio mono now"). The right shape is usually opt-in per
modality with an explicit "I optimized this" annotation in the
sent message, not implicit transcoding.
