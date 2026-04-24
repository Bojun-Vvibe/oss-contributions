---
pr_number: 15774
repo: ollama/ollama
head_sha: 0159e10a0eafcbd615a2767f5dc687ae77fcf199
verdict: merge-after-nits
date: 2026-04-25
---

# ollama/ollama#15774 — harden Qwen-family tool payload rendering + Qwen 3.5 truncation

**What changed.** 9 files / +618 / −17. Two largely-orthogonal fixes shipped together.

(1) Renderer hardening across `model/renderers/qwen35.go`, `qwen3coder.go`, `qwen3vl.go`. New `escapeQwenXMLText(s string) string` (qwen3coder.go line 62) handles `&`, `<`, `>` with an idempotent strategy: `existingEntityLength` (line 86) detects already-escaped `&lt;` / `&gt;` / `&amp;` / `&quot;` / `&apos;` plus numeric `&#…;` / `&#x…;` and skips re-escape. Tool-response payloads (`qwen35.go` line 176, `<tool_response>` insertion) and tool-call argument payloads now go through it. New `marshalToolArgument` (line 130) uses a `json.Encoder` with `SetEscapeHTML(false)` so embedded `<`/`>` in tool args aren't double-encoded by encoding/json's default.

(2) `server/prompt.go` Qwen 3.5 truncation: assistant messages with `ToolCalls` plus their immediately-following contiguous `tool` messages are treated as one atomic unit. Either the whole block survives or it's dropped — orphaned tool responses no longer survive past their parent assistant tool-call.

**Why it matters.** Untrusted tool-response content (e.g. shell output, web fetch) reaching the model with a literal `</tool_response>` substring is a classic prompt-injection / framing-confusion vector. The truncation fix prevents the model from seeing tool replies whose call context has been evicted, which historically produced "hallucinated tool memory" failure modes.

**Concerns.**
1. **`existingEntityLength` returns `end + 1` for the named entities** (qwen3coder.go line 100), where `end := strings.IndexByte(s, ';')`. If `&` is the last byte of `s`, `end == -1` and we fall through to the generic `&amp;` branch — correct. Edge case: `&lt` with no semicolon at end-of-string. `end == -1` → returns 0 → escapes the `&` only → output is `&amp;lt`. Acceptable but the test for "preserves existing entities" doesn't cover the truncated-entity-at-EOF case. Add `&lt` (no semi) → expect `&amp;lt`.
2. **`end > 10` cutoff** (line 92) is a heuristic for max entity length. Numeric entities like `&#1234567;` (7-digit decimal) are invalid Unicode (>U+10FFFF) so dropping them is fine, but the rationale should be commented. As-written it looks like a magic number.
3. **`marshalToolArgument`** trims a trailing `\n` from `json.Encoder` output. Correct (encoder appends one), but consider `bytes.TrimSuffix` on the buffer before allocating the string — saves one alloc on hot path. Minor.
4. **Truncation atomicity in `server/prompt.go`** (not visible in the 250-line diff slice but described in the body) — the unit-of-truncation widening is the right call. Ensure the test covers the "block too large to fit even alone" case: it should be dropped, not partially retained.
5. **Whitespace preservation test** (`TestQwen35RendererPreservesToolResponseWhitespace`, line 358) asserts EXACT bytes including leading/trailing newlines around the `<tool_response>` payload. That's a strong contract that locks in current renderer behavior. Good defensive test, but it will be brittle if the surrounding template ever adds a space; document the invariant in the renderer source.
6. **Two fixes in one PR.** The renderer hardening and the truncation fix are independent. Splitting would simplify review and bisect; the body itself separates them.

Security-flavored fix shipping with a real bug fix. Merge after the entity-EOF test (#1) lands and ideally split into two commits.
