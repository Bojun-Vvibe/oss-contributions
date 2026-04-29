# BerriAI/litellm#26725 — fix(proxy): block /ui/* routes when DISABLE_ADMIN_UI=true

- PR: https://github.com/BerriAI/litellm/pull/26725
- Head SHA: `203f493596e3d0bd888563543a25f09bbc634882`
- Author: ishaan-berri
- Diff: +14/-1 across `litellm/proxy/proxy_server.py`

## What changed

Adds a feature-gate around the static-file mount for the admin UI in `_try_populate_ui_directory(...)` (`proxy_server.py:1454-1469`). Previously, the function unconditionally executed:

```
app.mount("/ui", StaticFiles(directory=ui_path, html=True), name="ui")
```

The new code reads `DISABLE_ADMIN_UI` via `str_to_bool(os.getenv("DISABLE_ADMIN_UI", "false")) is True`, and on a true reading replaces the static mount with two stub handlers — `@app.get("/ui/{path:path}")` returning `admin_ui_disabled()` for any sub-path, and `@app.get("/ui")` for the bare-root request — with the existing mount preserved on the `else` branch. The new helper is imported at the top from the existing `litellm.proxy.common_utils.admin_ui_utils`.

## Observations

The intent (don't expose admin UI when an operator has explicitly disabled it) is sound and aligns with the existing `DISABLE_ADMIN_UI` env var, which is presumably already consumed elsewhere for auth flow. The shape of the fix — register stub handlers in lieu of the mount — is correct: `app.mount("/ui", StaticFiles(...))` and `@app.get("/ui/{path:path}")` are mutually exclusive at the FastAPI router level, so the `if/else` branching is the right primitive (you can't add the static mount and then hope a route handler shadows it; mounts and routes resolve in different layers).

A concrete gap worth raising before merge: the file's earlier blocks at `_try_populate_ui_directory` (the `mounted _next at {server_root_path}/ui/_next` comment is right above the changed lines) likely also do `app.mount("/ui/_next", ...)` for the Next.js static asset bundle. The new disable-gate doesn't appear to suppress that mount, which means `/ui/_next/*` still serves bundled JS/CSS even in the disabled state. That's mostly cosmetic — the JS would call into `/api/...` which is independently authed — but it leaks the fact that an admin UI bundle exists at all, and an operator who set `DISABLE_ADMIN_UI=true` for a stricter posture would reasonably expect zero `/ui*` surface. Worth confirming whether `_next` is gated by the same condition or whether this PR should extend coverage.

Also: HTTP method scope. The new handlers are `@app.get(...)` only. Browsers won't issue `POST /ui/...` for static-served content, but FastAPI/uvicorn will return `405 Method Not Allowed` for POST/PUT/DELETE under the disabled state, which leaks "this path exists" by virtue of giving a method-distinct response. Routing it via `@app.api_route("/ui/{path:path}", methods=["GET","POST","PUT","DELETE","PATCH","HEAD","OPTIONS"])` (or just `["*"]`) and returning `admin_ui_disabled()` uniformly would give the cleaner closed-shape.

The `str_to_bool(...) is True` defensive equality is consistent with elsewhere in the file. No test added in this diff — the change is small enough that's defensible, but a one-line `TestClient` assertion in the proxy-test suite (`response = client.get("/ui/foo"); assert response.status_code == 200 and "disabled" in response.text` under `DISABLE_ADMIN_UI=true`) would pin the intended behavior cheaply.

## Verdict

`merge-after-nits`
