# google-gemini/gemini-cli#26506 — feat: allow queuing messages during compression

- **URL:** https://github.com/google-gemini/gemini-cli/pull/26506
- **Head SHA:** `aebbca488dff75f632df427d667fcaa54dfa3dd8`
- **Closes:** #24071
- **Files touched:** 5 (+135 / −39)
  - `packages/cli/src/ui/AppContainer.test.tsx` (+66 / −1)
  - `packages/cli/src/ui/AppContainer.tsx` (+21 / −2)
  - `packages/cli/src/ui/commands/compressCommand.test.ts` (+5 / −0)
  - `packages/cli/src/ui/commands/compressCommand.ts` (+39 / −36)
  - `packages/cli/src/ui/hooks/useMessageQueue.ts` (+4 / −0)

## Summary

Lets users queue messages while `/compress` is running. The design choice is to derive the new `isCompressing` UI state from the existing `pendingHistoryItems` array rather than threading a new boolean through global state — so `streamingState` stays `Idle`, the global `LoadingIndicator` stays hidden (the inline `MessageType.COMPRESSION` item renders its own spinner), and `useMessageQueue` gets a single new opt-in option. The compress command itself stops awaiting the long-running `tryCompressChat` call and runs it in a fire-and-forget IIFE so the input prompt re-enables immediately.

## Line-level observations

- `compressCommand.ts:16` — `action: async (context)` → `action: (context)`. Removing the `async` keyword changes the signature: the command now returns synchronously. This is the load-bearing change that releases the global `isProcessing` UI lock.
- `compressCommand.ts:39-86` — the entire `try/catch/finally` is wrapped in a `void (async () => { ... })()` IIFE. The `setPendingItem(pendingMessage)` call at `:39` runs synchronously *before* the IIFE so the spinner appears immediately; the IIFE handles the async compression and unconditionally clears the pending item in `finally` at `:84`. The `void` keyword is correct (intentional fire-and-forget; we don't want the lint warning about an ignored promise).
- `compressCommand.test.ts:166, 103, 120, 134` — every existing test now needs `await new Promise((r) => setTimeout(r, 0))` after `await compressCommand.action!(context, '')` to flush the microtask queue before asserting on what the IIFE did. This is a minor but noisy test-side cost of the fire-and-forget refactor; the four `setTimeout(r, 0)` calls are mechanical and consistent.
- `AppContainer.tsx:1313-1320` — `isCompressing` is a `useMemo` that looks for `item.type === MessageType.COMPRESSION && item.compression.isPending` in `pendingHistoryItems`. Pure derivation, no new state, recomputes on `pendingHistoryItems` change. Clean.
- `AppContainer.tsx` (handleFinalSubmit, not in this snippet but referenced by the test) — gates input routing into the queue when `isCompressing` is true. The test at `AppContainer.test.tsx:3636-3641` asserts `messageQueue` contains `'follow up message'` after a submit during a pending compression, and a parallel test at `:3645-3654` asserts that even slash commands like `/help` get queued (and `handleSlashCommand` is NOT called immediately). Both tests are well-targeted at the steering-hint regression risk.
- `useMessageQueue.ts:15` — `isCompressing?: boolean` added to `UseMessageQueueOptions`, defaulting to `false` at `:36`. Backward-compatible default keeps existing call sites unaffected.
- `useMessageQueue.ts:74` — flush gate is now `isConfigInitialized && streamingState === Idle && !isCompressing && isMcpReady && messageQueue.length > 0`. Order in the conjunction is fine; `isCompressing` is added before `isMcpReady` which is the cheapest check, but JS short-circuits left-to-right and `streamingState === Idle` is a fast equality so the cost is irrelevant.
- `AppContainer.test.tsx:3580-3611` — `beforeEach` for the new describe block sets `mockedUseMessageQueue.mockImplementation(realUseMessageQueue)` so the test exercises the *real* hook rather than a stub. This is the right move because the regression risk lives in the real hook's flush gate; stubbing it would defeat the purpose.
- `AppContainer.test.tsx:3614-3622` — explicit assertion `expect(capturedUIState.streamingState).toBe(StreamingState.Idle)` pins the design intent: `isCompressing` must NOT cause `streamingState` to become non-Idle. If a future refactor accidentally re-introduces a streaming-state side-effect for compression, this test fails first.

## Concerns

- The `void (async () => { ... })()` pattern means errors thrown from `tryCompressChat` get caught by the inner `try/catch`, but if the *IIFE itself* throws synchronously before reaching the try (unlikely but possible if e.g. the closure setup throws), the rejection is silently dropped because the outer `void` discards the promise. The inner code is straightforward enough that this is a non-issue in practice.
- `compressCommand` previously returned the action's promise so callers could await it. After this change, callers can no longer wait for compression completion. The PR says no other caller awaits `/compress`, but worth a quick `grep` for `compressCommand.action(` callsites in the broader codebase.
- The four test-file `await new Promise((r) => setTimeout(r, 0))` calls work but are subtle. A small `flushMicrotasks()` test helper would make the intent obvious to the next reader.

## Verdict

**merge-after-nits**

## Rationale

The derived-state architecture is genuinely good — minimal surface area, no new global state, no duplicate-spinner bug — and the test coverage targets the exact regression risks (steering-hint conflict, double-spinner, slash-command queueing). The fire-and-forget IIFE is the standard JS pattern for "I want to do work after returning". The three concerns are non-blocking but worth a one-line response from the author.
