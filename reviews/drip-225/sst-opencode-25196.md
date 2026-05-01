# sst/opencode#25196 — fix(opencode): split Bedrock Claude into 200K and 1M catalog rows

- Link: https://github.com/sst/opencode/pull/25196
- Head SHA: `e313a3391fd5a6c55421e54f58af86adf241114a`
- Files: `packages/opencode/src/provider/provider.ts` (+32/−0)

## Notes
- Adds `splitBedrock1m(pid, models)` helper at `provider.ts:1042-1068` that, for `amazon-bedrock` provider only (`if (pid !== "amazon-bedrock") return` at `:1046`), walks the model catalog and emits a sibling `${id}-1m` row for each Claude opus/sonnet 4.x entry — the right "catalog enrichment at one boundary" shape rather than per-call-site context-limit arithmetic. Called once at `:1095` after the upstream catalog merge but before the `Info` return.
- Load-bearing distinction at `:1051` between `BEDROCK_1M_NATIVE = ["claude-opus-4-7"]` (no beta header needed) and `BEDROCK_1M_MODELS` requiring `BEDROCK_1M_BETA = "context-1m-2025-08-07"` — matches Bedrock's actual contract where opus-4-7 has the 1M context built in but other 4.x models still gate it behind the beta header. Native-vs-beta branching at `:1062` (`const betas = native ? existing : [...new Set([...existing, BEDROCK_1M_BETA])]`) preserves any existing `anthropicBeta` array (deduped via `Set`) rather than clobbering operator-set headers.
- The 200K row mutates the original (`model.name = `${name} (200K)`` at `:1056`, `model.limit.context = Math.min(model.limit.context, 200_000)` at `:1058`) which means upstream models.dev catalog entries already at 200K stay at 200K; the 1M row is a clone with `id: ModelID.make(`${id}-1m`)` and `limit.context: 1_000_000` at `:1063-1067`. Idempotent guard at `:1049` (`if (id.endsWith("-1m")) continue`) prevents double-suffix on re-entry.
- Name-stripping regex at `:1054` (`/\s+\((200K|1M Experimental)\)$/i`) tolerates upstream catalog entries that already carry a `(200K)` or `(1M Experimental)` suffix — the case-insensitive flag and `\s+` defend against minor whitespace drift in the source catalog.

## Nits
- `BEDROCK_1M_MODELS.some((m) => model.api.id.includes(m))` at `:1048` uses substring match, which would false-match an `id` like `bedrock-claude-opus-4-6-finetune` — a `===` or anchored regex would be tighter, though the current Bedrock id space doesn't have collisions yet.
- `delete opts["anthropicBeta"]` at `:1055` then `options: { ...model.options, ...(betas.length > 0 ? { anthropicBeta: betas } : {}) }` at `:1066` is slightly indirect — could be one expression, but the current shape makes the "clear-then-rebuild" intent obvious to readers, so this is borderline.
- Zero unit test for the helper. A fixture catalog entry for `bedrock/claude-sonnet-4-5` with no `anthropicBeta`, plus another with an existing `["foo"]` value, asserting the resulting two-row split contains the exact `(200K)`/`(1M)` names + correct beta-header dedupe, would pin the contract against future "we renamed the beta header" provider-side breakage.

**Verdict: merge-after-nits**
