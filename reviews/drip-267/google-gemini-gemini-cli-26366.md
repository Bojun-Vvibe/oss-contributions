# google-gemini/gemini-cli #26366 — fix(sea): run forked helper scripts directly instead of spawning a new session

- **Head SHA:** `42b74eea86cf5bbbc1178d4daba7697fa0ddaea4`
- **Files:** `integration-tests/ui-hang-repro.test.ts` (+58 NEW), `packages/cli/src/ui/contexts/KeypressContext.tsx` (+106 / -3), `sea/sea-launch.cjs` (+45), `sea/sea-launch.fork.integration.test.cjs` (+150 NEW), `sea/sea-launch.test.js` (+53 / -5)
- **Verdict:** `request-changes`

## Rationale

The PR's stated subject is the SEA launcher fix, but the diff actually combines two unrelated changes: (a) `sea/sea-launch.cjs` adjustments so forked helper scripts run directly rather than spawning a new SEA session, and (b) a substantial 106-line rewrite of `KeypressContext.tsx` that introduces `PASTE_BATCH_THRESHOLD = 32` plus an in-band ESC-sequence state machine (`inEscape`, `escapeIntro`) to batch large clipboard pastes into a single `paste` event. The accompanying `integration-tests/ui-hang-repro.test.ts` file confirms the second change is real product behavior, not incidental.

These should be separate PRs. The SEA launcher change is mechanical and reviewable in isolation; the keypress batching change is a hot-path rewrite that materially changes how arrow keys, function keys, OSC sequences, and bracketed paste interleave. The risk surface for the keypress change includes: (1) terminals that emit ESC sequences split across data chunks where `inEscape` persistence is correct in principle but easy to get subtly wrong; (2) the `parserEmitted` tracking flag relies on the parser invoking `trackedHandler` synchronously — please confirm `emitKeys` never defers; (3) the threshold of 32 is asserted as "small runs (typical fast typing) must still produce individual keypress events" but typing is rarely 32 chars in a single read — the danger is paste-vs-fast-input on Windows ConPTY where small chunks coalesce.

Ask the author to split: land the SEA fix as PR A (clearly mergeable on its own with `sea/sea-launch.fork.integration.test.cjs`), and reopen the keypress batching as PR B with explicit benchmarks and Windows ConPTY coverage. Combined as-is, this is too large a behavior change to merge under a misleading title.

