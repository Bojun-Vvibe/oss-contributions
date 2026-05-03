# QwenLM/qwen-code PR #3809 — feat(core): hint to background long-running foreground bash commands

- Author: wenshao
- Head SHA: `acc2b3f6df70d5aa3736846728be8540f6a3e486`
- Verdict: **merge-as-is**

## Rationale

Adds an advisory hint appended to the shell-tool result whenever a
foreground command runs ≥ `effectiveTimeout / 2` (60s for the 120s
default). Tests in `packages/core/src/tools/shell.test.ts:788-1052` are
unusually thorough: success path at line 805, under-threshold suppression
at line 818, error-exit still hints at line 832, abort/cancel suppression
at line 851, distinct timeout-vs-cancel branch at line 880 (with proper
`AbortSignal.any` mocking + cleanup in `finally`), external-signal
SIGTERM suppression at line 936, off-by-one boundary at line 968, and
post-truncation insertion order at line 988. The boundary tests pin the
`>=` semantics so a regression to `>` would fail loudly. Hint text
references `is_background: true` and `/tasks` so the LLM has actionable
next-step guidance. Earlier round at SHA `fd93f102` (drip-306) was
merge-after-nits; this revision tightens the test coverage materially.
Ship.
