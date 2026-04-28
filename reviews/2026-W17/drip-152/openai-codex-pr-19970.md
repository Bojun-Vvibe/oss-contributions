# openai/codex#19970 — feat: trigger memories from user turns with cooldown

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#19970](https://github.com/openai/codex/pull/19970)
- **Head SHA:** `21b922314086d2795fa99c48106f6e9ce655b13b`
- **Size:** +85 / -50 across 6 files (`codex_message_processor.rs`, `config/types.rs`, `codex_thread.rs`, `phase2.rs`, `state/model/memories.rs`, `state/runtime/memories.rs`)
- **State:** OPEN

## Context

Memory startup work (rollout scanning, phase-2 consolidation) was being
fired from three thread-lifecycle hooks: `thread/start`,
`thread/load`, and `thread/fork`. That has the wrong economics —
threads get created/loaded/forked by IDE plugin actions, app launches,
remote-control sessions, and many other paths where the user never
actually sends a turn. So memory work happened on app warm-up,
consuming model quota and wall time before any real signal that this
session needed memory at all. Plus phase-2 global consolidation could
be re-requested as fast as the lock would let it, with no cooldown.

## Design analysis

Three independent fixes bundled:

1. **Trigger relocation** at `codex_message_processor.rs:6477-6486`.
   Moves the `start_memories_startup_task` call from the three
   thread-lifecycle hooks (deletions at `:2666`, `:4241`, `:4841`) into
   the `thread/sendInput` handler, gated by `turn_has_input` (the
   `!mapped_items.is_empty()` check at `:6378`). The gate is correct:
   empty-input turns (which are valid for some tool-call-only flows)
   shouldn't trigger memory startup. The trigger is per-turn but the
   downstream `start_memories_startup_task` itself is idempotent
   against the in-flight-or-completed startup state, so per-turn
   firing doesn't multi-launch.

   `let turn_has_input = !mapped_items.is_empty();` is computed at
   `:6378` *before* the override-merge logic at `:6379`, so the gate
   reflects the user's submission shape not the post-merge shape.
   Right placement.

2. **Auth manager removal from ListenerTaskContext** at `:545` and
   `:2417`, `:7173`, `:7290`, `:7340`. The `auth_manager` field was
   carried in `ListenerTaskContext` solely to forward to
   `start_memories_startup_task`; since that call no longer fires from
   the listener-task lifecycle, the field becomes dead. Cleanup is
   load-bearing — without it the auth manager would be needlessly
   refcount-bumped on every listener-task creation. The struct
   shrinkage also catches the right thing at compile time: any future
   re-add of memory startup to the listener-task path will need the
   auth manager re-threaded explicitly, which is an opportunity to
   reconsider the policy.

3. **Cooldown + tightened defaults**. `state/runtime/memories.rs`
   adds 54 lines (only the helper signature visible in diff —
   `phase2_global_lock_respects_success_cooldown` test mentioned in
   PR body) introducing `SkippedCooldown` for a six-hour
   successful-run cooldown on global phase-2. `config/types.rs:31-32`
   reduces `DEFAULT_MEMORIES_MAX_ROLLOUTS_PER_STARTUP` from 16 → 2
   and `DEFAULT_MEMORIES_MAX_ROLLOUT_AGE_DAYS` from 30 → 10. Two
   independent bounds: per-startup how many rollouts get scanned
   (drops 8x), and how far back the scan window reaches (drops 3x).
   Combined effect is dramatically tighter — under the new defaults
   a session firing memory startup scans at most 2 rollouts within
   the past 10 days versus the old 16 over 30 days.

   The 6h cooldown on phase-2 success is the right shape for what
   global consolidation actually does — it's a coalescing operation
   over the population of recent memories, not a per-event reaction,
   so re-running it more than 4x/day is wasted compute.

