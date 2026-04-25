---
pr: 4717
repo: browser-use/browser-use
sha: 0112e6f2d91cce8bb0f14676e3ea61acf788650f
verdict: request-changes
date: 2026-04-26
---

# browser-use/browser-use#4717 — fix: video recording speed by implementing frame duplication

- **URL**: https://github.com/browser-use/browser-use/pull/4717
- **Author**: Myc911
- **Files**: `browser_use/browser/video_recorder.py`,
  `browser_use/browser/watchdogs/recording_watchdog.py` (+28/-8)

## Summary

CDP `Page.screencastFrame` events arrive at irregular intervals
(driven by repaints, not a fixed clock). The video writer was
appending one frame per event regardless, so a session with
sparse activity rendered a video that played back at a totally
wrong speed (much faster than real time, since N frames spread
over T real seconds were being played at the writer's nominal
framerate). The PR fixes this by passing the CDP frame's
`metadata.timestamp` through to the recorder and filling the gap
between frames with copies of the previous frame to keep
playback in real time.

## Reviewable points

- `browser_use/browser/video_recorder.py:55` — adds two new
  instance fields:
  - `self.last_frame_time: float | None`
  - `self._last_img_array: np.ndarray | None`

- `video_recorder.py:64` — `add_frame` now takes
  `timestamp: float | None = None`. Backwards-compatible default.

- `video_recorder.py:128-134` — the fill logic:

  ```python
  if timestamp is not None and self.last_frame_time is not None:
      dt = timestamp - self.last_frame_time
      if dt > 1.0 / self.framerate:
          num_fill_frames = int(dt * self.framerate) - 1
          if num_fill_frames > 0 and self._last_img_array is not None:
              for _ in range(num_fill_frames):
                  self._writer.append_data(self._last_img_array)
  ```

  This is the meat. Several real concerns:

  1. **No upper bound on `num_fill_frames`**. If a tab is
     backgrounded for 5 minutes (no screencast frames) and then
     a frame arrives, `dt` is 300s; at 30fps that's 8999 fill
     frames inserted in a tight Python loop. Each
     `_writer.append_data` re-encodes a frame, so this can
     pause the asyncio event loop for *seconds* on a long idle
     gap. **Recommendation**: cap `num_fill_frames` (e.g.
     `min(num_fill_frames, self.framerate * MAX_GAP_SECONDS)`)
     and log a warning when the cap fires. Alternative: use a
     ffmpeg variable-framerate output with explicit pts so you
     don't have to materialize duplicate frames at all.

  2. **CDP timestamp units are not asserted.** CDP
     `screencastFrame.metadata.timestamp` is "monotonic time
     when the frame was captured" — the units in CDP land are
     **seconds** (a fractional `MonotonicTime` since CDP
     started), not ms. The arithmetic here treats it as
     seconds, which matches the CDP spec, but there's no
     assertion or comment. If a watchdog ever passes
     `time.time()` instead (epoch seconds), the math still
     works because both endpoints use the same scale — but if
     someone passes ms (some CDP wrappers normalize to ms),
     `dt` becomes 1000× larger and `num_fill_frames` explodes.
     Worth a one-line comment + a sanity assertion like
     `if dt > 3600: log.warning(...); return` to fail
     gracefully on units bugs.

  3. **`int(dt * framerate) - 1`**: at exactly `dt == 1/fps`,
     this yields `0` fill frames. Good — that's the no-gap
     case. At `dt == 2/fps`, it yields `1` fill frame, plus
     the actual frame on line 137 = 2 frames covering 2/fps
     of time. Correct.

- `video_recorder.py:137-139` — stores the new frame as
  `_last_img_array` for the *next* gap fill. Note this stores
  the post-resize/pad numpy array, not the original PNG bytes
  — that's a memory consideration: holding a full RGBA frame
  (~10MB at 1920×1080) in memory for the lifetime of the
  recorder. Acceptable.

- `browser_use/browser/watchdogs/recording_watchdog.py:198` —
  the wiring:
  ```python
  timestamp = event.get('metadata', {}).get('timestamp')
  self._recorder.add_frame(event['data'], timestamp=timestamp)
  ```
  Defensive `.get` chain handles the case where CDP doesn't
  send metadata. Fine.

- The header refactor at `video_recorder.py:1-23` —
  `from __future__ import annotations` plus moving
  `imageio.core.format.Format` to a `TYPE_CHECKING` block — is
  good hygiene and removes an import-time dependency on
  imageio internals. Unrelated to the bug fix but a welcome
  cleanup.

## Risks

- The unbounded fill loop is the headline risk and reproducible
  with a single backgrounded tab. This is a medium-priority
  blocker — not catastrophic (the recorder eventually catches
  up) but it can stall the entire async loop for seconds and
  starve other watchdogs.
- No tests for the fill behavior. A small unit test that calls
  `add_frame` twice with a 1s `dt` at 10fps and asserts the
  writer received 10 `append_data` calls would lock the
  contract.
- See PR #4697 in this repo (already reviewed in drip-50, same
  area) — there's overlap with another timestamp-handling fix.
  Worth coordinating to avoid merge conflicts.

## Verdict

`request-changes`. Cap the fill count and add at least one
unit test. The core idea (use CDP timestamps to preserve real
playback time) is correct, but the unbounded loop is a real
foot-gun that will surface the first time someone records a
session with idle periods.

## What I learned

CDP screencast frames are paint-driven, not clock-driven, so any
naive 1-frame-per-event recording will produce a video at the
wrong speed when activity is sparse. The fix needs *some* form
of time alignment — duplicate-fill is the simplest, but it
requires an upper bound to avoid pathological cases. The cleaner
long-term answer is a variable-framerate writer (ffmpeg `-vsync
vfr` with explicit pts) so the player handles timing rather than
the encoder materializing duplicates.
