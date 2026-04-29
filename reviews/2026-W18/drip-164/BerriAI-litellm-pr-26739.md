# BerriAI/litellm #26739 — remove /ui/chat page

- **PR:** https://github.com/BerriAI/litellm/pull/26739
- **Head SHA:** `0ccd7d76c0428dba8746c8e4e8cf1a3f4357fc95`
- **Size:** +0 / −76 (8 files deleted)

## Summary
Pure-deletion PR. Removes the static Next.js build artifacts under `litellm/proxy/_experimental/out/chat*` so the `/ui/chat` route returns 404 instead of serving stale UI. The chat page was already removed from the dashboard nav in a prior PR, but the deprecated static `chat.html` / `chat.txt` / `chat/__next.*.txt` files remained in the served `out/` tree, letting users still reach the old chat UI by typing the URL directly.

## Specific observations

1. **Files deleted, all under `litellm/proxy/_experimental/out/`:**
   - `chat.html` (-1) — pre-rendered HTML shell
   - `chat.txt` (-22) — Next.js RSC payload
   - `chat/__next._full.txt`, `__next._head.txt`, `__next._index.txt`, `__next._tree.txt`, `__next.chat.__PAGE__.txt`, `__next.chat.txt` (-71 total) — Next.js App Router runtime payloads

   This matches the documented "deletes the static build output" intent. Confirmed by inspecting the diff: every deletion is a pre-rendered Next.js artifact, no source-code changes, no Python proxy changes, no router changes.

2. **The `_experimental/out/` directory is the Next.js `output: 'export'` static build target** — it's checked into the repo as a pre-built bundle so the Python proxy can serve it as static files. That means deleting these files is the *only* required change to remove the route (since FastAPI's `StaticFiles` mount won't synthesize a 404-vs-200 distinction beyond filesystem presence). No Python-side route gating needed.

3. **Risk: stale build regeneration could re-introduce the files.** The `out/` directory is a build output. If the team's CI or a developer's local `npm run build && npm run export` regenerates from a Next.js project that still has the `app/chat/` route, these artifacts will reappear. The PR doesn't touch the upstream Next.js source under `ui/litellm-dashboard/` (or wherever the source page directory lives) — that's the load-bearing thing to verify. If the source-side `app/chat/page.tsx` still exists, the next person to rebuild will silently re-add what this PR deletes.

4. **Two `__next.*.txt` files duplicate the content of `chat.txt` and `chat/__next._full.txt`** (verified: byte-identical 22-line RSC payloads) — this is normal Next.js dual-emission for app-router routes, and deleting both is correct. The `__next._head.txt` (-6) carries metadata (`title: "LiteLLM Dashboard"`, favicon links, viewport meta) — those are already declared in the parent layout's head, so removing them does not affect other routes.

5. **No tests, no description-required tests** ("No new tests needed" checkbox). For a pure asset-deletion PR this is reasonable, but a one-line e2e/integration assertion (`GET /ui/chat → 404`) in whatever proxy-route test suite exists would lock the regression.

## Verdict: `merge-after-nits`

Mechanically correct, minimal, addresses the symptom. The unaddressed root cause (the source-side Next.js `app/chat/` route generating these artifacts on next build) means this fix may regress on the next UI rebuild PR.

## Recommended actions
- **(Required)** Verify the source-side Next.js chat route is also gone — `find ui/ -type d -name chat` or equivalent, and delete the `app/chat/page.tsx` (or wherever it lives) if it survived the original removal PR. Otherwise this fix has a half-life of one rebuild.
- **(Recommended)** Add a one-line test: `assert client.get("/ui/chat").status_code == 404` in whatever proxy-routes test suite covers `/ui/*` static serving.
- **(Optional)** Consider gitignoring `litellm/proxy/_experimental/out/chat*` to harden against accidental re-add via stale local build.
