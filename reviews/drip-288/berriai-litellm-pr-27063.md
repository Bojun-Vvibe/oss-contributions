# BerriAI/litellm PR #27063 — Benchmark: Semantic Code Search (claude-context MCP) vs. Grep for Codebase Q&A

- Head SHA: `6413ddf040fbb12d21385989d7d5c6bd8e80ef41`
- URL: https://github.com/BerriAI/litellm/pull/27063
- Size: +1374 / -0, 3 files (`benchmark_mcp_vs_grep.py`, `benchmark_report.md`, `benchmark_results.json`)
- Verdict: **needs-discussion**

## What changes

A standalone benchmark comparing two retrieval strategies against the
LiteLLM codebase: (a) semantic search via `all-MiniLM-L6-v2` +
Milvus Lite (mirroring `zilliztech/claude-context`), and (b) ripgrep
keyword search with file ranking and snippet extraction. Both feed
into `claude-sonnet-4-20250514` for answer generation, scored by the
same model as judge across 4 dimensions × 5 points = 20 max. Headline
result: grep wins 19.70 vs 18.30 / 20 (5 wins, 4 ties, 1 loss for
semantic) with 3.2× more context tokens and 6.5× slower per-search
latency. Files added at repo root: `benchmark_mcp_vs_grep.py` (849
LOC), `benchmark_report.md` (133 LOC), `benchmark_results.json` (392
LOC).

## What looks good

- The methodology is honest about its limitations in the PR body:
  weaker embedding model than claude-context default, no BM25 hybrid,
  pre-defined grep terms (best-case for grep). Those caveats are
  mirrored in `benchmark_report.md` (lines ~123-130). That kind of
  self-disclosure is rare in vendored benchmarks.
- The script is reproducible: `BENCHMARK_QUERIES` (line ~50 of
  `benchmark_mcp_vs_grep.py`) is a single source-of-truth dict with
  query, ground truth, key files, and grep terms — anyone can re-run
  and diff.
- `count_tokens_approx` (line ~60) is explicitly approximate (words ×
  1.3) and `MAX_CONTEXT_TOKENS = 12000` is a hard cap, so both
  approaches compete in the same budget. That's the right control.
- The judge-scoring rubric (4 dims × 5 points) is consistent across
  approaches and the per-query JSON in `benchmark_results.json`
  preserves the raw answers + scores so reviewers can sanity-check
  the judge's calls.

## Why needs-discussion (not merge)

This is fundamentally a *research artifact*, not product code, and it
lands at the repo root with no integration into the existing
benchmark/eval surface. Concerns:

1. **Repo placement.** `benchmark_mcp_vs_grep.py` at the top level
   sits next to `setup.py`, `pyproject.toml`, etc. A throwaway
   benchmark script does not belong there. The project already has
   `litellm/tests/`, `litellm/proxy/tests/`, and (separately)
   `cookbook/`. A 1374-line one-shot benchmark with hard-coded
   `/workspace/litellm` paths (line ~25) belongs in `cookbook/` or
   a sibling repo, not next to the package metadata.
2. **Hard-coded host paths.** `CODEBASE_DIR = "/workspace/litellm"`
   and `MILVUS_DB_PATH = "/workspace/benchmark_milvus.db"`
   (`benchmark_mcp_vs_grep.py` line ~24) are baked into the script.
   The script will not run for any contributor without editing those
   constants. At minimum these should be CLI args / env vars.
3. **API key in a benchmark committed to a public repo.** Line ~22
   reads `os.environ["ANTHROPIC_API_KEY"]` at *import time* (KeyError
   if missing). That breaks `python -c "import benchmark_mcp_vs_grep"`,
   any IDE that walks the repo, and any test discovery that imports
   top-level modules. Move the key read inside `run_benchmark()`.
4. **`benchmark_results.json` (392 lines) committed alongside.** Raw
   per-query answers including LLM-generated text — this is fine for
   one snapshot but will rot the moment the codebase or judge model
   shifts. There's no script to regenerate it deterministically (it
   depends on Claude's non-deterministic output). Either drop it from
   the commit or write a `make benchmark` that regenerates it.
5. **Conclusion is overclaimed.** "Grep wins on answer quality" with
   n=10 queries, single judge model (= same model as the answerer,
   which is a known judge-bias trap), pre-cherrypicked grep terms
   (best-case for grep, worst-case for semantic which had to derive
   them), and a deliberately-weakened embedding model is *not* a
   conclusion that should be embedded into the project's docs. The
   PR body and `benchmark_report.md` do disclose these, but the
   summary tables don't.

## Risk

Low for the runtime — the script is dead code that's never imported
anywhere. High for *what shipping it implies*: this PR effectively
puts a "we recommend grep over semantic search" marker into the repo,
which is not a position the maintainers may want to defend in
issues / Slack threads.

## Recommendation

Move the script to `cookbook/benchmarks/code_search/`, drop
`benchmark_results.json` from the commit (or generate it from the
script), make `CODEBASE_DIR`/`ANTHROPIC_API_KEY` CLI-arg-driven, and
soften the report's headline from "Winner: Grep" to "Result snapshot
for n=10, MiniLM-L6-v2, Claude-as-judge". Then it's a clear
merge-after-nits.
