# QwenLM/qwen-code #3781 — feat(weixin): add image sending support via CDN upload

- URL: https://github.com/QwenLM/qwen-code/pull/3781
- Head SHA: `9149336b7b613666f8b920ad55bf6bd355aecfe2`
- Author: @Mr-Maidong
- Stats: +448 / -13 across 6 files

## Summary

Adds outbound image-send support to the WeChat channel via a four-step
CDN upload flow (read → getuploadurl → AES-encrypt + CDN upload →
sendmessage). New helpers in `media.ts` (`encryptAesEcb`, `computeMd5`,
exported `parseAesKey`), `api.ts` (`getUploadUrl`, `uploadToCdn`), and
`send.ts` (`sendImage`). `WeixinAdapter.ts` auto-injects an
"image-sending capability" instruction into model context and parses
`[IMAGE: /path/to/file]` markers from model output, sending each parsed
image and falling back to a text error message on failure.

## Specific feedback

- `packages/channels/weixin/src/WeixinAdapter.ts:46-60` — `if
  (!this.config.instructions)` defaults a non-trivial multi-line
  instruction block that teaches the model about the `[IMAGE: ...]`
  marker syntax. Concern: if a user has already configured
  `instructions` for any other reason, the image-send capability is
  silently NOT advertised. Either (a) document this clearly in the
  channel README, or (b) always append the image-capability snippet
  (with a sentinel to avoid double-append on reconnect).
- `packages/channels/weixin/src/WeixinAdapter.ts:170-176` — the
  `IMAGE_INSTRUCTION` is prepended to **every** inbound message envelope
  text. This is a sustained per-message context tax (and a deterministic
  prompt-injection vector if a future user message happens to legitimately
  contain `[IMAGE: ...]` text). Consider scoping to system-context
  injection only, or de-duplicating against the connect-time instruction.
- `packages/channels/weixin/src/WeixinAdapter.ts:190-194` — the regex
  `/\[IMAGE:\s*([^\]]+)\]/gi` is run on **outgoing** model text. This is
  the right place to do it, but there is no allowlist on the parsed
  path — a model that hallucinates `[IMAGE: /etc/shadow]` or
  `[IMAGE: ~/.ssh/id_rsa]` will cause `sendImage` to attempt to read,
  encrypt, and upload that file to a third-party CDN. **This is the
  primary blocker.** Needs at minimum: (a) path canonicalization, (b)
  allowlist directory enforcement, (c) deny dotfiles / parent-dir
  traversal.
- `packages/channels/weixin/src/api.ts:144-243` — `getUploadUrl` and
  `uploadToCdn` look correct on the protocol surface (both `upload_full_url`
  and `upload_param` paths handled, error shape `ret`/`errmsg`
  surfaced). The `aeskey: aeskeyHex` (16-byte AES key as 32-char hex)
  contract is documented at the call site, which is good.
- Error fallback at `WeixinAdapter.ts:218-227` sending `"图片发送失败:
  ${errMsg}"` over `sendText` will leak raw error messages (potentially
  including paths, CDN responses) to end users. Consider a sanitized
  user-facing message + structured server-side log.
- Test coverage claimed at 29/29 in PR body covers encryption round-trip
  and CDN four-step flow, which is the algorithmic core. The
  path-traversal / instruction-injection concerns above are NOT covered
  by those tests.
- Nit: AES-ECB is the documented WeChat protocol so this is forced, but
  worth a one-line comment noting that the choice is dictated by the
  upstream API rather than a security preference, so future reviewers
  don't try to "fix" it to CBC/GCM.

## Verdict

`request-changes` — feature works on the happy path, but the unfiltered
file-path read driven directly by model output is a real exfiltration
vector. Add path validation + allowlist before merge. The instruction-
injection-on-every-message pattern is also worth tightening but is a
secondary ask.
