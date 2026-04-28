# QwenLM/qwen-code #3519 — feat(cli): paste base64 / data URL images, drag image files, with `[Image #N]` placeholders

**Verdict:** `merge-after-nits`

- Repo: `QwenLM/qwen-code`
- PR: https://github.com/QwenLM/qwen-code/pull/3519
- Head SHA: `02e181eb`
- Closes: #3518
- Diff: +704 / -32 across 4 files: `packages/cli/src/ui/components/InputPrompt.tsx`, `packages/cli/src/ui/components/InputPrompt.test.tsx`, `packages/cli/src/ui/utils/clipboardUtils.ts`, `packages/cli/src/ui/utils/clipboardUtils.test.ts`

## What it does

Adds three new ways to attach an image to an input prompt and unifies all
four paths (existing Cmd+V binary clipboard plus the three new ones) under
a single `[Image #N]` placeholder UX. New paths:

1. **Data URL paste** — paste `data:image/<type>;base64,<payload>` text
   (browser devtools "copy as data URL", chat-message embeds, etc.).
2. **Raw base64 paste** — accepted only when the decoded prefix matches a
   known image magic byte sequence (PNG / JPEG / GIF / WebP / BMP /
   TIFF). JWTs, hashes, and other base64-shaped text are rejected so the
   normal large-paste flow isn't hijacked.
3. **Drag-and-drop image file** — handles both bracketed-paste-wrapped
   drops *and* terminals (notably macOS Terminal.app) that synthesize a
   drop as a rapid burst of individual keystrokes, via a `< 10ms`
   keystroke-gap heuristic plus a 150ms debounced scan of
   `buffer.text`'s trailing token.

All four paths converge on an inline `[Image #N]` placeholder + matching
chip in the attachment row, and at submit time `[Image #N]` is replaced
with `@<relative path>` so the model sees images exactly where typed.

## Design analysis

The shape is right for the user-facing UX problem (multi-image,
positional, with a stable placeholder), and the unification under a
single counter is the load-bearing structural choice that keeps this
from becoming four parallel code paths.

### Placeholder allocation (`InputPrompt.tsx:147-152`)

