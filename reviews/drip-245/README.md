# drip-245 (2026-05-02)

8 fresh PR reviews across 6 OSS AI-coding-agent repos. Local artifacts only;
nothing was submitted upstream. Verdict mix:

- merge-as-is: 4 (sst/opencode#25291, sst/opencode#25290, openai/codex#20627,
  block/goose#8947)
- merge-after-nits: 3 (sst/opencode#25303, google-gemini/gemini-cli#26332,
  block/goose#8948)
- request-changes: 0
- needs-discussion: 1 (QwenLM/qwen-code#3388)

## Themes

- **OpenAPI/SDK contract drift** (opencode #25291) — typed parity bridges
  catch class of bug that "schema present?" parity tests miss.
- **Cross-model conversation replay** (opencode #25303) — signed reasoning
  metadata can't be replayed under a different producing model; right fix
  is to demote to plain text.
- **mDNS service-name collisions** (opencode #25290) — fixed-prefix mDNS
  names collide on the same LAN; deriving prefix from configured domain
  is the right scope.
- **Supply-chain hygiene** (codex #20627) — pinning `dtolnay/rust-toolchain`
  to a commit + tracked-removal rationale on RUSTSEC ignores is what good
  cargo-deny config looks like.
- **ACP mode propagation** (gemini-cli #26332) — event-bus wiring of
  approval-mode changes to both the system prompt and the ACP host;
  listener-leak and string-payload concerns are non-blocking.
- **Native provider addition** (goose #8948) — additive native Rust
  provider for NVIDIA NIM following established `OpenAiCompatibleProvider`
  pattern; missing tests is the only nit.
- **Quota-aware UI feature opt-out** (goose #8947) — per-tool-call
  `complete_fast` upgrade is real LLM quota cost; clean opt-out via env-var
  mirrors `GOOSE_DISABLE_SESSION_NAMING`.
- **LLM-evaluated hooks as a security surface** (qwen-code #3388) — the
  feature works but the cost model, fail-open default, and prompt-injection
  surface deserve explicit design discussion before merge.

## Files

- `sst-opencode-25291.md` — fix(httpapi): preserve OpenAPI parameter parity
- `sst-opencode-25303.md` — fix: bedrock reasoning issue
- `sst-opencode-25290.md` — fix(mdns): derive service name from domain
- `openai-codex-20627.md` — fix: cargo deny
- `google-gemini-gemini-cli-26332.md` — fix(acp): resolve agent mode disconnect
- `block-goose-8948.md` — feat: add NVIDIA NIM as a native Rust provider
- `block-goose-8947.md` — feat(acp): GOOSE_DISABLE_TOOL_CALL_SUMMARY
- `qwenlm-qwen-code-3388.md` — feat(hooks): prompt hook type with LLM eval
