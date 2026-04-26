# block/goose PR #8810 — return 400 instead of panicking on invalid CSP header value

- **PR**: https://github.com/block/goose/pull/8810
- **Author**: @r0x0d (Rodolfo Olivieri)
- **Head SHA**: `868d9b06577dcf73d04978d3d7c3440eae8bc8c0`

## Summary

In `crates/goose-server/src/routes/mcp_app_proxy.rs:255-273` the
`serve_guest_html` handler previously did:

```rust
headers.insert(header::CONTENT_SECURITY_POLICY, csp.parse().unwrap());
```

`HeaderValue::parse` returns `Result<HeaderValue, InvalidHeaderValue>`
when the input contains characters illegal in HTTP header values
(NUL, CR, LF, or non-visible bytes). A user-supplied CSP string with
e.g. an embedded newline would hit `.unwrap()` and panic the server
thread. Fix: replace with a `match` that returns
`(StatusCode::BAD_REQUEST, "Invalid characters in Content-Security-Policy value").into_response()`
on parse error.

The neighboring `referrer-policy: strict-origin` insert is also
hardened, but in a different way: it's a static literal, so the
author switched from `.parse().unwrap()` to
`HeaderValue::from_static("strict-origin")`, which is a compile-time-
checked path and cannot fail.

## Verdict: `merge-after-nits`

Right intent, simple fix, but two improvements would make it
robustly user-facing.

## Specific references

- `mcp_app_proxy.rs:257-258` — the `from_static` swap. Cleanest form
  for static strings; correct.
- `mcp_app_proxy.rs:261-271` — the new `match` arm. Returns 400
  cleanly. No allocation needed for the message because it's `&'static str`.

## Concerns

1. **Caller doesn't learn what was wrong.** The 400 body is the
   constant string `"Invalid characters in Content-Security-Policy value"`.
   The `InvalidHeaderValue` from the `Err(_)` branch is *discarded*.
   For an internal diagnostic surface (`mcp_app_proxy`), include the
   error in the response body or at least log it via `tracing::warn!`
   so operators can see which character was the problem (NUL? CR?
   high-byte?). Currently the user is left guessing.

2. **Where does the bad CSP come from?** If `csp` originates from
   user MCP server config (declarative `mcpServers.*.csp` field or
   similar), the fix should ideally validate at *config-load* time,
   not at every request. A 400 on every guest-HTML serve is loud but
   doesn't tell the user "this server config is broken — fix it
   here". Even keeping the request-time guard, also rejecting at
   config load (with a clearer error path) prevents the bad config
   from ever being live.

3. **No regression test in the diff.** A two-line
   `#[tokio::test] async fn serve_guest_html_returns_400_on_invalid_csp()`
   that hands in a CSP with `"\0"` or `"\n"` in it and asserts
   `response.status() == 400` would lock this in. Without the test,
   a future refactor could re-introduce `.unwrap()` and not be caught.

## Nits

- The `CONTENT_SECURITY_POLICY` constant could move to the top of
  the function as a `const CSP_HEADER: HeaderName = ...` to make the
  match cleaner — minor.
- Consider using `header::HeaderValue::try_from(csp.as_str())`
  instead of `csp.parse::<HeaderValue>()` — same outcome, slightly
  more readable intent.
