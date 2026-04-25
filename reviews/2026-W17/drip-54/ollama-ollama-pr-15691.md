# ollama/ollama PR #15691 — add token calculation support with UI display

@19b96ee3 · base `main` · +261/-3 · author `KunjShah95`

## Summary
Adds a new `POST /api/tokenize` endpoint plus a chat UI that surfaces per-message and session-aggregate token counts. Closes #15639.

## What changed
- `api/types.go:1298+` — new `TokenizeRequest`/`TokenizeResponse`/`TokenizeDetail` structs. Keeps a backward-compat `tokens` field aliased to `input_tokens`.
- `server/routes.go:1715` — registers `POST /api/tokenize` → `s.TokenizeHandler`.
- `server/tokenize.go` (new, 127 lines) — handler implementation.
- `app/ui/app/src/components/Chat.tsx:67-114` — adds `SessionTokenStats` aggregating `metrics.prompt_eval_count` / `metrics.eval_count` across messages.
- `app/ui/app/src/components/Message.tsx:7-37` — per-message `TokenStats` block on assistant messages.

## Key observations
- API surface mixes two concerns: `Tokens` (backward-compat alias for `InputTokens`) plus `OutputTokens` derived "from detokenization" — the comment at `api/types.go:36-38` says output is *estimated*, which is surprising for an endpoint named `/tokenize`. Document this clearly or split into two endpoints.
- UI uses `(msg as any).metrics` (`Chat.tsx:81`, `Message.tsx:8`) — typing escape hatch; would be cleaner to extend `Message` in `gotypes`.
- `SessionTokenStats` JSX has a stray `)` on line 113 / mis-indented closing — passes only because TS forgives it; worth tidying.
- Inline SVG icons duplicated between `Chat.tsx` and `Message.tsx`; extract to shared component.
- No test for `TokenizeHandler` visible in the diff window — backend handler at `server/tokenize.go` (127 lines new code) deserves table-driven coverage for prompt vs messages branches and capability gating.

## Risks/nits
- Adding fields to public `api/types.go` is a wire-format change; needs a changelog note.
- UI shows tokens only when `metrics.eval_count > 0`, which means streaming responses won't display until completion — fine, but worth noting.

**Verdict: request-changes**
