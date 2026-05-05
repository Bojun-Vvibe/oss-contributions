# block/goose PR #9021 — feat(developer): add web_fetch tool for narrow URL fetching

- URL: https://github.com/block/goose/pull/9021
- Head SHA: `2985dfe072028227178837346dfe8116a7e5f957`
- Author: (related to #7824, supersedes-rationale of #7746)
- Size: +363 / -1 (4 files: `developer/mod.rs`, new `developer/web.rs`, two doc files)

## Verdict
`merge-after-nits`

## Rationale

The capability story is the correct one. Today the only built-in URL-fetch path runs through the
`computercontroller` extension, which also exposes screen control (`computer_control`), arbitrary
script execution (`automation_script`), and document parsers — all granted to the agent in one
bundle. Headless ACP integrations that just need to read a URL end up with an agent that can also
type into the user's active window, which has caused real reported incidents. This PR's argument
that adding `web_fetch` to `developer` is *strictly less capable* than `developer`'s existing
`shell` (which can already run `curl`) holds: the new tool at `developer/web.rs` enforces an
http(s)-only schema check at `:103-108`, a 30-second `REQUEST_TIMEOUT` at `:78`, an LLM-friendly
`Accept: text/markdown, text/html, application/json, */*` header at `:118` (the headline perf win
from the bench table — Mintlify and llms-aware docs serve a ~33× smaller markdown payload, dropping
the `docs.diversion.dev` case from 415K tokens / 39.7s to 17K / 6.7s), and a 64KB
`INLINE_BYTE_LIMIT` at `:73` that spills large bodies to a temp file rather than blowing the
context window. The `SaveAsFormat` enum at `:53-64` (`Text` / `Json` / `Binary`) has the right
ergonomic split — `Json` validates `serde_json::from_str::<Value>(&text)` at `:140` before
returning, so the model can branch on parse success without a separate validation tool call.

The wiring at `developer/mod.rs` is consistent with established conventions — the new
`web_fetch_tool: Arc<WebFetchTool>` field is initialized in `DeveloperClient::new()` (`:78`), the
tool is registered in `get_tools()` (`:157-168`) with `ToolAnnotations::from_raw(Some("WebFetch"),
Some(true), Some(false), Some(true), Some(true))` (read-only + idempotent + open-world for network,
which is the right tri-state for a network read), and dispatched through `call_tool` at `:225-231`
with the same `parse_args` + structured-error pattern the other developer tools use. The
`developer_tools_are_flat` test at `mod.rs:259` is updated to expect the new tool in the canonical
order, and the 7 wiremock-backed unit tests in `developer/web.rs` cover inline text, large-body
spill, JSON good-and-bad, binary spill, non-2xx, empty URL, and non-http schemes — the exact
boundary set worth pinning.

Nits to land before merge: (1) `WebFetchTool::new()` at `web.rs:84-90` falls back to
`reqwest::Client::new()` if `Client::builder()...build()` fails — that fallback silently *drops*
both the `goose/<version>` user-agent and the 30s timeout, so a misconfigured TLS root or
similar build failure would silently turn the tool into "no UA, no timeout" behavior that's
strictly worse than failing loudly; the fallback should `panic!` or `tracing::error!` and return
a tool that always-errors. (2) The `fetch()` path does *not* enforce a max response size *before*
buffering — `response.bytes().await` at `:153` will fully buffer the entire body into memory before
the `INLINE_BYTE_LIMIT` check decides whether to spill, so a server returning a 4GB body OOMs the
goose process; switching to `response.bytes_stream()` with a streaming write to the temp file +
running byte counter would close that gap. (3) The Wikipedia row in the bench table (+7.5% tokens
vs vanilla) is honest but worth a doc note in `documentation/docs/mcp/developer-mcp.md`: for
HTML pages over 64KB the spill-to-file requires a follow-up `shell` call to grep, so for that
workload `shell + curl | grep` is still better — calling that out preempts a "you said this was
faster" issue. (4) The PR description is excellent (n=10 benchmarks, 3 URLs, model-substitution
rate 10/10) but `documentation/docs/mcp/fetch-mcp.md` should explicitly cross-link to the new tool
and articulate "use built-in `web_fetch` for raw retrieval, use `mcp-server-fetch` for HTML→markdown
conversion of arbitrary sites", since that's the real decision boundary now.
