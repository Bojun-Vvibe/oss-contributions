# QwenLM/qwen-code #3720 — Background tasks unified overlay (subagents + shells)

- **PR:** https://github.com/QwenLM/qwen-code/pull/3720
- **Head SHA:** `314692e7f`
- **Size:** +500 / −100

## Summary
Unifies the BackgroundTasksDialog overlay so it surfaces both background subagent entries and managed shell entries as a single ordered list. Introduces a `DialogEntry` discriminated-union type (`{ kind: 'agent', ...BackgroundTaskEntry } | { kind: 'shell', ...BackgroundShellEntry }`), threads it through `useBackgroundTaskView`, and dispatches cancel through the right registry per `kind`. Adds `setStatusChangeCallback` semantics on `BackgroundShellRegistry` that mirror the agent registry (fire on register too, no-fire on no-op transitions).

## Specific observations

1. **`DialogEntry` discriminated union at `useBackgroundTaskView.ts:711-713`:**
   ```ts
   export type DialogEntry =
     | (BackgroundTaskEntry & { kind: 'agent' })
     | (BackgroundShellEntry & { kind: 'shell' });
   ```
   Inline shape preservation (`BackgroundTaskEntry & { kind: 'agent' }` keeps `agentId`, `description`, `abortController` etc. directly accessible) means the existing renderer code on the agent branch only needs a `if (entry.kind === 'agent')` guard, no field-renaming. Smart — minimizes diff.

2. **`entryId(entry)` helper at `useBackgroundTaskView.ts:721-723`:** stable id resolution `entry.kind === 'agent' ? entry.agentId : entry.shellId`. Used as React key + for lookup. Critical because the two registries assign ids from independent generators — without this, a `key={entry.agentId}` falls back to `undefined` for shell entries and React would re-mount on every render. Test at `BackgroundTasksDialog.test.tsx:21-22` re-exports the helper inline under the mocked module so the renderer can still resolve it — this is the "mock the module but preserve the helper" pattern, slightly fragile (drift risk between inline impl and real impl) but pragmatic.

3. **Two-registry merge at `useBackgroundTaskView.ts:742-755`:**
   ```ts
   const merged = [...agentEntries, ...shellEntries].sort(
     (a, b) => a.startTime - b.startTime,
   );
   ```
   Sorted by `startTime` so launch order is preserved across both registries. The comment at line 749-751 calls out the alternation case explicitly. Both registries' `setStatusChangeCallback` are wired to the same `refresh` function, with cleanup unsetting both on unmount. Standard React effect hygiene.

4. **Cancel dispatch at `BackgroundTaskViewContext` (line 654-669):** `target.kind === 'agent' ? cancel(target.agentId) : requestCancel(target.shellId)`. The asymmetry — `cancel()` on agents vs `requestCancel()` on shells — is documented in the inline comment at lines 660-664: shell cancel goes through `requestCancel` which only triggers the `AbortController`, then the spawn's settle path records the real terminal moment + outcome. This mirrors the `task_stop` tool path from PR #3687 (cited inline). Correct — bypassing the settle path would let the dialog mark a shell as cancelled while the OS process is still running.

5. **`requestCancel` is no-fire on already-terminal entries** — verified by the test at `backgroundShellRegistry.test.ts:832`: `reg.requestCancel('a'); // already terminal — also no fire`. This is the load-bearing assertion that prevents the dialog from infinite-looping on a stuck-terminal entry where the user repeatedly hits cancel. The test asserts `expect(transitions).toEqual([])` for the whole no-op cluster (`complete`/`fail`/`cancel`/`requestCancel` all on terminal entry).

6. **Error resilience: "keeps the registry usable when a callback throws"** at line 837-839 (visible in the diff). This is a new test that pins down the contract that a throwing callback doesn't poison the registry's internal state. Good defensive coverage.

7. **Section header rename `Local agents` → `Background tasks`** at lines 184 and 193. Mechanical and correct given the dialog now shows both kinds. The empty-state text is preserved; only the bold header label changes.

