---
pr: 2549
repo: charmbracelet/crush
title: "feat: replace standard JSON library with bytedance/sonic for better performance"
url: https://github.com/charmbracelet/crush/pull/2549
head_sha: e2ceb14621
author: BrunoKrugel
verdict: request-changes
date: 2026-04-27
---

# charmbracelet/crush #2549 — Replace `encoding/json` with `bytedance/sonic`

Drop-in swap of the stdlib `encoding/json` package for `github.com/bytedance/sonic v1.15.0` across 44 files. +208/−180. Roughly 144 call-site substitutions of `json.Marshal`/`json.Unmarshal` → `sonic.Marshal`/`sonic.Unmarshal`. Also adds three transitive deps (`bytedance/gopkg`, `bytedance/sonic/loader`, `cloudwego/base64x`, `twitchyliquid64/golang-asm`) and amends the security workflow's allow-license list.

## Why this is being proposed

Sonic is a JIT-accelerated JSON encoder/decoder that is faster than stdlib for medium-to-large payloads. The PR title pitches "better performance." For an interactive TUI agent the JSON-shaped hot paths are: provider streaming (`internal/agent/hyper/provider.go`), tool invocation (`internal/agent/tools/*`), session/event metadata, and config marshaling.

## Concerns

### 1. No benchmarks. Author claims a perf win without a single before/after measurement.

Sonic's docs are explicit that it shines on **large** documents and amd64. Crush's per-call payloads are mostly small (tool-call args, config blobs, event frames). At small-payload sizes sonic is often *slower* than `encoding/json` because of its assembly setup cost. There is no `BenchmarkX_Stdlib` vs. `BenchmarkX_Sonic` in the diff. Without that, this PR is asking maintainers to take on a substantial new dependency on faith.

Actionable: add `go test -bench` numbers for at least:
- One representative `provider.go` stream parse cycle.
- One `bash_test.go`-shaped `Marshal` / `Unmarshal` round-trip.
- A cold-start measurement (first call latency) — sonic's JIT compiles on first use.

### 2. Sonic is a non-trivial portability and security hit.

- **arm64 fallback.** Sonic uses generated amd64 assembly. On other architectures it falls back to `encoding/json` via `sonic/loader`. macOS arm64 (Apple Silicon, the literal target audience for a TUI written in Go in 2026) gets the fallback path — no perf win at all for that audience. Worth saying out loud in the PR body.
- **Dependency surface.** Adds `bytedance/gopkg`, `cloudwego/base64x`, `twitchyliquid64/golang-asm`. The last is a fork of golang's internal asm package. Each is one more thing the security audit (`.github/workflows/security.yml`) has to keep current.
- **License allow-list change.** `security.yml:92` adds `LicenseRef-github-NOASSERTION` to the allow-licenses list. That's GitHub's catch-all "we couldn't classify the license" tag — a regression in license clarity. Which dependency triggered this? The PR body needs to call it out by name and link to the upstream LICENSE so reviewers can independently classify it. **Adding `NOASSERTION` to a security gate is a no-go without that justification.**

### 3. Drop-in swap loses behavior compatibility in subtle places.

Sonic is documented to be 99% compatible with `encoding/json`, but the gaps that exist matter for an agent codebase:
- `json.RawMessage` semantics with sonic vary in edge cases (whitespace preservation, error messages).
- Number handling: sonic treats `json.Number` differently when chained.
- Error messages embedded in user-visible UI (`util.InfoMsg`) will now read with sonic's wording, which changes test assertions and user error messages. I see no audit of error-string-asserting tests in the diff.

I noticed the test files `bash_test.go` and `projects_test.go` were also converted to `sonic.Unmarshal`. **Tests should keep using `encoding/json` as the reference encoder and assert that the production code's sonic output round-trips through stdlib.** Otherwise you're proving "sonic agrees with sonic," which is tautological. Convert tests back.

### 4. Streaming decoder hot path needs special attention.

`internal/agent/hyper/provider.go` change:
```go
-		if err := json.Unmarshal(dataBuf.Bytes(), &part); err != nil {
+		if err := sonic.Unmarshal(dataBuf.Bytes(), &part); err != nil {
```
This is on a per-SSE-frame path. Sonic's `Unmarshal` JIT-compiles a decoder on first call for each unique type, so the *first* SSE frame for a given stream-part type will pay the JIT cost. For agents that open and close streams frequently (Crush's pattern), the warm path may not be reached. Add a benchmark.

Also: I don't see a switch to `sonic/decoder.NewStreamDecoder` for genuinely streamed input. `sonic.Unmarshal` on a fully buffered slice is the same shape as before; if the goal is performance, the streaming variant is the lever here.

### 5. The diff swept tests, examples, and tooling all at once.

44-file blast radius for a "swap one std import" is hard to review. Suggested split:
- PR 1: introduce `internal/json` shim that wraps either stdlib or sonic via build tag, plus a single benchmark test demonstrating measurable win.
- PR 2: swap call sites incrementally, hot-path first.
- PR 3: only after (1) and (2) land, decide whether tests should swap.

### 6. License gate change must be reverted or argued.

`security.yml` change is a separate concern and arguably should be its own PR with a CHANGELOG / SECURITY.md entry naming the dependency, the actual upstream license, and why `NOASSERTION` is acceptable. As written, future contributors won't know why the gate was loosened.

## Smaller things

- `internal/cmd/projects_test.go` — convert tests back to stdlib (see above).
- `internal/agent/tools/mcp/tools.go` — MCP tool args come from outside processes; sonic's parser is more permissive about some malformed JSON than stdlib (e.g. trailing data). For a tool-execution boundary, you want *less* permissive, not more.
- `internal/cmd/schema.go` — generated JSON schema output may differ in field ordering / whitespace. Snapshot tests, if any, will need rebaselining. Worth confirming.
- The PR title says "for better performance" — soften to "introduce sonic for hot-path JSON" or similar until benchmarks back the claim.

## Verdict

**request-changes.** Three blocking items:

1. **Benchmarks before/after** for at least 2-3 representative call sites (incl. a cold-start number).
2. **Justify the `NOASSERTION` license-list entry** — name the dep, link the upstream LICENSE, explain why the catch-all is acceptable here, or remove that hunk.
3. **Revert the test-file conversions.** Tests should keep stdlib as the reference encoder.

Nice-to-have: split the patch as suggested in §5 so the perf claim and the dep change land separately from the call-site sweep. The underlying idea (use sonic for hot JSON paths) is reasonable; the execution needs evidence and a smaller blast radius.

## Banned-string scan

None.
