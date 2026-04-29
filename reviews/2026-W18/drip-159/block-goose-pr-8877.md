# Review: block/goose#8877 — render mcp apps inline in goose2

- **Author**: aharvard
- **Head SHA**: `5d37fdd91653b6446ba62c3eda2d801f8d7be1a5`
- **Diff**: +2120 / −52 across 39 files
- **Tracking**: #8591
- **Draft**: yes

## What it does

Turns goose2 into a real inline MCP-Apps host. Pre-PR (after #8632) the durable MCP-app payload was carried through live updates and replay, but the UI only displayed it as data. This PR adds:

1. **A local authenticated proxy + sandboxed iframe for app HTML.** `crates/goose/src/acp/mcp_app_proxy.rs` (+229) is a brand-new axum router with three endpoints: a `GET` proxy serving `templates/mcp_app_proxy.html` (a 141-line CSP-shaped wrapper) gated by a `secret` query param plus configurable CSP allowlists (`connect_domains`, `resource_domains`, `frame_domains`, `base_uri_domains`, `script_domains`); a `GET` for guest HTML keyed by `(secret, nonce)`; and a `POST` to store guest HTML keyed by `secret` with TTL=300s and max-entries=64 (`GUEST_HTML_TTL_SECS`, `GUEST_HTML_MAX_ENTRIES`).
2. **A new ACP method `_goose/tool/call`** (`crates/goose-sdk/src/custom_requests.rs:77-99`) with `GooseToolCallRequest { session_id, name, arguments }` and `GooseToolCallResponse { content, structured_content, is_error, _meta }`. Mirrored in `acp-meta.json` and `acp-schema.json` (+59). This is the channel for app→Goose nested tool calls without going through chat.
3. **CLI plumbing for the proxy secret**: `crates/goose-cli/src/cli.rs:1086-1091` reads `GOOSE_SERVER__SECRET_KEY` (default `"goose-acp-local"`) and threads it into `create_router(server, secret_key)`.
4. **Goose2 frontend host bridge** (`ui/goose2/src/features/chat/ui/McpAppView.tsx` +412 / −10): mounts the app inside an `AppRenderer` sandbox, fetches `httpBaseUrl + secretKey` via `get_goose_serve_host_info()` (new Tauri command, `ui/goose2/src-tauri/src/services/acp/goose_serve.rs:14`), provides hostContext (theme, locale, display mode, timezone, container dimensions), routes app→host requests to:
   - `ui/message` → `onSendMessage(text)` → normal chat send path
   - `ui/get-context` → return current `hostContext`
   - `tools/call` → `_goose/tool/call(sessionId, extension__toolName, args)`
   - `resources/read` → ACP resource-read for the same session/extension
   - `ui/open-link` → native opener
   - `ui/notifications/size-changed` → resize inline frame + sticky autoscroll
5. **Replay / MessageBubble / MessageTimeline integration** (+57/+176/+64) — preserve MCP app payloads through both live updates and replay; attach to the correct assistant message; respect app's `border` preference.
6. **Tests**: `McpAppView.test.tsx` (+238), `ChatView.mcpApp.test.tsx` (+122), `MessageBubble.mcpApp.test.tsx` (+28/-6), `acpNotificationHandler.test.ts` (+54), `MessageBubble.test.tsx` (+20). Covers the renderer, the chat-view integration, the bubble path, and the ACP notification handler.

## Concerns (DRAFT — author lists own follow-ups)

1. **HTTP (not HTTPS) local proxy with a long-lived shared secret.** `GOOSE_SERVER__SECRET_KEY` defaults to the literal string `"goose-acp-local"` (`cli.rs:1088`). If the user never sets it, every goose2 install on every machine has the same secret. The `GET /proxy?secret=goose-acp-local` endpoint is bound to localhost (presumably — not shown in the new file's address-bind, which inherits from `handle_serve_command`'s socket), but if the host is later reused on a multi-user box (shared dev VM, codespace), any unprivileged process can fetch app HTML and exfiltrate it. The PR's own scope-statement acknowledges "explore whether the local proxy should move to HTTPS and/or a stronger origin model for long-term hardening" — that should be in scope before this leaves DRAFT, not after, because the path is structurally vulnerable.

2. **Guest HTML store is unbounded-growth in pathological flows.** `GUEST_HTML_MAX_ENTRIES=64` is the cap, but eviction policy isn't shown in the visible diff (need to check the full `mcp_app_proxy.rs`). If it's FIFO, a malicious or buggy app can flush legitimate guest HTML out of the store within one tool call. If it's LRU, the same attack is harder but still possible. A test that fills the store past the cap and asserts the eviction shape would pin it.

3. **`extension__toolName` namespacing in the `_goose/tool/call` payload.** The frontend constructs `extension__toolName` (double-underscore concat) and sends it to `_goose/tool/call` (`McpAppView.tsx`, per the README). The server side dispatches by name. If two extensions register tools with double-underscore-bearing names (`weather__forecast` and a hypothetical `weather__forecast` from a second extension), the routing collides silently. The right fix is structured `{ extension, tool }` fields, not a flat string. The added `GooseToolCallRequest` shape has only `name`, so this is a wire-protocol decision that's hard to walk back.

4. **i18n updated for `en` and `es` only** (`ui/goose2/src/shared/i18n/locales/{en,es}/chat.json` +6/−2 each). `de`, `fr`, `ja`, `pt-BR`, `zh-CN` (the rest of goose2's i18n contract per recent drips) are not updated. This is consistent with #8881 and #8886 reviewed earlier this week (drip-157, drip-156) — locale parity drift is becoming a pattern in goose2 PRs that should be addressed at process level (e.g., a CI gate on translation-key parity).

5. **No CSRF protection on `POST` store-guest-html.** The proxy uses a shared secret in the *query string* for both the GET and POST endpoints. Query strings end up in browser history, server logs (depending on logging config), and `Referer` headers. Even bound to localhost, a CSRF attack from a malicious page that the user's browser is currently rendering can issue a cross-origin POST against `http://localhost:<port>/store-guest-html?secret=goose-acp-local` and inject arbitrary HTML into the next iframe load. Standard mitigations (move secret to a header, add CSRF token, strict CORS) are missing.

6. **Container dimensions in hostContext are user-controlled and reflected back to the app.** `ui/notifications/size-changed` triggers `Resize inline frame + request sticky autoscroll`. If the app reports a 100000px height, what's the ceiling? A misbehaving app could push the page into a scroll-storm. A bound on the iframe `height` would be defensive.

7. **Generated SDK files (`ui/sdk/src/generated/{client,index,types,zod}.gen.ts`) are committed in the PR.** That's correct for goose2's pattern, but means the PR's logical surface is smaller than the +2120 line count suggests. Worth flagging in the description so reviewers don't get sticker-shock.

## Verdict

**request-changes** — the core integration shape (separate inline-app proxy, distinct ACP method for app-originated tool calls, host-context surface, replay-preservation) is the right architecture. But the security shape of the local proxy (default-shared secret, query-string auth, no HTTPS, no CSRF protection on the POST endpoint, no clear eviction policy on the guest-HTML store) is not yet ready to ship even in DRAFT-merge form, because the security model has to be the *first* thing this code locks in — UI iteration is cheap, retrofitting authn/authz onto a deployed iframe-host bridge is not. The wire-protocol choice of flat-string `extension__toolName` is also harder to walk back than to fix now. Locale parity (Concern 4) is a process issue but should not block this PR alone. Move to merge-after-nits once Concerns 1, 3, and 5 are addressed structurally (not merely commented away as "future hardening").

