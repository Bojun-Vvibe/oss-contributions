# sst/opencode #25572 — feat: add message_click_actions tui config

- **PR:** https://github.com/sst/opencode/pull/25572
- **Head SHA:** `68c427cf9ae0f9002f5bc22277dee84b4c2af019`
- **Author:** macknight
- **Size:** +2 / -0 across 2 files

## Summary

Adds a `message_click_actions` boolean to `TuiOptions` so users can disable the Revert/Copy/Fork popup that opens on every message click. Useful when terminal mouse capture interferes with native selection or when the popup is just noise.

## Specific references

- `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts:27` — new `message_click_actions: z.boolean().optional()` field on `TuiOptions`. Description correctly documents default-true semantics.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1152` — guard `if (tuiConfig.message_click_actions === false) return` placed *after* the existing `getSelectedText()` short-circuit, so the explicit-false opt-out doesn't change the existing selection path.

## Concerns

- The check is `=== false` (strictly opt-out), which matches the `Default: true` doc string. Good.
- No test added, but this is a 2-line config plumb-through with no logic; matching adjacent options (e.g. `mouse`) similarly have no dedicated test.

## Verdict

**merge-as-is** — minimal, additive, default-preserving config option. No functional risk.