```ts
const imagePlaceholderCounter = useRef(0);
const allocateImagePlaceholder = useCallback((): string => {
  imagePlaceholderCounter.current += 1;
  return `[Image #${imagePlaceholderCounter.current}]`;
}, []);
```

Monotonic counter per input, reset in `handleSubmitAndClear` at `:366`
(`imagePlaceholderCounter.current = 0`). Right shape — placeholders are
stable while the user is editing, and the user's mental model ("paste
two images, refer to them as #1 and #2") matches the implementation
exactly.

### Submit-time substitution (`InputPrompt.tsx:329-355`)

The two-bucket approach (`refByPlaceholder` for placeholder-having
attachments, `prependRefs` for legacy paths) is the right backwards-compat
shape:

```ts
if (refByPlaceholder.size > 0) {
  finalValue = finalValue.replace(IMAGE_PLACEHOLDER_RE, (match) =>
    refByPlaceholder.has(match) ? refByPlaceholder.get(match)! : match,
  );
}
if (prependRefs.length > 0) {
  finalValue = `${prependRefs.join(' ')}\n\n${finalValue.trim()}`;
}
```

Critically: if the user *deletes* a placeholder before submitting (e.g.
pastes `[Image #1]`, then backspaces it out), the substitution no-ops and
the image is silently dropped. That's the right behavior — the
attachment chip should also disappear, and from the diff context I
*think* it does, but a test specifically for "delete placeholder, image
chip disappears, submit, no `@path` in finalValue" would lock that in.

### Drag-burst detection (`InputPrompt.tsx:155-167`)

```ts
const DRAG_BURST_MAX_INTERVAL_MS = 10;
const DRAG_CHECK_DEBOUNCE_MS = 150;
const DRAG_MIN_BURST_CHARS = 4;
```

Three magic numbers, all reasonable. The `< 10ms` between consecutive
chars is below human typing latency (a fast typist hits ~80ms inter-key)
so false positives during ordinary typing are nearly impossible. The
150ms debounce is enough for the entire path to land before the scan
runs. The minimum-4-chars threshold prevents single-character bursts
from triggering scans.

### Tail-scan for dragged path (`InputPrompt.tsx:438-457`, `scanBufferTailForDraggedImage`)

```ts
const match = text.match(/('([^']+)'|"([^"]+)"|(\S+))\s?$/);
```

Handles single-quoted, double-quoted, and bare-word paths with an
optional trailing space — that's the right cross-terminal coverage
(macOS Terminal wraps in single quotes, iTerm without bracketed paste
adds a trailing space, kitty does neither).

The validation in `clipboardUtils.detectDraggedImagePath` (per the PR
body: "validates a token is an existing local image file; supports
single/double-quoted paths and escaped spaces") is the right gate — it
prevents the scan from converting random text-ending-in-`.png`-string
into an attachment.

### Magic-byte sniffing for raw base64 (`clipboardUtils.ts`)

Per the PR body, raw base64 is only accepted when the decoded prefix
matches PNG/JPEG/GIF/WebP/BMP/TIFF magic. This is exactly the right
guard — rejecting JWT-shaped or hash-shaped base64 keeps the existing
large-paste flow from being silently hijacked.

### Test coverage (`InputPrompt.test.tsx:+158`, `clipboardUtils.test.ts`)

Three describe blocks added in `InputPrompt.test.tsx`:

- `'base64 / data URL paste'` — covers data URL paste → `[Image #1]`,
  multiple pastes → sequential allocation, non-image base64 →
  fallthrough to large-paste placeholder.
- `'drag-and-drop image paste'` — covers dragged image path → `[Image
  #1]`, non-image path → falls through to regular `@path` flow.

Right cells, particularly the fallthrough negative tests which prevent
the new code from regressing the existing paths.

## Risks

1. **Drag-burst heuristic on slow terminals or remote SSH.** A user
   typing inside a 50ms-RTT SSH session may have inter-key intervals
   that approach the 10ms threshold during burst-mode IME composition or
   xterm.js rapid-replay. Worth a real-world soak test on tmux-over-ssh.
2. **Path validation runs synchronously on every burst-end.** The
   `detectDraggedImagePath` call hits the filesystem (per PR body:
   "validates a token is an existing local image file"). If the user
   types a real-image path *as text* (e.g. they're discussing
   `/Users/me/screenshot.png` in a question), the burst-detection won't
   trigger (>10ms gaps), but if they paste it, the scan will fire and
   the path will be silently converted to an attachment. That may be
   surprising. A confirmation chip ("📎 attached image #1 — undo?") is
   over-engineering, but the substitution being silent is a UX choice
   worth documenting.
3. **Three magic numbers without a config knob.** If the heuristic
   causes false positives in the wild, debugging needs an env-var
   override. Promoting the three constants to a config-readable section
   is a small follow-up.
4. **The 150ms debounce delays attachment feedback.** A user dragging a
   file sees the path text appear, then 150ms later it's replaced with
   `[Image #1]`. That flicker may confuse first-time users. A 50ms
   debounce would feel more responsive at the cost of slightly higher
   false-trigger risk.
5. **`useEffect` cleanup at `:294-302`** clears `dragCheckTimerRef` on
   unmount — good. Worth confirming the same cleanup runs when
   `buffer.text` is reset, otherwise a stale timer could fire after a
   submit and try to scan an empty/new buffer.
6. **No e2e test exercising the full Cmd+V → submit → model-receives-`@path`
   round-trip.** All tests mock out `clipboardUtils.*`. A single
   integration test hitting the real filesystem with a fixture image
   would catch path-resolution bugs (relative-path edge cases,
   trailing-slash, symlinks) the unit tests can't.

## Suggestions

- Add the "delete placeholder → no `@path` in finalValue + chip removed"
  test described above.
- Promote `DRAG_BURST_MAX_INTERVAL_MS` / `DRAG_CHECK_DEBOUNCE_MS` /
  `DRAG_MIN_BURST_CHARS` to env-var-overridable constants for field
  debugging.
- Add one integration test using a real fixture image to validate
  end-to-end path resolution.
- Document the silent "paste-an-image-path → attachment" behavior in
  the PR body or release notes — it's a useful feature but worth being
  explicit about.
- Consider gating the drag-burst heuristic behind a settings toggle
  (`experimental.dragDropImageDetection: true` default) so users on
  problematic terminals can opt out without losing the
  bracketed-paste-wrapped drop path.

## What I learned

The right way to ship "many input methods, one mental model" is to
unify at the *output* layer (one placeholder format, one attachment
state) rather than at the input layer (one event handler that
discriminates between paste types). This PR does that — the four input
paths converge at `setAttachments` + `[Image #N]` insertion, and the
substitution at submit time is shape-blind. The hard part of the PR
(and the reason it's 700+ lines) is the burst-detection heuristic for
terminals that don't use bracketed paste, and that work is well-isolated
and well-commented. The follow-ups are all about hardening the
heuristic for adversarial timing conditions, not about reshaping the
core design.
