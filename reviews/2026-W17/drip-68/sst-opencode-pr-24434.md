# sst/opencode #24434 — feat(tui): show per-message input/output token counts

- **Author:** kunalkohli
- **Head SHA:** `400db2a5ac360e37fa57275fbbde162499143f26`
- **Base:** main (sst/opencode)
- **Size:** +12 / -0 (2 files)
- **URL:** https://github.com/sst/opencode/pull/24434

## Summary

Surfaces per-message input/output token counts in two places in the
TUI: (1) the sidebar `context` plugin's footer summary and (2) the
metadata footer under each assistant message in the session view. Pure
additive presentation change — no schema, no wire format, no calc
logic moved.

## Specific findings

- `packages/opencode/src/cli/cmd/tui/feature-plugins/sidebar/context.tsx:23-24`
  — adds `input: 0, output: 0` to the early-return shape when there's
  no last assistant message yet. Symmetry with the steady-state branch;
  prevents the JSX consumer below from rendering "undefined" briefly
  during route transitions.
- `context.tsx:34-35` — `input: last.tokens.input + last.tokens.cache.read`
  and `output: last.tokens.output + last.tokens.reasoning`. The
  decisions:
    - **Cache reads counted as input** — this is correct for a "what
      did I send" framing that matches the way most billing reads, but
      worth noting in release notes that the user-visible "input"
      number is *not* the prompt-tokens-billable number for providers
      that bill cache reads at a discount. A small `(includes
      cache reads)` tooltip might prevent confusion later.
    - **Reasoning tokens counted as output** — also defensible (they
      are model-generated) but again diverges from the way some
      providers bill reasoning separately.
- `context.tsx:46` — new `<text>` line: `{state().input.toLocaleString()}↓ in · {state().output.toLocaleString()}↑ out`.
  Uses `↓`/`↑` glyphs and `·` separator, consistent with the existing
  `tokens · percent · spent` rhythm above. `.toLocaleString()` will
  render thousands separators per the user's locale (`12,300` vs
  `12.300`) — matches the established format on the line above
  (`{state().tokens.toLocaleString()} tokens`).
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1430-1436`
  — the assistant-message footer addition is gated by `<Show
  when={props.message.tokens.input > 0 || props.message.tokens.output > 0}>`.
  This is the right gate: messages with zero token accounting (e.g.
  freshly streaming, error states, or aborted-before-billing) won't
  render a misleading `0↓ 0↑` line.
- `index.tsx:1432-1435` — uses `Locale.number(...)` here, not
  `.toLocaleString()` as in the sidebar. Two different formatters for
  the same kind of value across two files is a small inconsistency;
  `Locale.number` is presumably the project's wrapper. Worth picking
  one and using it in both places.
- `index.tsx:1432-1435` — same `input + cache.read` and `output +
  reasoning` math as the sidebar. Math is duplicated rather than
  factored into a helper. For a 12-line diff that's fine; if a third
  surface ever needs it (e.g. JSON export, status bar), pull it into
  `tokens.ts` then.

## Verdict

`merge-after-nits`

## Reasoning

The functional change is small, defensive, and correct. Two minor
nits worth a review comment but not blocking:

1. **Inconsistent number formatter** between `context.tsx`
   (`.toLocaleString()`) and `index.tsx` (`Locale.number(...)`) — pick
   one. The sidebar already uses raw `.toLocaleString()` on the line
   above, so there's a local-consistency argument either way.

2. **Cache-reads-as-input + reasoning-as-output framing** is fine but
   should be called out in release notes / a tooltip; users comparing
   the displayed number to a provider's billing dashboard will see a
   mismatch on cache-heavy or reasoning-heavy turns.

Neither nit is a correctness issue. The `<Show when={...}>` gate is
exactly what I'd want for the zero-state case. Author is closing a
real issue (#24433). Ship after addressing the formatter consistency.