8. **`STATUS_VERBS` and `terminalStatusPresentation` retyped from `BackgroundTaskEntry['status']` to `EntryStatus = DialogEntry['status']`** at lines 121, 130, 139. Comment at line 117-120 explains: the shell status union widens the agent enum but they share the same four values (`running`/`completed`/`failed`/`cancelled`), so handlers keyed on the agent enum still cover every shell case. Worth verifying via a `satisfies` assertion or a unit test that the two unions stay congruent — if the shell registry ever adds a fifth status, this code silently misses it.

9. **`elapsedFor` retyped to `{ startTime: number; endTime?: number }`** at line 161. Structural type instead of nominal `BackgroundTaskEntry`. Good — makes it work for both kinds without per-kind branches.

10. **Test harness at `BackgroundTasksDialog.test.tsx:78-83` adds a stub `get(id)` method on the mocked registry:** "the dialog now re-reads agent entries via `.get()` to pick up live activity/stats mutations the snapshot misses." This is a real correctness fix — `useBackgroundTaskView` ignores `appendActivity` (documented in the hook's docstring as intentional, to avoid render churn from tool-call traffic), but the *detail view* needs the live mutation. So the dialog reaches into the registry for the freshest entry by id when rendering detail. Test stubs this with a closure over `currentEntries`.

## Risks

- **Status-union widening** (point 8): if `BackgroundShellRegistry` adds a status value the agent registry doesn't have, the `EntryStatus` cast in `STATUS_VERBS` becomes a lie at runtime. A `satisfies Record<DialogEntry['status'], string>` on `STATUS_VERBS` would catch this at compile time.
- **`DialogEntry` inline mock-helper** in `BackgroundTasksDialog.test.tsx:20-22` re-implements `entryId` inline. If the real `entryId` ever changes (e.g. composite keys for multi-tenant), the test silently keeps using the old behavior. Worth a follow-up PR that uses `vi.importActual` to re-export the real helper.
- **Two `setStatusChangeCallback` registrations to the same `refresh` function** mean every shell-status change triggers a re-merge of *both* lists. Cheap when the lists are small (<20 entries each), but if either grows large, switching to a per-registry incremental update would be lower-overhead.
- **`buildBackgroundEntryLabel` is only called for `kind === 'agent'`** (line 150-151), and the shell branch hardcodes `[shell] ${entry.command}` (line 156-157). If shell labels ever need richer formatting (truncation, syntax highlighting hints), this becomes a parallel-implementation drift point.

## Suggestions

- **(Recommended)** Add `satisfies Record<DialogEntry['status'], string>` to `STATUS_VERBS` so the union-widening assumption is compile-checked.
- **(Recommended)** Use `vi.importActual('../../hooks/useBackgroundTaskView.js')` in the test mock so `entryId` doesn't drift between real and stub impls.
- **(Optional)** Consider extracting a `BackgroundEntryLabel` component that branches on `kind` internally, so renderers don't have to remember the `[shell]` prefix convention.
- **(Optional)** Telemetry: emit a counter for `dialog.refresh` per registry source so future profiling can distinguish "agent-driven churn" from "shell-driven churn."
- **(Nit)** Sort comparator at line 752-754 doesn't break ties — two entries with identical `startTime` (rare but possible at sub-ms resolution) would render in insertion order from the spread `[...agentEntries, ...shellEntries]`, which means agents always win ties. Probably fine, but a deterministic tiebreaker on `entryId` would lock the order.

## Verdict: `merge-after-nits`

Solid unification work. The discriminated-union approach is the right shape (preserves existing renderer code on the agent branch, adds explicit `kind === 'shell'` handling for the new branch). The cancel-path asymmetry is correctly handled and documented. Test coverage on the registry callback semantics is thorough. The two real concerns — status-union widening assertion and inline mock helper drift — are documentation/compile-check level, not correctness blockers.

## What I learned

Merging two parallel "live entity registry" surfaces into one view-model is one of those refactors where the discriminated union pays for itself many times over: every renderer site that needs to dispatch on kind is forced (by exhaustive-switch typechecking) to handle both branches, and the inline-shape-preservation trick (`BackgroundTaskEntry & { kind: 'agent' }` instead of `{ kind: 'agent', entry: BackgroundTaskEntry }`) keeps the diff to existing renderer code minimal. The cost is a leaky abstraction at registry-callback wiring (you have to remember to subscribe to *both* registries for *every* derived view), but a quick "subscribe to all live registries" hook abstraction can close that gap if a third kind ever lands.
