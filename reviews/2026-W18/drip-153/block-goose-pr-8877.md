# block/goose #8877 — render mcp apps inline in goose2

- PR: https://github.com/block/goose/pull/8877
- Head SHA: `5d37fdd91653b6446ba62c3eda2d801f8d7be1a5`
- Files (35+): notable: `crates/goose/src/acp/mcp_app_proxy.rs` (new, 229 LoC), `crates/goose/src/acp/templates/mcp_app_proxy.html`, `crates/goose-cli/src/cli.rs`, `crates/goose-sdk/src/custom_requests.rs`, `crates/goose/acp-meta.json`, `crates/goose/acp-schema.json`, `ui/goose2/src/features/chat/ui/McpAppView.tsx` and tests, `ui/goose2/src/shared/api/acpToolCallContent.ts`.

## Citations

- `crates/goose-cli/src/cli.rs:1086-1091` — `handle_serve_command` reads `GOOSE_SERVER__SECRET_KEY` from env (default `"goose-acp-local"`) and threads it to `create_router(server, secret_key)`. Default value is a constant string, which means a CLI invocation without the env var has a *known* secret.
- `crates/goose/src/acp/mcp_app_proxy.rs:14-16` — guest-store bounds: `GUEST_HTML_TTL_SECS: u64 = 300`, `GUEST_HTML_MAX_ENTRIES: usize = 64`. Memory footprint is bounded; entries TTL out at 5min.
- `mcp_app_proxy.rs:276`, `:308`, `:345` — three handlers (`proxy GET`, `store POST`, `guest GET`) all gate on `params.secret != state.secret_key { return UNAUTHORIZED }`. The check is pure string equality, no constant-time compare.
- `mcp_app_proxy.rs:317-322` — store-time eviction: TTL prune `cutoff = Instant::now() - Duration::from_secs(GUEST_HTML_TTL_SECS)` then capacity prune to `GUEST_HTML_MAX_ENTRIES`. The capacity prune evicts arbitrary entries (HashMap iteration order), not LRU/oldest.
- `mcp_app_proxy.rs:631` — guest iframe `sandbox` attribute: `'allow-scripts allow-same-origin allow-forms'`. `allow-scripts + allow-same-origin` is the documented *unsafe* combo for untrusted content (the iframe can use scripts to access its parent's same-origin frames if any cousins share the origin, and to escape the sandbox via `parent.location`).
- `acp/server.rs` — `AcpServer` now stores `secret_key: String` and exposes `secret_key()` getter. `goose_serve.rs:841` (Tauri host) generates per-process `let secret_key = format!("goose2-{}", uuid::Uuid::new_v4().simple())` and passes via `.env("GOOSE_SERVER__SECRET_KEY", &secret_key)` — i.e. the desktop host *does* mint a fresh per-launch secret; only the bare CLI defaults to the static string.
- `crates/goose-sdk/src/custom_requests.rs:78-99` — new RPC method `_goose/tool/call` with `GooseToolCallRequest { session_id, name, arguments }` / `GooseToolCallResponse { content, structured_content, is_error, _meta }`. The host uses this to route tool invocations *from* the embedded MCP app *back* into the goose extension namespace.
- `acp-meta.json:23-27` and `acp-schema.json:107-148` — schema entries match the SDK exactly (`isError` required, `content` defaults to `[]`, `structuredContent` and `_meta` optional). Generation pipeline is consistent.
- `ui/goose2/src/shared/api/acpToolCallContent.ts` (new) — content extraction helper feeds `McpAppView.tsx`'s `useMemo` hook computing `sandbox` (line 1107) and `renderableDocument` gating render at line 1234.

## Verdict

`request-changes`

## Reasoning

The architecture here — payload→inline-app pipeline established by an earlier PR (#8632) is now wired through to a real iframe-host with bidirectional MCP tool routing — is the right shape for the feature. Bounding the guest-HTML store at 64 entries / 300s TTL is the right discipline. The Tauri host's per-process UUID secret is correctly minted (`goose_serve.rs:841`). The new `_goose/tool/call` RPC and its schema-meta-SDK triple stay consistent. None of this is a "rewrite the PR" critique.

But there are three issues I think need to be addressed before merge, two of them security-shaped:

**1. CLI default secret is static.** `cli.rs:1086`'s `unwrap_or_else(|_| "goose-acp-local".into())` means anyone running `goose serve` from the bare CLI without setting `GOOSE_SERVER__SECRET_KEY` is sharing a fixed secret with every other such launch on every other machine. The proxy's only defense against unauthorized iframe-store-and-load is the secret check at lines 276/308/345. With a known default, a malicious local process that can reach `127.0.0.1:<port>` can `POST /store` arbitrary HTML and then `GET /guest?secret=goose-acp-local&nonce=<theirs>` to render it inside the goose UI's origin — which has script execution and `allow-same-origin`. This is a local-port escalation surface, real but not catastrophic; still, the CLI should mint a fresh UUID like `goose_serve.rs` does and *log* it so power users can plumb it explicitly. I would not ship this with `"goose-acp-local"` as the production default.

**2. Constant-time compare for secret equality.** `params.secret != state.secret_key` (lines 276, 308, 345) is plain `!=` on `String`. For the random UUID case this is fine — there's no oracle to attack. For the static-default case it doesn't matter either. But once secrets become long-lived per-installation values (which the env-var override invites), `subtle::ConstantTimeEq` or `constant_time_eq` is cheap and removes the foot-gun.

**3. Sandbox + `allow-same-origin` is the unsafe combo.** `mcp_app_proxy.html:631` sets `sandbox='allow-scripts allow-same-origin allow-forms'` on the guest iframe. MDN and the WHATWG spec are explicit: "if both `allow-scripts` and `allow-same-origin` are set, the iframe can remove the sandbox attribute from itself." This is acceptable when the iframe content is *fully trusted*, but the whole point of this proxy is that the content comes from an MCP app payload (i.e., third-party). Either drop `allow-same-origin` (and use `postMessage` for all host↔guest comms, which the rest of the file is doing already with `ui/notifications/sandbox-*-ready` events at lines 666 and 701) or document why `allow-same-origin` is required and pin a CSP. I lean toward "drop it" — the message-passing infrastructure is already there.

**Smaller issues for the same pass:**

- The capacity-cap eviction (`mcp_app_proxy.rs:320`) evicts in HashMap iteration order rather than LRU/oldest. With TTL pruning happening first this is mostly cosmetic, but a burst of 65 stores within a single millisecond will evict a uniformly-random entry — possibly the one the user is about to load. Switch to a small `BTreeMap<Instant, key>` index or `linked_hash_map` if predictability matters.
- The PR's 35-file footprint and 100K+ diff make it hard to do a single-pass review of the host↔guest message contract. Splitting the schema/SDK/server/UI changes into 2-3 PRs would have made each easier to land. (Not a blocker, just an observation.)

Address points 1-3 (especially 1 and 3) and I'll switch to merge-after-nits. The feature is good; the defaults aren't safe yet.
