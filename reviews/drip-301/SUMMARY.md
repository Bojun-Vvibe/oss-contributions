# drip-301 (2026-05-03)

8 fresh PR reviews across 5 carriers (sst/opencode, openai/codex, BerriAI/litellm, block/goose, QwenLM/qwen-code).

| Repo | PR | Verdict | File |
|---|---|---|---|
| sst/opencode | #25554 | merge-after-nits | `sst-opencode-pr-25554.md` |
| sst/opencode | #25544 | needs-discussion | `sst-opencode-pr-25544.md` |
| sst/opencode | #25537 | merge-after-nits | `sst-opencode-pr-25537.md` |
| openai/codex | #20849 | merge-after-nits | `openai-codex-pr-20849.md` |
| BerriAI/litellm | #27076 | merge-after-nits | `berriai-litellm-pr-27076.md` |
| block/goose | #8973 | merge-as-is | `block-goose-pr-8973.md` |
| block/goose | #8972 | needs-discussion | `block-goose-pr-8972.md` |
| QwenLM/qwen-code | #3801 | merge-after-nits | `qwenlm-qwen-code-pr-3801.md` |

## Verdict mix

- merge-as-is: 1 (block/goose #8973 — config-only LM Studio improvements)
- merge-after-nits: 5 (cluster of small surgical fixes / well-scoped refactors)
- request-changes: 0
- needs-discussion: 2 (sst/opencode #25544 is a draft design sketch by the author's own framing; block/goose #8972 raises a payment-rail-MCP policy question for maintainer judgment)

## Themes

- **Effect-native flatten cluster** (sst/opencode #25537, #25544) continues the kitlangton series — handlers shedding the `Effect.promise(async)` outer wraps and HttpApi adopting typed errors at the service boundary. The typed-error sketch in particular sets up the migration off the `mapNotFound` shim used today.
- **Config / path resolution correctness** (openai/codex #20849) locks in a three-leg matrix: `$CODEX_HOME/config.toml` paths anchor to codex_home, profile paths anchor to the config file dir, `-c` overrides anchor to cwd. The fact that this needed a regression test suite at all is interesting — suggests recent path-handling changes were close to silently breaking one of the three cases.
- **Defensive coercion bug fixes** (BerriAI/litellm #27076) — `int("litellm_error")` crashing the fallback chain is the kind of latent bug that only surfaces under specific upstream provider error shapes; the fix's test reproducing the exact `'litellm_error'` string is the right approach.
- **Cross-cutting "where do background tasks list" question** (QwenLM/qwen-code #3801) — `/tasks` was silently dropping the monitor entry type that landed in #3684/#3791. Phase B closure with a four-state test matrix (running / completed / auto-stop / failed) and a justified "keep + scope hint to interactive" design decision over straight deletion.
- **Docs / config additions** (block/goose #8973 LM Studio dynamic models, #8972 Voidly Pay MCP docs) — one is a clean factual config improvement, the other surfaces a real policy question about third-party payment-rail MCPs that deserves maintainer discussion before merge.
