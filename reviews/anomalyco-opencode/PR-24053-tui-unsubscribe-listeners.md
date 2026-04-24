# PR-24053 — fix(tui): unsubscribe event listeners on component disposal

[sst/opencode#24053](https://github.com/sst/opencode/pull/24053)

## Context

The Solid-based TUI in `packages/opencode/src/cli/cmd/tui/app.tsx`
registered ~6 long-lived `event.on(...)` subscriptions
(`CommandExecute`, `ToastShow`, `SessionSelect`, `session.deleted`,
`session.error`, `installation.update-available`) directly at
component-render time, without capturing the unsubscribe handles. On
session switch / route teardown, the closures (and everything they
captured: `route`, `toast`, `dialog`, `kv`, `sdk`) leaked. PR
restructures the registrations into a `const unsubs: Array<() =>
void> = [...]` collected from each `event.on(...)` return and calls
them in `onCleanup(() => { for (const unsub of unsubs) unsub() })`.

Same pattern is applied to `prompt/index.tsx`
(`TuiEvent.PromptAppend`), `context/exit.tsx` (`process.on("SIGHUP")`
now paired with `process.off`), `context/keybind.tsx` (`clearTimeout`
on the leader-key `setTimeout`), and `context/project.tsx` /
`context/sync.tsx` / `routes/session/index.tsx`.

## Strengths

- Idiomatic Solid: `onCleanup` is the correct disposal hook, and
  collecting unsubscribes into a single array is cleaner than 6
  separate `onCleanup` calls.
- The `process.on("SIGHUP")` pairing in `exit.tsx` is the kind of
  leak that's nearly invisible until you watch process listener
  count climb across reconnects — good catch.
- Author notes "I've been using this for weeks locally" — non-trivial
  burn-in time for a TUI rendering fix.
- The keybind `setTimeout` cleanup (`if (timeout) clearTimeout(timeout)`
  in `onCleanup`) prevents a stale timer from firing into a torn-down
  store.

## Concerns / risks

- **Async event handlers are not awaited before disposal.** The
  `installation.update-available` handler is `async` and awaits
  `DialogConfirm.show`, then `sdk.client.global.upgrade`, then
  `DialogAlert.show`. If `onCleanup` fires while one of those
  awaits is in flight (e.g. the session is replaced during an
  upgrade dialog), the dialog promise will resolve into a destroyed
  Solid context. The unsubscribe stops *new* events but doesn't
  cancel the in-flight handler. Consider adding an `AbortController`
  or a `disposed` flag guarded at each `await` boundary.
- **All-or-nothing cleanup.** Wrapping every subscription in one
  array means a single throw inside one unsub stops the rest. Wrap
  the loop with `try { unsub() } catch (e) { console.warn(...) }`
  per iteration so a third-party event bus that throws on double-
  unsubscribe doesn't strand the remaining handlers.
- **Order of effects.** `onCleanup` callbacks fire in reverse
  registration order in Solid. The unsubscribes fire in array order
  here. For these specific listeners that's fine — they're
  independent — but if a future addition introduces a cleanup that
  must run *before* the event handlers detach (e.g. a flush of
  pending `toast.show` queued from an event), the ordering will
  surprise. Worth a one-line comment.
- **`prompt/index.tsx` only collects one unsubscribe.** That's
  fine for now (only `PromptAppend` is registered), but the pattern
  diverges from `app.tsx`'s array — if a second `event.on` is
  added later, a maintainer is likely to add another bare
  registration and skip the `onCleanup` pairing. Standardize on the
  array pattern even for n=1.
- **No regression test.** Listener-leak bugs are easy to assert by
  counting `event.on`/`event.off` deltas across mount/unmount of a
  test root. Without one, the next refactor of `app.tsx` will
  regress this silently.

## Verdict

**Approve.** Correct and clearly motivated fix. Follow-ups: add a
listener-count regression test, guard the async update-available
handler against post-disposal resolution, and either standardize the
array pattern in `prompt/index.tsx` or document why the divergence is
deliberate.
