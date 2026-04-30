# google-gemini/gemini-cli #26241 — fix(cli): resolve tmux scroll issue by using ink's useStdout for terminal size

- **URL:** https://github.com/google-gemini/gemini-cli/pull/26241
- **Head SHA:** `86b7b72aed9a523eef4ab06e003d8c575b2a9f66`
- **Files:** `packages/cli/src/ui/hooks/useTerminalSize.ts` (+11/-8)
- **Verdict:** `merge-after-nits`

## What changed

`useTerminalSize` previously read terminal dimensions from `process.stdout.columns` / `process.stdout.rows` and bound the resize listener directly on `process.stdout`. Inside ink (which can render to an alternate output stream — e.g. when invoked from tmux pane management, a wrapped tty, or a test harness with a mocked stdout), `process.stdout` may not match the stream ink is actually drawing into. The result: the cached `size` state goes stale relative to the *render target's* dimensions, manifesting as the reported tmux scroll glitch where the pane re-renders at the wrong width and trailing rows scroll incorrectly.

Fix routes through ink's `useStdout` hook at `useTerminalSize.ts:9`:

```typescript
const { stdout } = useStdout();
const [size, setSize] = useState({
  columns: stdout?.columns || process.stdout.columns || 60,
  rows: stdout?.rows || process.stdout.rows || 20,
});
```

…with the resize listener now bound to `target = stdout || process.stdout` at `:31-32` (and removed in cleanup at `:34-35`), and `[stdout]` added to the effect deps at `:38` so a stdout swap re-binds the listener.

## Why it's right

- The triple-fallback chain `stdout?.columns || process.stdout.columns || 60` is the right defensive shape: prefer ink's render-target stream, fall back to the global process stdout, and finally to a sane default of 60 cols / 20 rows. The `||` chain means a stream with `columns === 0` (some pty wrappers report this transiently during resize) falls through to the next tier — that's the right behavior because 0 cols is unrenderable, but technically it'd be preferable as `??` for `null`/`undefined`-only narrowing. As written, this is permissive in the right direction.
- The `target = stdout || process.stdout` resolution at `:31` and the symmetric cleanup at `:35` mean the listener is bound and unbound on the *same* stream — no listener leak if `stdout` swaps mid-component-life. The `[stdout]` dep at `:38` is what ensures this: when `stdout` changes, the effect tears down the old listener and binds to the new stream.
- Defensive `?.` chaining on `stdout?.columns` handles the case where `useStdout()` returns `{ stdout: undefined }` early in the ink render lifecycle (which it can during the first synchronous render before ink has fully initialized the writeable stream).

## Nits / blockers

1. **`||` vs `??` for the size fallback.** `stdout?.columns || process.stdout.columns || 60` will fall through to `process.stdout.columns` when `stdout.columns === 0`. This is *probably* what you want for the tmux-resize transient case, but it's worth a one-line comment explicitly naming "0-col is a transient resize state, treat as unknown and re-fall back". A `?.` purist would write `stdout?.columns ?? process.stdout.columns ?? 60`, which would *trust* a 0 value — the wrong choice here. Documenting the `||` choice prevents a well-intentioned future refactor that "modernizes" to `??`.

2. **Missing test.** This is a behavior fix that's load-bearing for tmux/wrapped-pty users, but the diff ships zero test coverage. Mocking `useStdout` to return `{ stdout: { columns: 80, rows: 24, on: jest.fn(), off: jest.fn() } }` and asserting:
   - The hook returns `{columns: 80, rows: 24}` (uses ink's stdout, not `process.stdout`).
   - On `target.on('resize', ...)` invocation, `setSize` is called with the new dimensions.
   - On unmount, `target.off('resize', ...)` is called with the same handler reference.
   - When `stdout` changes (e.g. ink swaps streams), the listener is rebound on the new stream and removed from the old.
   
   …would lock the contract. Without these, a future refactor that drops the `[stdout]` dep array (or accidentally narrows the fallback chain) silently regresses without any test failing.

3. **Race on `target` capture.** The `useEffect` callback closes over `target` computed once at effect-run time. If `stdout` changes *value* without changing *reference* (e.g. ink mutates the stdout object in-place — unusual but not impossible for pty wrappers that swap underlying streams), the listener stays bound to the old underlying stream while reads happen against the new one. The `[stdout]` dep covers reference changes, not in-place mutation. This is a corner case worth a defensive `target.removeListener('resize', updateSize)` on every re-bind, but probably not worth blocking on.

4. **`updateSize` re-reads `stdout?.columns` from the closure.** Because `updateSize` is defined inside the effect, it captures the `stdout` from when the effect ran. If the effect re-runs due to `[stdout]` change, a *new* `updateSize` is created with the new `stdout` reference — correct. But verifies that the old listener was bound to the old `target` and is removed via the old cleanup, not the new one. The current symmetric `target` capture in cleanup at `:35` does this correctly.

5. **`useStdout()` may not be available outside ink's rendering context.** If `useTerminalSize` is ever called from a hook composed outside an `<App>` boundary (e.g. in a unit test), `useStdout()` returns the default which may not have `columns`/`rows`. The fallback to `process.stdout.columns` covers this, but the test-mocking story should ensure `useStdout` is always available.

## Risk

Low. Surface area is one tiny hook used by render-layout code. The fallback chain preserves existing behavior in environments where `useStdout()` returns `{stdout: undefined}` (e.g. tests, non-ink callers). Worst case if ink's `useStdout` is unstable across versions, the hook silently uses `process.stdout` and the bug returns — same as current behavior, no regression. The lack of test (#2) is the highest-leverage nit because this is exactly the kind of fix that quietly regresses on the next ink upgrade without anyone noticing until tmux users complain again.
