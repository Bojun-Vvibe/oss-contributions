# BerriAI/litellm #27218 — [UI] Rename "Default" key type to "Full Access" and reorder dropdown

- Author: ryan-crabbe-berri
- Head SHA: `1c50fd1a115ce85f08d962cc2e054eb1d99d2239`

## Files changed (top)

- `ui/litellm-dashboard/src/components/organisms/create_key_button.tsx` (+15 / -15)

## Observations

- `ui/litellm-dashboard/src/components/organisms/create_key_button.tsx:982-1006` — The dropdown is reordered: previously `default → llm_api → management`, now appears to be `llm_api → management → full_access` (the formerly-"default" option) per the diff context (only the first two `Option` blocks show in the truncated patch but the rename + reorder is clear from the diff structure). The on-the-wire `value` for the renamed option needs verification — if `value` stays `"default"` then this is purely cosmetic and safe; if it changes to `"full_access"` then any persisted backend config / API client carrying `"default"` will stop matching.
- `:9` — `import { ..., Typography } from "antd"` added to render the option labels with `Typography.Text strong` instead of inline `<div style={{ fontWeight: 500 }}>`. This is a styling refinement that's consistent with the rest of the antd component vocabulary used in the file. Good.
- `:982-996` (visible context) — the formerly-"default" Option's wire `value="default"` is still present in the truncated visible portion; assuming the rest of the file preserves that, this is **label-only** which is the safe choice. **Verify**: maintainer should confirm the value props stay `"default"` / `"llm_api"` / `"management"` (or whatever the backend expects) to avoid silent breakage.
- The label "Full Access" is clearer than the ambiguous "Default" — UX improvement is real (users have been confused about whether "Default" means "the default option" or "the default permission set").
- Reordering a dropdown can change muscle-memory for existing users; a brief note in release notes would smooth the transition. Not blocking.
- No tests added/modified, but this is a label-rename + reorder and the file has limited test coverage already; consistent with codebase norms.

## Verdict

`merge-after-nits`

Pure UI/UX polish with no behavior change *if* the wire-level `value` props stay stable. Single nit: maintainer should explicitly confirm in PR body that the renamed Option's `value` prop is still `"default"` (not changed to `"full_access"`) to avoid breaking persisted keys or API clients. If values were changed, this should be flagged as `request-changes` instead.
