# drip-160 (2026-04-29)

8 fresh PR reviews across 5 repos:

- **sst/opencode**: #24869 (paste summary palette toggle), #24867 (sidebar load-more cache predicate)
- **openai/codex**: #20117 (`--permissions-profile` debug flag), #20111 (bounded bwrap probe)
- **BerriAI/litellm**: #26733 (short-ID MCP tool prefix), #26729 (negative cache for invalid virtual keys)
- **google-gemini/gemini-cli**: #26149 (`runtime.json` session sidecar)
- **block/goose**: #8870 (cumulative token usage in stream-json/json complete)

Verdict mix: 2 merge-as-is, 6 merge-after-nits, 0 needs-discussion, 0 request-changes.

Theme: "the right primitive in the wrong place leaks state through a side channel" — most of the merge-after-nits reviews fault either an inconsistent read/write split (opencode #24869 init vs gate; litellm #26729 hash twice), an under-cached hot-path rebuild (opencode #24867 root scan; litellm #26733 prefix dict), or a forward-compat asymmetry between producer and consumer (gemini-cli #26149 schema-version equality).
