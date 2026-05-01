# openai/codex#20524 — deprecate legacy notify

- Repo: openai/codex
- PR: https://github.com/openai/codex/pull/20524
- Head SHA: `b07de87e48c9`
- Size: +72 / -159 (six source files + tests + deletes `hooks/src/user_notification.rs`)
- Verdict: **merge-as-is**

## What the diff actually does

Coordinated deprecation of the legacy `notify` config surface, which is
the last compat layer from the pre-lifecycle-hook era. Five mechanically
distinct moves:

1. **README at `codex-rs/README.md:46–48`** — rewrites the "Notifications"
   section opening: was "You can enable notifications by configuring a
   script…", now "The legacy `notify` setting is deprecated and will be
   removed in a future release. Existing configurations still work, but
   new automation should use lifecycle hooks instead." The WSL2/Windows
   Terminal toast fallback paragraph is preserved verbatim (so the
   downstream OS-specific behavior contract is unchanged).

2. **TOML schema docstrings** at `codex-rs/config/src/config_toml.rs:126`,
   `codex-rs/core/config.schema.json:3818`, and
   `codex-rs/core/src/config/mod.rs:476/495` — every public-facing
   description string for the `notify` field gains the "Deprecated"
   prefix. The JSON schema mirror is updated in lockstep, which is the
   right shape for a deprecation that ships through codegen.

3. **Startup deprecation event** at `codex-rs/core/src/session/session.rs:558–578`
   — adds a `legacy_notify_configured` boolean computed as
   `config.notify.as_ref().is_some_and(|argv| !argv.is_empty() && !argv[0].is_empty())`
   (the `!argv[0].is_empty()` arm correctly excludes a configured-but-empty
   argv-zero, which is the right edge case). When true, pushes a
   `DeprecationNotice(DeprecationNoticeEvent { summary, details })` event
   onto `post_session_configured_events` with the exact strings:
   - summary: `` `notify` is deprecated and will be removed in a future release. ``
   - details: `Use lifecycle hooks for new automation instead of adding new \`notify\` configurations.`

4. **Two telemetry watchpoints** — `LEGACY_NOTIFY_CONFIGURED_METRIC`
   incremented once at session start when `legacy_notify_configured` is
   true (`session.rs:637–639`), and `LEGACY_NOTIFY_RUN_METRIC` incremented
   per-turn at `core/src/session/turn.rs:579–585` only when
   `!hook_outcomes.is_empty()` (i.e., the legacy notifier actually ran,
   not just was configured). Both come from `codex_otel`, both pass
   `inc=1` and an empty-attribute slice — clean counter-style increments,
   no histograms.

5. **Test coverage** at `codex-rs/core/tests/suite/deprecation_notice.rs:118–148`
   — adds `emits_deprecation_notice_for_notify`, a `multi_thread`
   `tokio::test` that builds a TestCodex with
   `config.notify = Some(vec!["notify-send", "Codex"])`, calls
   `wait_for_event_match` for a `DeprecationNotice` whose summary
   contains `"`notify`"`, and asserts both the summary and details
   strings by exact value. The exact-value assertion pins the operator
   contract — a future contributor can't silently re-word the message.

6. **Hard delete** of `codex-rs/hooks/src/user_notification.rs` (153
   lines). The `UserNotification` enum and its `command_from_argv`-based
   spawn helper are gone; the legacy notify path now flows through the
   generic hook engine, with the deprecation metric and event capturing
   the cutover surface for the eventual removal.

## Why merge-as-is

This is the textbook shape of a "before-removal" deprecation diff:

- **Every public surface flipped in lockstep.** README, TOML
  doc-comment, JSON schema description, Rust struct doc-comment — all
  four say "Deprecated" with consistent wording. The codegen-mirror
  update means SDK users reading the schema from any direction see
  the same signal.
- **Two metrics, two semantics.** Configured-once-per-session vs
  ran-once-per-turn is the right split: it answers two different
  operator questions ("how many users have legacy config?" vs "how
  many turns are still hitting the legacy path?") with the right
  cardinality. The emission decisions are guarded so empty-argv users
  don't pollute the count.
- **Behavior-preserving.** Existing `notify` configs still fire (the
  hook engine processes them); the only new effect is the startup
  event + two metric increments. No user is silently broken.
- **The empty-argv guard is load-bearing and correct.**
  `is_some_and(|argv| !argv.is_empty() && !argv[0].is_empty())` correctly
  excludes both `notify = []` and `notify = [""]`, which a careless
  `is_some()` check would have classified as "configured."
- **The test pins the operator-visible contract by exact string match.**
  `assert_eq!(summary, "...")` rather than `.contains("notify")` means
  the eventual rewording requires updating the test, which is the right
  forcing function.
- **The `user_notification.rs` deletion is the right scope** — the
  generic hook engine already owns the spawn behavior, so keeping the
  legacy module would be dead code mirroring the live path.

## Nits (none blocking)

- The metric names (`LEGACY_NOTIFY_CONFIGURED_METRIC`,
  `LEGACY_NOTIFY_RUN_METRIC`) live in `codex_otel` but the diff doesn't
  show their declaration site — fine because schema mirrors propagate
  via the existing release process, but a one-line link in the PR body
  would help reviewers verify the metric-name string.
- The startup event uses `INITIAL_SUBMIT_ID` for `id`, matching the
  neighboring deprecation events — consistent, no nit.
- Eventual removal will need a separate PR that drops the metric
  declarations after operators confirm cardinality is at zero. The PR
  body does not commit to a removal version, which is correct — that's
  a release-management decision, not a code decision.

## Theme tie-in

Spec the deprecation at every codegen surface and watchpoint at the two
right boundaries before removing anything. The "configured" vs "ran"
metric split is the spec-form way to answer "is it safe to delete yet"
— configuration is migration-easy, but actual runs reveal the long
tail. Both numbers must reach zero before the removal PR is safe.
