# block/goose #9021 — feat(developer): add web_fetch tool for narrow URL fetching

- Head SHA: `2985dfe072028227178837346dfe8116a7e5f957`
- Diff: +363 / −1 across `crates/goose/src/agents/platform_extensions/developer/{mod.rs,web.rs}`, `documentation/docs/mcp/{developer-mcp.md,fetch-mcp.md}`

## Verdict: `request-changes`

## What it does

Adds a `web_fetch` tool to the built-in `developer` extension that does an HTTP GET against an `http(s)://` URL and returns either inline text/JSON (≤64 KiB), a JSON-validated body, or a path to a temp file (binary or oversized text). Wires the tool into `DeveloperClient` (`developer/mod.rs:30`, `:78`, `:157-167`, `:225-231`), updates the `developer-mcp.md` docs table with a `⚠️ Medium` risk annotation, and adds a "Built-in alternative" tip to `fetch-mcp.md` pointing users at the new tool when they don't need HTML→markdown conversion.

## Why this needs changes before landing

The implementation is clean Rust — `WebFetchTool::fetch` (`web.rs:43-92`) is straightforward, the wiremock-backed test suite at `web.rs:137-330` covers the inline-text / non-2xx / JSON-good / JSON-bad / binary-temp-file / oversized-text-spill / empty-URL / non-http-scheme paths, and the temp-file delivery via `tempfile::Builder.keep()` is the right primitive. But the **safety surface around outbound URL fetching** has two real gaps that a built-in `developer` tool — i.e., one that ships enabled by default with no per-host allowlist UI today — needs to close before shipping:

1. **No SSRF protection. `http(s)://` is not enough.** Accepting any `http://` or `https://` URL means a model that's been prompt-injected can hit `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (AWS IMDS), `http://localhost:8080/`, `http://[::1]:6443/api/v1/secrets`, or arbitrary RFC 1918 / link-local / loopback / `.internal`/`.local` hosts on the user's machine and corporate network. The wiremock tests use `server.uri()` which is `http://127.0.0.1:<port>` — the test suite is *itself* implicit proof that loopback works fine, which is exactly the SSRF surface. The PR's own `developer-mcp.md` row says "same network reach as `shell` + `curl`" — but `shell` requires user approval per call, while `web_fetch` is a model-callable tool with no approval gate visible in this diff. Mitigation options: (a) a built-in deny list for IMDS / loopback / RFC 1918 / link-local with an explicit `allow_local: bool` opt-in param, (b) explicit hostname allowlist in goose config, (c) requiring user approval per-host on first use. Pick at least one.
2. **No size cap on the response body.** `INLINE_BYTE_LIMIT = 64 * 1024` (`web.rs:35`) only governs *whether* the body goes inline vs. temp file — it does *not* cap how much bytes get read off the wire. `response.text().await` and `response.bytes().await` will buffer the entire response in memory; a malicious or misconfigured endpoint that streams 10 GiB will OOM the process. Set a `Content-Length` cap (reject early if header exceeds N), and use `response.bytes_stream()` with a running byte-counter that aborts the connection past the cap. Same applies to the binary path.

Two more issues that are smaller but worth fixing in this PR:

3. **No redirect policy — `reqwest::Client::builder()` defaults to following up to 10 redirects** (`web.rs:51-55`). That defeats the SSRF mitigation above: a model gets the host `evil.example` past your allowlist, then the server 302s to `http://169.254.169.254/...` and `reqwest` follows transparently. Either set `.redirect(reqwest::redirect::Policy::none())` and surface 3xx as the result, or implement a policy that re-runs the host check on each redirect target.
4. **Loss of HTTP status / headers / content-type signal.** A successful 200 returns just the body string with no way for the model to know the `Content-Type` (was that JSON or HTML?), the final URL after redirects, or response headers like `Cache-Control` / `Content-Disposition`. For a tool whose output is a model's view of the web, dropping all that metadata makes downstream reasoning brittle. Consider returning a small JSON envelope `{ status, final_url, content_type, body }` for the inline path, or at least `Content-Type` so JSON-vs-HTML disambiguation works.

## Smaller nits (not blockers individually)

- `INLINE_BYTE_LIMIT` checks `text.len()` (UTF-8 *byte* length), which is correct for the size-cap intent but the constant name `64 * 1024` reads as KiB without comment — a `// 64 KiB; chosen to keep model context bounded` would document the rationale.
- `error_result` (`web.rs:130-132`) wraps the message in a `Content::text` with priority `0.0` and `is_error: Some(true)`; consistent with the success path. Fine.
- `tempfile::Builder.keep()` (`web.rs:124`) intentionally leaks the temp file (renames it to a non-`O_TMPFILE` path that survives process exit) so the model can later `read_file` it. That's correct for the use case, but means `goose-web-*.{txt,json,bin}` files accumulate in `$TMPDIR` indefinitely with no GC story. Consider a startup sweep of older `goose-web-*` files (e.g., delete > 24h old) or document the manual cleanup.
- `User-Agent` is `goose/{CARGO_PKG_VERSION}` — good, lets servers attribute the traffic. Consider also setting `Accept-Encoding` so the body isn't always uncompressed (`reqwest` will handle decompression but you have to enable `gzip`/`brotli`/`deflate` cargo features).
- Error path for `error_result(&format!("Failed to fetch URL: {e}"))` (`web.rs:71`) leaks the `reqwest` error chain into the model's context, which on a connection-refused includes the resolved IP — minor info leak in the SSRF context above. After the SSRF guard lands, this is fine; until then, redact or summarize.
- The unrelated `fetch-mcp.md` "Built-in alternative" tip is a nice user-experience touch and is the right call — assuming the safety surface is closed first.

## Verdict justification

The Rust code is competent and well-tested in isolation; the issues are at the design surface, not the implementation. SSRF + unbounded body buffering on a default-enabled platform tool is a meaningful new attack surface in a project where prompt-injected models can already do a lot of damage. Once redirect policy, host filtering, and a body-size cap land — even as a v1 with conservative defaults that opt-in via params — this is `merge-after-nits`.

## Citations

- `crates/goose/src/agents/platform_extensions/developer/web.rs:43-92` — `WebFetchTool::fetch` core
- `crates/goose/src/agents/platform_extensions/developer/web.rs:35` — `INLINE_BYTE_LIMIT = 64 * 1024` (size *of inlining decision*, not response cap)
- `crates/goose/src/agents/platform_extensions/developer/web.rs:51-55` — `reqwest::Client::builder()` lacks `.redirect(...)` policy
- `crates/goose/src/agents/platform_extensions/developer/web.rs:73-91` — `response.text()` / `response.bytes()` reads full body, no streaming cap
- `crates/goose/src/agents/platform_extensions/developer/web.rs:124` — `tempfile.keep()` permanent on disk
- `crates/goose/src/agents/platform_extensions/developer/mod.rs:157-167` — tool registration with `ToolAnnotations` (read-only/idempotent flags)
- `documentation/docs/mcp/developer-mcp.md:191` — `⚠️ Medium` risk annotation acknowledges network reach
- `crates/goose/src/agents/platform_extensions/developer/web.rs:309-321` — `fetch_rejects_non_http_scheme` test only covers `file://`, not RFC 1918 / loopback / IMDS
