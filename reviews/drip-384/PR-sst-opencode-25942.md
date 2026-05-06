# sst/opencode#25942 — feat(tui): add width_method config option for ZWJ emoji handling

- **Head SHA**: `681ce24dc6cc0dee12cd6fc3780a0443706bed4f`
- **Stats**: +7 / -0, 2 files

## Summary

The TUI renderer's character-width method was hardcoded, so terminals/multiplexers (notably tmux on older Linux distros) that mis-render ZWJ emoji sequences (`👨‍👩‍👧‍👦` etc.) caused cursor drift and layout corruption. This PR exposes the renderer's existing `widthMethod` knob via a new `tui.width_method` config option with three values: `unicode` (default, current behavior), `wcwidth` (POSIX `wcwidth(3)` semantics), and `no_zwj` (ignore ZWJ joiners — fixes the tmux/legacy case).

## Specific citations

- `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts:27-32`: zod enum `["unicode", "wcwidth", "no_zwj"]` plus `.optional()` so existing configs are unaffected. Description text correctly names the failure mode (tmux + older terminals) and the symptom ("ZWJ emoji sequences cause layout corruption").
- `packages/opencode/src/cli/cmd/tui/app.tsx:77`: pipes `_config.width_method` through to `CliRendererConfig.widthMethod`. Forwarding `undefined` is the right call — leaves the renderer's internal default in place when the option isn't set, so existing user configs render identically.

## Verdict

**merge-after-nits**

## Rationale

Tiny, well-scoped surface that correctly defers to the renderer's existing capability rather than reinventing width logic in opencode. Three nits: (1) the description text says "Default: 'unicode'" but the schema doesn't `.default("unicode")` — the actual default is "whatever the renderer chooses when fed `undefined`," which today is `unicode` but is an implicit coupling that could silently change if the renderer dependency ever updates. Either set `.default("unicode")` here or change the description to "Default: renderer default (currently unicode)"; (2) no test coverage — a single bun:test asserting that an absent config produces `widthMethod: undefined` and a set config of `"no_zwj"` propagates would lock the wiring. (3) the variable shadow `_config` (underscore prefix) suggests the parameter was previously unused; the leading underscore is now misleading because the parameter *is* used. A small follow-up rename to `config` would clean this up. None of these block merge — the feature is a targeted, opt-in escape hatch with zero default-behavior risk.
