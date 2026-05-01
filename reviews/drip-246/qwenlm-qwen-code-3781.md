# PR #3781 — feat(weixin): add image sending support via CDN upload

- Repo: QwenLM/qwen-code
- Author: Mr-Maidong
- Head: `9149336b7b613666f8b920ad55bf6bd355aecfe2`
- URL: https://github.com/QwenLM/qwen-code/pull/3781
- Verdict: **needs-discussion**

## What lands

6-file +448/-13 in `packages/channels/weixin/`. Adds outbound image-send
support to the WeChat channel adapter via:

1. New `getUploadUrl()` and `uploadToCdn()` helpers in `api.ts:151-222`.
2. Refactored `sendMessage()` in `WeixinAdapter.ts:49-114` to parse
   `[IMAGE: /path/to/file.png]` markers out of model text, send the
   cleaned text, then upload+send each image via the CDN flow.
3. New default-channel-instructions block at `WeixinAdapter.ts:18-32` that
   tells the model "you CAN send images" using the bracket-marker syntax.
4. A per-message reminder injected at `WeixinAdapter.ts:42-44`:
   `envelope.text = "${IMAGE_INSTRUCTION}\n\n${envelope.text}"`.
5. Crypto-helper changes (`media.ts:54-67` adds `encryptAesEcb` /
   `computeMd5` exports) plus new test cases in `media.test.ts:90-142`
   and `send.test.ts:1-42,80-212`.

## Specific findings

- The marker-based protocol at `WeixinAdapter.ts:64-69`:
  ```ts
  const imageRegex = /\[IMAGE:\s*([^\]]+)\]/gi;
  const parsedImages: string[] = [];
  let cleanedText = text.replace(imageRegex, (_, path: string) => {
      parsedImages.push(path.trim());
      return '';
  });
  ```
  has two problems baked in:
  - **Path injection / arbitrary file read.** The regex captures any
    string between `[IMAGE: ` and `]` and feeds it into `sendImage({
    imagePath: ... })` at `:92-97` without any allowlist, sandbox, or
    realpath check. A model-output containing `[IMAGE: /etc/passwd]` or
    `[IMAGE: /Users/me/.ssh/id_rsa]` will dutifully exfiltrate the file
    to the WeChat CDN. The model is treated as a trusted operator of the
    local filesystem here, but in practice a prompt-injection attack via
    inbound user message could craft an `[IMAGE: ...]` reply. This needs
    either an allowlist directory (e.g. only `/tmp/qwen-images/` or a
    configured `outbound_images_dir`) or an explicit `realpath` +
    prefix-check gate.
  - **No size or MIME validation.** `sendImage` will read whatever path
    the model produces and push it into the CDN encryption + upload path
    in `api.ts:197-222`. A 4 GB binary, a non-image file, an arbitrary
    blob — all flow through. Need a bounded read with size cap and a
    magic-byte check before encryption.
- The forced per-message reminder at `:42-44` prepends the
  `IMAGE_INSTRUCTION` to *every* inbound user message. This is a coarse
  hammer:
  - Token cost: ~80 tokens added to every turn for the lifetime of the
    session, even when the user is plainly not asking for images.
  - Conversation pollution: the reminder is visible to the model as if
    the user said it. If the user asks "what did I just say?", the model
    will quote the reminder back. The intent should be `system`-channel
    text (channel capabilities), not in-band user-channel text.
  - The default `instructions` block at `:18-32` already instructs the
    model — duplicating it on every turn is a workaround for the model
    forgetting, which is better solved by re-emitting `instructions` in
    the system prompt or by elevating the marker to a tool call.
- `connect()` at `:18-32` overwrites `this.config.instructions` only when
  not already set. Good — respects user override. But the default
  instructions are written in mixed English+Chinese (`'Example: 如果你想发送
  /tmp/cat.png 给用户...'`). For an i18n-friendly channel adapter, the
  default should come from a localized resource or be English-only with
  the Chinese reminder gated on a locale config.
- `uploadToCdn()` at `api.ts:202-221`:
  ```ts
  const url = urlOrParam.startsWith('http')
      ? urlOrParam
      : `https://novac2c.cdn.weixin.qq.com/c2c/upload?encrypted_query_param=...`
  ```
  hardcodes the `novac2c.cdn.weixin.qq.com` host. Worth pulling that into
  a constant so it can be region-overridden, and worth confirming with
  upstream WeChat docs that a single CDN host serves all geos.
- `WeixinAdapter.ts:99-111` error path on per-image failure sends
  `图片发送失败: ${errMsg}` back to the user channel as a text message.
  Two issues: (a) the error message may contain the local filesystem path
  which leaks server-internal paths to the user, (b) the failure mode
  should also be logged structurally, not just `process.stderr.write`.
- `sendImage` is exported from `send.ts` but the diff snippet at line 471+
  isn't fully shown — confirm it (a) reads the file with a size cap,
  (b) validates MIME, (c) properly closes file handles on error paths.
- Test coverage at `media.test.ts:90-142` and `send.test.ts:80-212` is
  meaningful (crypto round-trip, send-text mocking) but no test exercises
  the *marker-extraction* code path or the path-traversal scenario.

## Risks

- **Security: arbitrary local file exfiltration via prompt injection.**
  The single most important issue. A user message that gets the model to
  echo `[IMAGE: /home/user/.aws/credentials]` will quietly upload that
  file to the WeChat CDN as an encrypted blob.
- **Token cost regression** on every WeChat-channel turn from the
  always-prepended reminder.
- **Resource exhaustion** from unbounded file reads in `sendImage`.

## Verdict

**needs-discussion**. The CDN-upload implementation in `api.ts` looks
mechanically correct and well-tested at the crypto layer, but the
adapter-level marker protocol needs a path allowlist + size cap before
this can ship without opening a file-exfiltration vector. Separately, the
per-turn reminder belongs in the system prompt not in-band, and the
default-instructions text should be locale-aware. Recommend splitting:
land the `api.ts` CDN helpers + the bounded-read `sendImage` first, then
land the marker protocol with the security gates in a follow-up so each
piece can be reviewed in isolation.