4. **Config plumbing** via new `CodexThread::config()` at
   `codex_thread.rs:380-382`. Returns `Arc<Config>` from the session
   state. This is what lets the new `thread/sendInput` trigger pass
   the *current* (potentially override-merged) config into memory
   startup at turn time, rather than the snapshot taken at thread
   creation. Important: if the user runs `thread/sendInput` with
   per-turn `cwd`/`approval_policy`/etc. overrides, those overrides
   are now reflected in the memory-startup config. Whether that's the
   desired behavior is policy — could argue memory startup should
   always use the *thread-level* baseline config not per-turn
   overrides — but the PR makes it visible at one call site rather
   than buried in the runtime.

## Risks / nits

1. **Per-turn memory-startup firing has surprising cost shape on
   long sessions.** Even though the downstream call is idempotent
   against in-flight startup, the *check* still fires once per user
   turn for the lifetime of the thread. For a session that processes
   hundreds of turns, that's hundreds of cheap-but-non-zero
   `start_memories_startup_task` invocations doing
   already-completed-bail logic. A simple `AtomicBool` short-circuit
   on `CodexMessageProcessor` ("startup already requested for this
   thread") would reduce this to one invocation per thread per
   process-lifetime.

2. **Per-turn config snapshot at `:6485` may diverge from intended
   policy.** `thread.config().await` returns the current effective
   config including any overrides applied to this thread's session.
   If the user runs `thread/sendInput` with `--approval-policy=auto`
   for one specific turn, that policy now leaks into the memory
   config used for startup. That's almost certainly not what the
   memory subsystem wants — memory rollout scanning should use a
   stable thread-level policy, not per-turn intent. Worth a
   one-comment policy declaration: "memory startup uses the current
   thread config snapshot which includes turn overrides; this is
   intentional/unintentional."

3. **Default-tightening is breaking for users with custom configs
   keyed against the old defaults.** Anyone who explicitly set
   `memories.max_rollouts_per_startup = 16` in their config gets the
   *same* behavior as before, but anyone who relied on the default
   for tuning ratios elsewhere (e.g., picked their
   `max_raw_memories_for_consolidation` to be 16 ratio of
   `max_rollouts_per_startup`) now has the ratio shifted 8x. Worth a
   migration note or a config-schema-version bump.

4. **Removed lifecycle-hook trigger is asymmetric on `/resume` and
   `/fork`.** The three deletions remove memory startup from create,
   load, and fork. But a load or fork *might* be the user's first
   contact with a thread that has historical context — if they
   `/resume` an old thread and immediately `Ctrl+C` without a turn,
   the memory state is now *less* consolidated than it would have
   been before. Edge case, but the load case in particular feels
   wrong — a load is much closer to "user is about to engage with
   this thread" than a create-from-IDE-plugin is.

## Verdict

**merge-after-nits.** The architectural shape is right (turn-based
trigger removes ghost cost, success-cooldown bounds repeat
consolidation, tightened defaults bring scan cost in line with the
new trigger cadence). Pre-merge asks: nit 1 (single-fire short-circuit
per thread) and nit 2 (config-snapshot policy comment). Nits 3 and 4
warrant release-notes language and a maintainer policy decision on
the `/resume` case.

## What I learned

- "Memory startup runs on first user turn" is a strictly better
  trigger than "memory startup runs on thread create/load/fork" for
  sessions where IDE plugins or remote-control clients create threads
  speculatively — the per-turn fire only happens when there's actual
  intent.
- Removing a field from a context struct is a useful compile-time
  ratchet against re-introducing the dependency. Once `auth_manager`
  is gone from `ListenerTaskContext`, any future re-add of memory
  startup to that path must re-thread the field through every
  caller, forcing the author to reconsider whether the trigger
  belongs there.
- Success-cooldowns on coalescing operations (here: 6h on phase-2
  global consolidation) are the right shape for any operation whose
  cost is roughly constant per invocation but whose value scales
  with elapsed time since last run. Reactive cooldowns (back-off on
  failure) are different — they bound failure-loop cost; success
  cooldowns bound redundant-success cost.
