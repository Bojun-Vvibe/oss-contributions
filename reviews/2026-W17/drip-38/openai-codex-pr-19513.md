# openai/codex #19513 — Delay approval prompts while typing

- **Repo**: openai/codex
- **PR**: [#19513](https://github.com/openai/codex/pull/19513)
- **Head SHA**: `2e21d2256af785cd22b71711b15e5a8c6a804575`
- **Author**: etraut-openai
- **State**: OPEN (+248 / -8)
- **Verdict**: `merge-after-nits`

## Context

Issue #7744. While the user is type-ahead in the composer, an
approval modal can pop up underneath the next keystroke and a plain
`y`/`a` gets eaten by the modal as an approval shortcut instead of
landing in the draft. Classic "modal stole my keystroke" bug — most
visible with auto-approve regressions and trust prompts.

## Design

Adds a 1-second composer-idle gate before any new approval modal
appears.

1. `APPROVAL_PROMPT_TYPING_IDLE_DELAY = Duration::from_secs(1)`
   constant at `codex-rs/tui/src/bottom_pane/mod.rs:142` — module
   scope, single source of truth.
2. New `BottomPane` fields (lines 192-195): `delayed_approval_requests:
   VecDeque<DelayedApprovalRequest>` and `last_composer_activity_at:
   Option<Instant>`. `VecDeque` rather than `Vec` because the FIFO
   pop semantics matter (see point 5).
3. `approval_prompt_delay_remaining` (lines 459-466) is the single
   predicate — `last_activity + idle_delay - now`, filtered to
   non-zero. `checked_add`/`checked_duration_since` keep this safe
   against `Instant` saturation, which is the right paranoia for a
   long-running TUI.
4. Composer key gate (lines 568-587): only `Press`/`Repeat` events
   without `CONTROL`/`ALT` modifiers, restricted to
   `Char | Backspace | Delete | Enter | Tab` count as activity.
   Deliberately excludes arrow keys and ctrl combos — those are
   navigation, not "I'm composing a thought." Good carve-out.
5. `maybe_show_delayed_approval_requests_at` (lines 470-486) drains
   the queue when idle: pops the oldest with `pop_front`, then drains
   the rest from the back via `pop_back` and feeds them into the
   overlay's internal queue with `enqueue_request`. The comment
   ("`ApprovalOverlay` advances its internal queue with `pop()`, so
   drain... from the back to preserve FIFO display order") is the
   kind of co-implementation detail that's worth leaving in code —
   reverse it without reading the comment and you get LIFO order in
   the overlay.
6. `push_approval_request` (lines 1070-1086) routes through the
   delayed path if either the queue is non-empty or the idle window
   is still open; otherwise opens immediately (preserving today's
   no-typing-no-delay behavior).
7. `cancel_approval_for_resolved_request` (lines 1200-1212) was
   updated to also drop matching requests from `delayed_approval_
   requests`. Without this, a server-side cancel would leave a
   ghost approval in the delayed queue and surface it later out
   of context. `matches_resolved_request` had to be promoted from
   private to `pub(super)` for this — minimal scope bump.
8. Paste path (lines 645-650) gates on `has_pasted_text` so empty
   pastes don't reset the idle clock.

Tests cover the matrix:
- `approval_request_shows_immediately_without_recent_typing`
  (line 1675-1685) — null hypothesis.
- `approval_request_is_delayed_after_recent_typing`
  (line 1687-1700) — primary case.

## Risks

- 1s is hard-coded. Heavy typists with slow approval loops will
  perceive a delay; users who only ever stab `y` in response to a
  prompt won't notice. No config knob — fine for v1, worth a
  follow-up if anyone complains.
- The `KeyEventKind::Press | Repeat` filter is correct but assumes
  `Repeat` events fire (they require `enhanced_keys_supported` on
  some terminals). Held-down keys on basic terminals may not extend
  the idle window. Acceptable: held-down letters in a chat composer
  are uncommon.
- Queue is unbounded. A misbehaving agent flooding approvals while
  the user types could grow the deque. Realistic floor is "tens" —
  no leak in practice.

## Suggestions

Nit: `record_composer_activity_at` (line 469) requests a redraw
in the future via `request_redraw_in(delay)` only if the queue is
non-empty. That's right — but if the queue is empty and a new
approval lands a microsecond later, the next `pre_draw_tick_at`
will re-request the redraw on the same code path (line 491). So
the omission is intentional, not a bug. Worth a `// no queue, no
need to wake the renderer` comment for the next reader.

## What I learned

The clean part of this design is putting "what counts as composer
activity" into one filter expression rather than scattering
`record_composer_activity()` calls into `composer.handle_key_event`.
The activity definition (printable + edit keys, not modifiers/nav)
is policy, and policy belongs at the boundary, not buried inside
the composer. Once you see that, the symmetry with the paste-path
guard (`has_pasted_text`) falls out naturally.
