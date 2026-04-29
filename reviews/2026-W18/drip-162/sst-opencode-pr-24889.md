# sst/opencode PR #24889 — feat(tool): add bounded wait tool

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24889
- Head SHA: `b928668d5b`
- State: OPEN, +1174/-0 across 6 files (closes #24888)

## What it does

Adds a built-in `wait` tool for bounded asynchronous waits inside an assistant turn. Five modes that all share a common `seconds` (1-3600) timeout and `cancel_if_file` sentinel:

1. `fixed` — pure delay
2. `until_file` — poll a path until size stable for `stable_ms` (default 10000) and at least `min_size_bytes`
3. `until_url` — `HEAD`/`GET` poll until status matches `until_url_status` (default 200)
4. `until_pid_exit` — poll `process.kill(pid, 0)` until ESRCH
5. `until_text` — tail-and-regex (default `m` flag, last 32KiB by default, max 1MiB)

Wires `wait` into `AVAILABLE_PERMISSIONS` (`agent.ts:34`), the registry (`tool/registry.ts:118,209,232`), and the TUI inline render (`session/index.tsx:1596-1598` + `Wait` component at `:2233-2256`). Notably **never gated by `ctx.ask`** — wait is treated as side-effect-free.

## Specific reads

- `tool/wait.ts:9-19` — bounds: `MAX_SECONDS=3600`, `DEFAULT_POLL_INTERVAL_MS=5000`, `MIN_POLL_INTERVAL_MS=100`, `MAX_POLL_INTERVAL_MS=60000`. Sound. The 100ms minimum prevents accidental tight-loop polls. The 1-hour cap is generous but bounded — model can't accidentally pause a turn for days.
- `tool/wait.ts:142-153` — `abortable` race uses `Effect.race` with `Effect.callback` registering `signal.addEventListener("abort", handler, { once: true })` and returning a sync cleanup. Correct shape — no leaked listener if the underlying effect wins. The `"__aborted__"` string sentinel is a code smell vs a tagged union (`Either`/`Effect.Tagged`), but workable.
- `tool/wait.ts:155-178` — `readFileTail` opens with `fsOpen(filePath, "r")`, reads only the last `min(size, maxBytes)` bytes via positional `fh.read(buf, 0, n, start)`. Correct and avoids reading multi-GiB log files end-to-end. Closes the handle in `finally` with `.catch(() => undefined)`. Good.
- `tool/wait.ts:180-186` — `compilePattern` strips invalid regex flags via `.replace(/[^gimsuy]/g, "")`. Misses the `d` (hasIndices) and `v` (set notation) flags introduced in modern Node — not a correctness issue (they get stripped), but `'v'` is sometimes copy-pasted from MDN examples and silently dropped here would surprise users. Add `dv` to the allow-set.
- `tool/wait.ts:188-211` — `probeUrl` does HEAD, falls back to GET on `405`/`501`. Only those two — many APIs return `400` or `404` on HEAD without explicitly disallowing it. Safer rule: fall back to GET on any 4xx/5xx if the user's `expected_status < 400`. Also: response body is never consumed, so on Node `fetch` the underlying socket stays in pool with body undrained — for poll loops that will leak buffer pressure. Recommend `await res.body?.cancel()` after status read.
- `tool/wait.ts:200-205` — single 10s `timeoutMs` for the whole HEAD-then-GET sequence. If HEAD takes 9s before timing out, GET only gets 1s. Worth either bumping per-request or accepting a per-request `url_request_timeout_ms`.
- `until_pid_exit` mode (not shown in diff slice) — uses `process.kill(pid, 0)` with `EPERM === alive` heuristic at `:147-152`. Standard Unix idiom. Note Windows: `process.kill(pid, 0)` on Windows throws `EINVAL` for any non-existent or denied PID, so the `EPERM` branch never fires, which is conservative-correct (treats inaccessible Windows PIDs as not-alive). Worth a `// Windows: ...` comment.
- `until_text` mode (`tool/wait.ts:289-358`): the polling loop checks `cancel_if_file` first, then reads the tail, then runs `regex.exec`. Title is updated at most once per second via `lastTitleUpdate` throttle. Good UX — TUI doesn't churn.
- `tool/wait.ts:300-305` — combined-arg validation (`until_text` and `until_text_pattern` must come together) is at runtime, not in the schema. Could be expressed as `Schema.Struct(...).check(...)` or as two related `Optional`s in a discriminated union. Minor.
- TUI `Wait` (`session/index.tsx:2233-2256`): renders a chip per active condition (`reason`, `target`, `pattern`, `url`, `pid`). The pattern preview uses `String(pattern()).slice(0, 24)` — fine, but doesn't escape special characters for terminal display, so a regex containing `\x1b` (unlikely but possible) could mangle the rendered line. Tag with `escapeAnsi` for safety.

## Risk

1. **No `ctx.ask` gate**. `wait` is treated as inert, but `until_url` issues real network requests to arbitrary URLs (including LAN/VPN endpoints, AWS metadata `169.254.169.254`, etc.) on a 5-second poll for up to 1 hour. That's an SSRF vector if the assistant is influenced by adversarial input (a user pasting "wait until http://10.0.0.5:8080/admin returns 200"). At minimum, gate `until_url` on a permission ("wait_url") with the URL as the pattern, or refuse non-public IP ranges by default.
2. **`until_file` polling** — `fs.stat` only, no FD lock or rename-vs-write distinction. If the producer writes to a temp file and renames-into-place, the wait sees `size = 0 → final` instantly without the stable-window ever satisfying. That's the *desired* behavior for atomic writes (since by the time we see the file it's done), but combined with `min_size_bytes` it can prematurely return on partial writes. Document the expected producer model.
3. **No max-concurrent-wait**. Nothing here prevents the model from issuing 50 parallel `wait` tool calls each polling a URL. The TUI inline-tool surface might handle that fine but the network/FD pressure is real.
4. **Test coverage**: file `test/tool/wait.test.ts` is in the diff (couldn't read here). Critical branches to pin: cancel_if_file race (sentinel appears mid-poll), abort during fetch (signal cleanup), text mode with regex backtracking on a 1MiB tail (catastrophic regex DoS — no `RegExp` timeout in Node, so a bad pattern + large tail can hang the entire wait until `seconds` expires), and the `EPERM === alive` branch on Linux (run as non-root vs PID 1).

## Verdict

`merge-after-nits` — careful, well-bounded design with thoughtful UX (per-second title updates, sentinel cancellation, tail-only reads). The `until_url` SSRF concern is the only structural one; everything else is tightening (regex backtracking guard, body drain on `probeUrl`, allow `d`/`v` regex flags, document atomic-write semantics). Net positive feature.

## What I learned

The "wait" primitive is deceptively easy to get wrong. The right shape isn't `setTimeout` — it's a poll loop with bounded interval, cancellation token, sentinel file, and one-shot regex match against a bounded tail. This PR's mode-discriminated schema (rather than five separate tools) keeps the surface small while making each branch its own self-contained loop, which is how I'd build it. The structural miss is treating "wait for URL to be reachable" as inert when it's actually a permissioned action against arbitrary network targets.
