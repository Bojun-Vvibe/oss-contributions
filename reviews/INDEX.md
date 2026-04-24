# Review Index

140 PR reviews across 8 OSS AI-coding-agent projects. Each review
contains: context, problem, design analysis with quoted snippets
where useful, risks, suggestions, verdict, and a "what I learned"
section.

- **Round 1** (W7): the original 24, focused on feature additions
  and integration work.
- **W9 batch**: 24 more, focused on subtle correctness bugs —
  silent payload drops, deadlock recovery, cross-route loops,
  parity bugs between two surfaces sharing a backend.
- **W13 batch**: 24 more, focused on contract violations,
  diagnostic-first debugging, fail-open vs fail-loud policy
  decisions, and "wrong abstraction was added → roll it back"
  refactors.
- **W17 drip (2026-04-24)**: 4 extras — small-surface fixes in
  compaction, LSP cancel paths, error-class idempotence, and
  deadline plumbing.
- **W17 drip-2 (2026-04-24)**: 4 more — TUI render-completeness
  for multi-part user messages, exec-server EOF/exit ordering,
  cross-vendor `tool_choice="none"` translation, and processor
  registration parity across conversation creation paths.
- **W17 drip-3 (2026-04-24)**: 4 more — feature-flagged HttpApi
  workspace-read carve-out, retiring spawned-agent context
  scaffolding in multi-agent v2, working around a Prisma `Json?`
  null-filter limitation via `query_raw IS NOT NULL` in the
  budget-window reset job, and V0→V1 dependency carve-off (memory
  package external-reference cleanup).
- **W17 drip-4 (2026-04-24)**: 4 more — Bun fetch
  stream-disconnect retry classification, experimental
  `disable_vcs_diff` config flag as a session-page mitigation,
  `truncation_policy` clamp pushed down into `unified_exec` so
  rollout persistence stops diverging from model-visible context,
  and the `PermissionProfile` tagged-union refactor that makes
  `managed`/`disabled`/`external` enforcement modes round-trip
  faithfully across the v2 wire.
- **W17 drip-5 (2026-04-24)**: 4 more — a `User-Agent` header
  collision in the session LLM layer that silently dropped
  user-configured provider headers, a quota-vs-window 429
  classification gap that turned weekly-cap rejections into
  unbounded retry loops, a layered-config `AbsolutePathBuf`
  base-path bug that broke relative `agents.*.config_file`
  paths in agent role configs, and a Unix-socket transport
  unification onto the standard WebSocket HTTP-Upgrade
  handshake to close the framing gap with the TCP path.
- **W17 drip-6 (2026-04-24)**: 4 more — numeric-tool-call-ID
  coercion at the fetch interceptor for spec-violating
  OpenAI-compatible providers (NVIDIA NIM kimi-k2.5 class),
  collapse of the dual-arm (substring + JSON-parse) retry
  classifier into a single broad-regex predicate covering
  ~25 transient surfaces, SQLite WAL-mode corruption fix
  via `MaxOpenConns(1)` + `_txlock=immediate` against
  concurrent sub-agent write races, and `$VAR` env-var
  expansion in stdio MCP server `args` via the same
  `ResolveValue` resolver already used for `command`/`env`/
  `headers`.
- **W17 drip-7 (2026-04-24)**: 4 more — snapshot revert
  E2BIG fix moving the file list off argv onto stdin via
  `git checkout --pathspec-from-file=- --pathspec-file-nul`,
  Windows elevated-sandbox PID gate on the named-pipe
  consumer (`GetNamedPipeClientProcessId` compared against
  the PID returned from `CreateProcessWithLogonW`) layered
  with a ~260-line dedup of the one-shot capture path onto
  the shared `spawn_runner_transport` helper, skill
  discovery dedup-set keyed on `filepath.EvalSymlinks`
  output so symlinked discovery roots don't double-render
  in the sidebar, and a duplicate
  `MAX_SIZE_PER_ITEM_IN_MEMORY_CACHE_IN_KB` constant
  removal that silently doubles the in-memory cache item
  ceiling from 512 KB to 1024 KB across deployments.
- **W17 drip-8 (2026-04-24)**: 8 more — default-flip from
  opt-in to opt-out for `compaction.prune` (one operator
  flip, paired test split with shared fixture); LAN-host
  proxy carve-out (`ensureNoProxyForBaseURL` mutates
  `process.env.NO_PROXY` for RFC1918/loopback/link-local/
  ULA hostnames when `HTTP_PROXY` is set); `model_provider`
  restore on thread resume in `merge_persisted_resume_
  metadata` (one missing field assignment, four-branch test
  expansion); enabled-row renumbering and digit-shortcut
  reindex in `ListSelectionView` so disabled rows no longer
  steal default focus or shortcut keys; reasoning-model
  empty-title symptom-detection retry path
  (`isUsableTitleResponse` swap of small→large model when
  `finish_reason=stop` lands with empty content); ctx
  cancellation threaded through every layer of the
  pure-Go grep fallback (`searchFilesWithRegex` →
  `fileContainsPattern` → per-line scanner) with deadline +
  cancellation regression tests; `output_config` adapter-
  internal key added to the exclusion set so adaptive
  thinking translates to `reasoning_effort` without leaking
  to non-Anthropic backends; and the
  `_enforce_upperbound_key_params(data, fill_defaults=False)`
  call added to `_execute_virtual_key_regeneration` to
  close the budget/duration privilege-escalation primitive
  on `/key/regenerate`.
- **W17 drip-9 (2026-04-24)**: 8 more — desktop sidecar
  health-check spin-poll replaced with bounded
  exponential-backoff retry (6 attempts, 500→1000→2000→
  4000→4000 ms, 2 s per-attempt timeout, ~23.5 s total
  budget) plus an unconditional `no_proxy()` flip that
  silently widens past the previous loopback-only check;
  plan-mode bash deny-by-default with a 14-command
  read-only allowlist enforced via a merge-order inversion
  in a new `Agent.permissions(agent, session)` helper
  (agent rules win for `plan`, session rules win
  elsewhere); `onCleanup`-paired unsubscribe handles for
  ~6 long-lived Solid TUI `event.on` registrations + a
  `process.on("SIGHUP")` leak; HttpApi bridge of `GET
  /mcp` status with the Effect-Schema-with-`.zod`-shim
  pattern that lets the legacy Hono route keep working;
  `CancellationToken`-propagated deferred network-proxy
  denials so a Guardian-class denial arriving mid-command
  cancels the running process instead of being silently
  swallowed (with a `cancel_when_either` watchdog-task
  pattern that risks leaking a parked tokio task per
  command); supply-chain hardening sweep that pins
  `@openai/codex` from `latest` to `0.121.0` in two
  Dockerfiles, adds two committed `pnpm-lock.yaml`
  install-boundary directories with `minimumReleaseAge:
  10080` + `blockExoticSubdeps: true` + `strictDepBuilds:
  true` + `trustPolicy: no-downgrade`, and commits two
  Python `uv.lock` files; LSP `workspace/applyEdit`
  filesystem-mutation boundary check via a new
  `validateWorkspacePath` that calls `fsext.HasPrefix` on
  resolved abs paths (does NOT call `EvalSymlinks`, so
  symlink-escape gap remains); `OnRetry` callback body
  filled with `slog.Warn` + `providerRetryLogFields`
  helper after a stale TODO; and a Bedrock guardrails
  `experimental_guardrail_input_roles` filter that drops
  non-matching roles from INPUT validation (case-
  sensitive role compare, silent-bypass on
  empty-after-filter that should log).
- **W17 drip-10 (2026-04-24)**: 8 more — empty-string
  `reasoning_content` survival through the
  openai-compatible transform for DeepSeek V4 thinking
  mode (truthy-guard removal, fragile coupling to AI-SDK
  outbound metadata spread); `tui thread` config-aware
  network-options resolver restoration (two-symbol revert
  of an undocumented regression in `675a46e23e` that
  silently bypassed `opencode.json` hostname/port);
  LSP-tool permission metadata enrichment bundled with a
  silent `Effect.orDie` error-policy flip; OAuth login
  callback ephemeral-port fallback when `127.0.0.1:1455`
  is held by Cursor (load-bearing assumption: OAuth
  client must be registered with `http://localhost`
  without a fixed port, otherwise fallback succeeds at
  bind but fails at `redirect_uri_mismatch`); dynamic
  tool-handler `pre/post_tool_use_payload` impls that
  silently drop the hook event on `parse_arguments`
  failure (audit hooks miss the malformed-call case
  they were designed to observe); stateful
  `StreamingPatchParser` refactor that trades typed
  `ParseError` returns for an `invalid: bool + return
  None forever` failure mode (+619/-433 churn,
  removes public `parse_patch_streaming` /
  `ParseMode::Streaming`); crush command-palette
  three-fer (`exit` alias as duplicate row not true
  alias, `\p{L}\p{N}` Unicode placeholder regex, empty
  tool-call input `""→"{}"` normalization duplicated
  across `agent` and `message` packages); and Azure
  container file-endpoint deployment-credential
  resolution by decoded `model_id` with an
  unconditional `api_base`/`api_key`/`api_version`
  override that breaks BYOK proxies (should be
  fill-defaults, not overwrite) plus a silent
  `model_group_name` fallback that can re-route to a
  different deployment.
- **W17 drip-10 (2026-04-24)**: 8 more — surfacing
  `reasoning_output_tokens` on the public `codex exec
  --json` `Usage` payload (and the breaking-direction
  asymmetry that creates for strict downstream consumers);
  `#[schemars(skip)]` on `RawMcpServerConfig.bearer_token`
  to stop the JSON schema from advertising a field the
  runtime rejects (with the unfinished editor-UX migration
  to a `deprecated + description` pattern); two-layer
  (JS kernel + Rust host) MIME allowlist on
  `codex.emitImage` so unsupported encodings stop tripping
  the output sanitizer's `debug_assert!`; `include_archived:
  true` symmetry fix for the `thread/archive` parent read
  that turns a server-side ghost-failure into idempotent
  success; MiniMax `sk-` prefix check removal so token-plan
  keys are no longer rejected at config load; mode-gated
  `max_tokens` injection in the litellm proxy health check
  (denylist of `image_generation` only — worth inverting to
  an allowlist of chat-only modes); `kwargs_for_followup`
  vs `optional_params` dedup at request-patch construction
  in the websearch interception agentic loop to stop
  `TypeError: got multiple values for keyword argument`
  follow-up crashes; and a three-in-one guardrail PR
  combining recursive sensitive-key masking, masked
  `litellm_params` in `/guardrails/submissions`, and
   team-scoped non-admin filtering on `/v2/guardrails/list`
   with an authorization-shape change to the route signature.
- **W17 drip-11 (2026-04-24)**: 8 more — narrowing of a
  loose substring match (`id.includes("deepseek")`) into
  a four-family allow-list that trades forward-compat for
  precision; a `parseManagedPlist` non-object-JSON guard
  that fixes a startup crash but silently masks malformed
  MDM payloads; a second `CONFIG_KEY_ALIASES` table entry
  (`agents.max_concurrent_threads_per_session` →
  `max_threads`) with no collision policy or deprecation
  warn; a tagged-union `ApprovalPolicyConstraint` that
  introduces a scoped `granular = "any"` wildcard on
  `allowed_approval_policies` (with the permanent
  governance trade-off that adding new granular sub-flags
  later silently loosens existing wildcards); env-resolution
  test coverage for the hyper provider's `sync.OnceValue`
  package globals via test-file-local reset helpers (with
  the parallel-safety hazard that comes with mutating
  package state from tests); a one-line `AtBottom()` fix
  in the list widget that subtracts `offsetLine` before the
  early-exit comparison so auto-follow stops desyncing
  after manual scroll-back; addition of a `get_user_models`
  helper plus integration into `get_available_models_for_user`
  to close a listing-vs-invocation ACL parity gap on
  `GET /v1/models`; and a high-severity Bedrock passthrough
  ACL fix that adds path-extraction + reverse-lookup from
  Bedrock `modelId` to LiteLLM `model_group` so that
  `/bedrock/.../invoke` and friends can no longer bypass
  `key.models` / `user.models` restrictions.

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.

## Aider-AI/aider

| PR | Title | File |
|---|---|---|
| [#4748](https://github.com/Aider-AI/aider/pull/4748) | Fix regression in the LiteLLM exception list | [PR-4748.md](Aider-AI-aider/PR-4748.md) |
| [#4830](https://github.com/Aider-AI/aider/pull/4830) | Allow adding files outside repo when git commits off | [PR-4830.md](Aider-AI-aider/PR-4830.md) |
| [#4838](https://github.com/Aider-AI/aider/pull/4838) | fix: handle symlink loops in safe_abs_path() | [PR-4838.md](Aider-AI-aider/PR-4838.md) |
| [#4855](https://github.com/Aider-AI/aider/pull/4855) | feat: Add native EUrouter provider support | [PR-4855.md](Aider-AI-aider/PR-4855.md) |
| [#4858](https://github.com/Aider-AI/aider/pull/4858) | feat: add double-buffer context management | [PR-4858.md](Aider-AI-aider/PR-4858.md) |
| [#4936](https://github.com/Aider-AI/aider/pull/4936) | Add ACP (Agent Communication Protocol) support adapter | [PR-4936.md](Aider-AI-aider/PR-4936.md) |
| [#4674](https://github.com/Aider-AI/aider/pull/4674) | Allow adding read-only files to editable with git: false | [PR-4674.md](Aider-AI-aider/PR-4674.md) |
| [#4682](https://github.com/Aider-AI/aider/pull/4682) | Don't stage files with no-auto-commits | [PR-4682.md](Aider-AI-aider/PR-4682.md) |
| [#4935](https://github.com/Aider-AI/aider/pull/4935) | fix: replace two deprecated models, delete one | [PR-4935.md](Aider-AI-aider/PR-4935.md) |

## All-Hands-AI/OpenHands

| PR | Title | File |
|---|---|---|
| [#13977](https://github.com/All-Hands-AI/OpenHands/pull/13977) | Warn before resetting hidden SDK settings | [PR-13977.md](All-Hands-AI-OpenHands/PR-13977.md) |
| [#13983](https://github.com/All-Hands-AI/OpenHands/pull/13983) | feat(app-server): route ACP agents to the ACP conversation endpoint | [PR-13983.md](All-Hands-AI-OpenHands/PR-13983.md) |
| [#13994](https://github.com/All-Hands-AI/OpenHands/pull/13994) | feat(frontend): render ACPToolCallEvent in conversation viewer | [PR-13994.md](All-Hands-AI-OpenHands/PR-13994.md) |
| [#14042](https://github.com/All-Hands-AI/OpenHands/pull/14042) | fix: prevent infinite redirect loop on org-defaults settings pages | [PR-14042.md](All-Hands-AI-OpenHands/PR-14042.md) |
| [#14044](https://github.com/All-Hands-AI/OpenHands/pull/14044) | fix(backend): repair org-defaults LLM save flow and sync managed keys to members | [PR-14044.md](All-Hands-AI-OpenHands/PR-14044.md) |
| [#14049](https://github.com/All-Hands-AI/OpenHands/pull/14049) | fix(frontend): restore notification sound and browser tab flash | [PR-14049.md](All-Hands-AI-OpenHands/PR-14049.md) |
| [#14032](https://github.com/All-Hands-AI/OpenHands/pull/14032) | fix(backend): restore git-organizations endpoint for git conversation routing | [PR-14032.md](All-Hands-AI-OpenHands/PR-14032.md) |
| [#14051](https://github.com/All-Hands-AI/OpenHands/pull/14051) | fix: restore org settings payload contract | [PR-14051.md](All-Hands-AI-OpenHands/PR-14051.md) |
| [#14065](https://github.com/All-Hands-AI/OpenHands/pull/14065) | fix: migrate core SQLAlchemy models to SQLAlchemy 2.0 mapped_column | [PR-14065.md](All-Hands-AI-OpenHands/PR-14065.md) |
| [#14102](https://github.com/All-Hands-AI/OpenHands/pull/14102) | Fix: Register SetTitleCallbackProcessor for webhook-created conversations | [PR-14102.md](All-Hands-AI-OpenHands/PR-14102.md) |
| [#14106](https://github.com/All-Hands-AI/OpenHands/pull/14106) | refactor: remove external dependencies on V0 packages (controller, memory, microagent) | [PR-14106.md](All-Hands-AI-OpenHands/PR-14106.md) |

## anomalyco/opencode

| PR | Title | File |
|---|---|---|
| [#23755](https://github.com/anomalyco/opencode/pull/23755) | fix: preserve thinking/redacted_thinking blocks in Anthropic message transforms | [PR-23755.md](anomalyco-opencode/PR-23755.md) |
| [#23797](https://github.com/anomalyco/opencode/pull/23797) | fix: preserve UTF-8 BOM through edit/patch operations | [PR-23797.md](anomalyco-opencode/PR-23797.md) |
| [#23837](https://github.com/anomalyco/opencode/pull/23837) | fix: fail fast on invalid session payloads | [PR-23837.md](anomalyco-opencode/PR-23837.md) |
| [#23862](https://github.com/anomalyco/opencode/pull/23862) | fix: sessions missing from sidebar on Windows due to path separator mismatch | [PR-23862.md](anomalyco-opencode/PR-23862.md) |
| [#23866](https://github.com/anomalyco/opencode/pull/23866) | fix(project): use worktree paths for project ids | [PR-23866.md](anomalyco-opencode/PR-23866.md) |
| [#23870](https://github.com/anomalyco/opencode/pull/23870) | fix: session compaction loss-of-context | [PR-23870.md](anomalyco-opencode/PR-23870.md) |
| [#23771](https://github.com/anomalyco/opencode/pull/23771) | fix(opencode): drain async iterator on Anthropic stream interruption | [PR-23771.md](anomalyco-opencode/PR-23771.md) |
| [#23808](https://github.com/anomalyco/opencode/pull/23808) | fix: race condition in tool-call ID generation under parallel sub-agent runs | [PR-23808.md](anomalyco-opencode/PR-23808.md) |
| [#24001](https://github.com/anomalyco/opencode/pull/24001) | feat: send-many-streams concurrency primitive for sub-agent fan-out | [PR-24001.md](anomalyco-opencode/PR-24001.md) |
| [#24087](https://github.com/anomalyco/opencode/pull/24087) | fix(session): drop zero-length assistant turns before compaction | [PR-24087.md](anomalyco-opencode/PR-24087.md) |
| [#24009](https://github.com/anomalyco/opencode/pull/24009) | fix(tui): render all non-synthetic text parts of a user message | [PR-24009.md](anomalyco-opencode/PR-24009.md) |
| [#24062](https://github.com/anomalyco/opencode/pull/24062) | feat(httpapi): bridge workspace read endpoints | [PR-24062.md](anomalyco-opencode/PR-24062.md) |
| [#24076](https://github.com/anomalyco/opencode/pull/24076) | fix: handle Bun stream connection errors with automatic retry | [PR-24076.md](anomalyco-opencode/PR-24076.md) |
| [#24079](https://github.com/anomalyco/opencode/pull/24079) | fix(app): add experimental flag to disable vcs diff auto-fetch | [PR-24079.md](anomalyco-opencode/PR-24079.md) |
| [#24066](https://github.com/anomalyco/opencode/pull/24066) | fix(provider): preserve custom User-Agent from provider.options.headers | [PR-24066.md](anomalyco-opencode/PR-24066.md) |
| [#24013](https://github.com/anomalyco/opencode/pull/24013) | fix(opencode): stop retrying non-transient rate limits | [PR-24013.md](anomalyco-opencode/PR-24013.md) |
| [#24026](https://github.com/anomalyco/opencode/pull/24026) | fix(provider): coerce numeric tool call IDs for OpenAI-compatible providers | [PR-24026.md](anomalyco-opencode/PR-24026.md) |
| [#24033](https://github.com/anomalyco/opencode/pull/24033) | tweak: simplify retry logic | [PR-24033.md](anomalyco-opencode/PR-24033.md) |
| [#24116](https://github.com/anomalyco/opencode/pull/24116) | fix(snapshot): avoid E2BIG during batched revert checkout | [PR-24116.md](anomalyco-opencode/PR-24116.md) |
| [#24117](https://github.com/sst/opencode/pull/24117) | fix(provider): allow remote local-network hosts when proxy env vars are set | [PR-24117-noproxy-private-hosts.md](anomalyco-opencode/PR-24117-noproxy-private-hosts.md) |
| [#24127](https://github.com/sst/opencode/pull/24127) | fix: enable compaction prune by default | [PR-24127-compaction-prune-default.md](anomalyco-opencode/PR-24127-compaction-prune-default.md) |
| [#24138](https://github.com/sst/opencode/pull/24138) | fix(desktop): add retry logic with exponential backoff to health check system | [PR-24138-health-check-retry-backoff.md](anomalyco-opencode/PR-24138-health-check-retry-backoff.md) |
| [#24110](https://github.com/sst/opencode/pull/24110) | fix(opencode): enforce read-only bash permissions in plan mode | [PR-24110-plan-mode-readonly-bash.md](anomalyco-opencode/PR-24110-plan-mode-readonly-bash.md) |
| [#24053](https://github.com/sst/opencode/pull/24053) | fix(tui): unsubscribe event listeners on component disposal | [PR-24053-tui-unsubscribe-listeners.md](anomalyco-opencode/PR-24053-tui-unsubscribe-listeners.md) |
| [#24100](https://github.com/sst/opencode/pull/24100) | feat(httpapi): bridge mcp status endpoint | [PR-24100-httpapi-mcp-status.md](anomalyco-opencode/PR-24100-httpapi-mcp-status.md) |
| [#24146](https://github.com/sst/opencode/pull/24146) | fix: preserve empty reasoning_content for DeepSeek V4 thinking mode | [PR-24146-deepseek-empty-reasoning.md](anomalyco-opencode/PR-24146-deepseek-empty-reasoning.md) |
| [#24140](https://github.com/sst/opencode/pull/24140) | cli/tui: honor configured network defaults in thread mode | [PR-24140-thread-network-defaults.md](anomalyco-opencode/PR-24140-thread-network-defaults.md) |
| [#24139](https://github.com/sst/opencode/pull/24139) | tool/lsp: include request details in permission metadata | [PR-24139-lsp-permission-metadata.md](anomalyco-opencode/PR-24139-lsp-permission-metadata.md) |
| [#24157](https://github.com/sst/opencode/pull/24157) | fix: deepseek variants — substring → allow-list | [PR-24157-deepseek-variant-narrowing.md](anomalyco-opencode/PR-24157-deepseek-variant-narrowing.md) |
| [#24118](https://github.com/sst/opencode/pull/24118) | fix(config): avoid parseManagedPlist crash on non-object JSON | [PR-24118-managed-plist-non-object-guard.md](anomalyco-opencode/PR-24118-managed-plist-non-object-guard.md) |

## BerriAI/litellm

| PR | Title | File |
|---|---|---|
| [#26219](https://github.com/BerriAI/litellm/pull/26219) | fix: ChatGPT responses bridge recovery | [PR-26219.md](BerriAI-litellm/PR-26219.md) |
| [#26228](https://github.com/BerriAI/litellm/pull/26228) | fix(anthropic): skip file-block discovery on unsupported content | [PR-26228.md](BerriAI-litellm/PR-26228.md) |
| [#26266](https://github.com/BerriAI/litellm/pull/26266) | fix(bedrock): guardrail logging + redaction | [PR-26266.md](BerriAI-litellm/PR-26266.md) |
| [#26274](https://github.com/BerriAI/litellm/pull/26274) | fix(mcp): harden OAuth authorize/token endpoints (BYOK + discoverable) | [PR-26274.md](BerriAI-litellm/PR-26274.md) |
| [#26279](https://github.com/BerriAI/litellm/pull/26279) | fix(auth): centralize common_checks to close authorization bypass | [PR-26279.md](BerriAI-litellm/PR-26279.md) |
| [#26285](https://github.com/BerriAI/litellm/pull/26285) | fix(anthropic): preserve reasoning content and add think-tag regression coverage | [PR-26285.md](BerriAI-litellm/PR-26285.md) |
| [#26264](https://github.com/BerriAI/litellm/pull/26264) | fix(z.coerce): silent type-coercion bug in OpenAPI param validation | [PR-26264.md](BerriAI-litellm/PR-26264.md) |
| [#26270](https://github.com/BerriAI/litellm/pull/26270) | fix: unify diagnostic surfacing across push/pull pipelines | [PR-26270.md](BerriAI-litellm/PR-26270.md) |
| [#26287](https://github.com/BerriAI/litellm/pull/26287) | fix: model deprecation pruning + downstream test fixups | [PR-26287.md](BerriAI-litellm/PR-26287.md) |
| [#26312](https://github.com/BerriAI/litellm/pull/26312) | fix(router): don't re-wrap structured errors from upstream providers | [PR-26312.md](BerriAI-litellm/PR-26312.md) |
| [#24457](https://github.com/BerriAI/litellm/pull/24457) | fix(anthropic): handle tool_choice type 'none' in messages API | [PR-24457.md](BerriAI-litellm/PR-24457.md) |
| [#26346](https://github.com/BerriAI/litellm/pull/26346) | fix: reset_budget_windows around Prisma Json? null-filter limitation via query_raw IS NOT NULL | [PR-26346.md](BerriAI-litellm/PR-26346.md) |
| [#26385](https://github.com/BerriAI/litellm/pull/26385) | fix: remove duplicate MAX_SIZE_PER_ITEM_IN_MEMORY_CACHE_IN_KB definition | [PR-26385.md](BerriAI-litellm/PR-26385.md) |
| [#26340](https://github.com/BerriAI/litellm/pull/26340) | fix(key_management): enforce upperbound_key_generate_params on /key/regenerate | [PR-26340-key-regenerate-upperbound.md](BerriAI-litellm/PR-26340-key-regenerate-upperbound.md) |
| [#26383](https://github.com/BerriAI/litellm/pull/26383) | fix: prevent Azure output_config leakage | [PR-26383-azure-output-config-leak.md](BerriAI-litellm/PR-26383-azure-output-config-leak.md) |
| [#26393](https://github.com/BerriAI/litellm/pull/26393) | feat: bedrock guardrail input roles filter | [PR-26393-bedrock-guardrail-input-roles.md](BerriAI-litellm/PR-26393-bedrock-guardrail-input-roles.md) |
| [#26402](https://github.com/BerriAI/litellm/pull/26402) | fix(proxy): route azure container file requests by decoded deployment | [PR-26402-azure-container-file-routing.md](BerriAI-litellm/PR-26402-azure-container-file-routing.md) |
| [#26417](https://github.com/BerriAI/litellm/pull/26417) | fix(health_check): skip max_tokens for image_generation mode | [PR-26417-image-generation-skip-max-tokens.md](BerriAI-litellm/PR-26417-image-generation-skip-max-tokens.md) |
| [#26400](https://github.com/BerriAI/litellm/pull/26400) | fix(websearch): deduplicate overlapping keys between kwargs and optional_params | [PR-26400-websearch-kwargs-optional-params-dedup.md](BerriAI-litellm/PR-26400-websearch-kwargs-optional-params-dedup.md) |
| [#26390](https://github.com/BerriAI/litellm/pull/26390) | [Fix] Guardrail param handling in list and submission endpoints | [PR-26390-guardrail-team-scoping-mask-recursion.md](BerriAI-litellm/PR-26390-guardrail-team-scoping-mask-recursion.md) |
| [#26421](https://github.com/BerriAI/litellm/pull/26421) | fix(proxy): apply user.models restriction to GET /v1/models list | [PR-26421-user-models-list-filter.md](BerriAI-litellm/PR-26421-user-models-list-filter.md) |
| [#26416](https://github.com/BerriAI/litellm/pull/26416) | fix(auth): enforce model ACL on Bedrock passthrough routes | [PR-26416-bedrock-passthrough-acl.md](BerriAI-litellm/PR-26416-bedrock-passthrough-acl.md) |

## charmbracelet/crush

| PR | Title | File |
|---|---|---|
| [#2579](https://github.com/charmbracelet/crush/pull/2579) | feat(tool): add `ask-user-questions` tool | [PR-2579.md](charmbracelet-crush/PR-2579.md) |
| [#2613](https://github.com/charmbracelet/crush/pull/2613) | fix(agent): prune excess images from history to prevent session deadlock | [PR-2613.md](charmbracelet-crush/PR-2613.md) |
| [#2615](https://github.com/charmbracelet/crush/pull/2615) | fix(agent): validate tool call/results + strip tags from titles | [PR-2615.md](charmbracelet-crush/PR-2615.md) |
| [#2619](https://github.com/charmbracelet/crush/pull/2619) | fix(lsp): mitigate stale diagnostics | [PR-2619.md](charmbracelet-crush/PR-2619.md) |
| [#2622](https://github.com/charmbracelet/crush/pull/2622) | fix: inject synthetic tool_result for orphaned tool_use on session resume | [PR-2622.md](charmbracelet-crush/PR-2622.md) |
| [#2663](https://github.com/charmbracelet/crush/pull/2663) | fix(app): replace single events channel with pubsub.Broker for fan-out | [PR-2663.md](charmbracelet-crush/PR-2663.md) |
| [#2597](https://github.com/charmbracelet/crush/pull/2597) | fix(agent): prevent session corruption due to malformed image data | [PR-2597.md](charmbracelet-crush/PR-2597.md) |
| [#2607](https://github.com/charmbracelet/crush/pull/2607) | feat: generally render output that looks like a diff as a diff | [PR-2607.md](charmbracelet-crush/PR-2607.md) |
| [#2611](https://github.com/charmbracelet/crush/pull/2611) | fix(events): prevent early events from being dropped before init | [PR-2611.md](charmbracelet-crush/PR-2611.md) |
| [#2691](https://github.com/charmbracelet/crush/pull/2691) | fix(lsp): cancel in-flight diagnostics request on buffer close | [PR-2691.md](charmbracelet-crush/PR-2691.md) |
| [#2690](https://github.com/charmbracelet/crush/pull/2690) | fix(db): prevent SQLITE_NOTADB corruption under concurrent sub-agents | [PR-2690.md](charmbracelet-crush/PR-2690.md) |
| [#2693](https://github.com/charmbracelet/crush/pull/2693) | fix(mcp): expand environment variables in stdio MCP server args | [PR-2693.md](charmbracelet-crush/PR-2693.md) |
| [#2694](https://github.com/charmbracelet/crush/pull/2694) | fix(skills): deduplicate skills discovered via symlinked directories | [PR-2694.md](charmbracelet-crush/PR-2694.md) |
| [#2652](https://github.com/charmbracelet/crush/pull/2652) | fix(grep): stop regex fallback after cancellation | [PR-2652-grep-cancel-fallback.md](charmbracelet-crush/PR-2652-grep-cancel-fallback.md) |
| [#2681](https://github.com/charmbracelet/crush/pull/2681) | fix(agent): retry title generation with large model on empty output | [PR-2681-title-empty-retry.md](charmbracelet-crush/PR-2681-title-empty-retry.md) |
| [#2699](https://github.com/charmbracelet/crush/pull/2699) | fix(lsp): enforce workspace boundary for workspace edits | [PR-2699-lsp-workspace-boundary.md](charmbracelet-crush/PR-2699-lsp-workspace-boundary.md) |
| [#2700](https://github.com/charmbracelet/crush/pull/2700) | fix(agent): implement OnRetry logging with structured retry fields | [PR-2700-onretry-structured-logging.md](charmbracelet-crush/PR-2700-onretry-structured-logging.md) |
| [#2675](https://github.com/charmbracelet/crush/pull/2675) | Fix command aliases/args parsing and empty tool-call input normalization | [PR-2675-cmd-aliases-empty-toolcall-input.md](charmbracelet-crush/PR-2675-cmd-aliases-empty-toolcall-input.md) |
| [#2688](https://github.com/charmbracelet/crush/pull/2688) | fix: remove minimax api key validate | [PR-2688-minimax-key-validation-removal.md](charmbracelet-crush/PR-2688-minimax-key-validation-removal.md) |
| [#2698](https://github.com/charmbracelet/crush/pull/2698) | test(hyper): add provider env behavior coverage | [PR-2698-hyper-provider-env-tests.md](charmbracelet-crush/PR-2698-hyper-provider-env-tests.md) |
| [#2647](https://github.com/charmbracelet/crush/pull/2647) | fix(ui): AtBottom() early exit not accounting for offsetLine | [PR-2647-atbottom-offsetline.md](charmbracelet-crush/PR-2647-atbottom-offsetline.md) |

## cline/cline

| PR | Title | File |
|---|---|---|
| [#10210](https://github.com/cline/cline/pull/10210) | Remove `/deep-planning` built-in slash command | [PR-10210.md](cline-cline/PR-10210.md) |
| [#10254](https://github.com/cline/cline/pull/10254) | fix: use deterministic keys for MCP server tool routing | [PR-10254.md](cline-cline/PR-10254.md) |
| [#10266](https://github.com/cline/cline/pull/10266) | Fix cache reflection for Cline and Vercel handlers | [PR-10266.md](cline-cline/PR-10266.md) |
| [#10269](https://github.com/cline/cline/pull/10269) | fix: unblock stuck command_output ask when terminal command ends | [PR-10269.md](cline-cline/PR-10269.md) |
| [#10283](https://github.com/cline/cline/pull/10283) | feat: wire up remote globalSkills with enterprise UI and architectural fixes | [PR-10283.md](cline-cline/PR-10283.md) |
| [#10329](https://github.com/cline/cline/pull/10329) | feat(cost-control): enforce third-party API spend limits | [PR-10329.md](cline-cline/PR-10329.md) |
| [#10286](https://github.com/cline/cline/pull/10286) | feat(models): prepare Claude Opus 4.7 provider support | [PR-10286.md](cline-cline/PR-10286.md) |
| [#10291](https://github.com/cline/cline/pull/10291) | fix: stabilize flaky Windows CI test paths | [PR-10291.md](cline-cline/PR-10291.md) |
| [#10343](https://github.com/cline/cline/pull/10343) | feat(memory-observability): add periodic memory logging to cline-core | [PR-10343.md](cline-cline/PR-10343.md) |

## modelcontextprotocol/servers

| PR | Title | File |
|---|---|---|
| [#3297](https://github.com/modelcontextprotocol/servers/pull/3297) | fix(memory): correct openNodes/searchNodes edge semantics | [PR-3297.md](modelcontextprotocol-servers/PR-3297.md) |
| [#3434](https://github.com/modelcontextprotocol/servers/pull/3434) | fix(filesystem): normalize bare drive letters on Windows | [PR-3434.md](modelcontextprotocol-servers/PR-3434.md) |
| [#3545](https://github.com/modelcontextprotocol/servers/pull/3545) | fix(git): guards against arg injection in git operations | [PR-3545.md](modelcontextprotocol-servers/PR-3545.md) |
| [#3890](https://github.com/modelcontextprotocol/servers/pull/3890) | feat: Add compare_directories tool for directory comparison | [PR-3890.md](modelcontextprotocol-servers/PR-3890.md) |
| [#3922](https://github.com/modelcontextprotocol/servers/pull/3922) | fix(fetch): fall back when Readability strips hidden SSR content | [PR-3922.md](modelcontextprotocol-servers/PR-3922.md) |
| [#3959](https://github.com/modelcontextprotocol/servers/pull/3959) | feat(memory): add lightweight read modes for scalability | [PR-3959.md](modelcontextprotocol-servers/PR-3959.md) |
| [#3262](https://github.com/modelcontextprotocol/servers/pull/3262) | test(fetch): add unit tests for fetch MCP server | [PR-3262.md](modelcontextprotocol-servers/PR-3262.md) |
| [#3515](https://github.com/modelcontextprotocol/servers/pull/3515) | fix: don't crash on malformed JSON-RPC frames (raise_exceptions=False) | [PR-3515.md](modelcontextprotocol-servers/PR-3515.md) |
| [#3533](https://github.com/modelcontextprotocol/servers/pull/3533) | fix(mcp): unify JSON-Schema → Zod conversion behavior across surfaces | [PR-3533.md](modelcontextprotocol-servers/PR-3533.md) |

## openai/codex

| PR | Title | File |
|---|---|---|
| [#18868](https://github.com/openai/codex/pull/18868) | Add MITM hooks for host specific HTTPS request clamping | [PR-18868.md](openai-codex/PR-18868.md) |
| [#19030](https://github.com/openai/codex/pull/19030) | Support to add prefetched tool result to a user turn | [PR-19030.md](openai-codex/PR-19030.md) |
| [#19031](https://github.com/openai/codex/pull/19031) | Fix relative stdio MCP cwd fallback | [PR-19031.md](openai-codex/PR-19031.md) |
| [#19046](https://github.com/openai/codex/pull/19046) | exec-server: require explicit filesystem sandbox cwd | [PR-19046.md](openai-codex/PR-19046.md) |
| [#19058](https://github.com/openai/codex/pull/19058) | Add /auto-review-denials retry approval flow | [PR-19058.md](openai-codex/PR-19058.md) |
| [#19086](https://github.com/openai/codex/pull/19086) | app-server: include filesystem entries in permission requests | [PR-19086.md](openai-codex/PR-19086.md) |
| [#19033](https://github.com/openai/codex/pull/19033) | Fix MCP permission policy sync | [PR-19033.md](openai-codex/PR-19033.md) |
| [#19038](https://github.com/openai/codex/pull/19038) | feat: warn and continue on unknown feature requirements | [PR-19038.md](openai-codex/PR-19038.md) |
| [#19129](https://github.com/openai/codex/pull/19129) | Reject agents.max_threads with multi_agent_v2 | [PR-19129.md](openai-codex/PR-19129.md) |
| [#19204](https://github.com/openai/codex/pull/19204) | fix(exec-server): honor request timeout across tool loop iterations | [PR-19204.md](openai-codex/PR-19204.md) |
| [#19130](https://github.com/openai/codex/pull/19130) | exec-server: wait for close after observed exit | [PR-19130.md](openai-codex/PR-19130.md) |
| [#19127](https://github.com/openai/codex/pull/19127) | feat: drop spawned-agent context instructions | [PR-19127.md](openai-codex/PR-19127.md) |
| [#19231](https://github.com/openai/codex/pull/19231) | permissions: make profiles represent enforcement | [PR-19231.md](openai-codex/PR-19231.md) |
| [#19247](https://github.com/openai/codex/pull/19247) | chore: apply truncation policy to unified_exec | [PR-19247.md](openai-codex/PR-19247.md) |
| [#19261](https://github.com/openai/codex/pull/19261) | Resolve relative agent role config paths from layers | [PR-19261.md](openai-codex/PR-19261.md) |
| [#19244](https://github.com/openai/codex/pull/19244) | Update unix socket transport to use WebSocket upgrade | [PR-19244.md](openai-codex/PR-19244.md) |
| [#19283](https://github.com/openai/codex/pull/19283) | check PID of named pipe consumer | [PR-19283.md](openai-codex/PR-19283.md) |
| [#19170](https://github.com/openai/codex/pull/19170) | Skip disabled rows in selection menu numbering and default focus | [PR-19170-skip-disabled-rows-numbering.md](openai-codex/PR-19170-skip-disabled-rows-numbering.md) |
| [#19287](https://github.com/openai/codex/pull/19287) | Restore persisted model provider on thread resume | [PR-19287-resume-model-provider.md](openai-codex/PR-19287-resume-model-provider.md) |
| [#19184](https://github.com/openai/codex/pull/19184) | fix: handle deferred network proxy denials | [PR-19184-deferred-proxy-denials.md](openai-codex/PR-19184-deferred-proxy-denials.md) |
| [#19163](https://github.com/openai/codex/pull/19163) | Harden package-manager install policy | [PR-19163-package-manager-hardening.md](openai-codex/PR-19163-package-manager-hardening.md) |
| [#19334](https://github.com/openai/codex/pull/19334) | Fallback login callback port when default is busy | [PR-19334-login-port-fallback.md](openai-codex/PR-19334-login-port-fallback.md) |
| [#19282](https://github.com/openai/codex/pull/19282) | feat(hooks): add dynamic tool hook payloads | [PR-19282-dynamic-tool-hook-payloads.md](openai-codex/PR-19282-dynamic-tool-hook-payloads.md) |
| [#19160](https://github.com/openai/codex/pull/19160) | Make apply_patch streaming parser stateful | [PR-19160-apply-patch-stateful-streaming.md](openai-codex/PR-19160-apply-patch-stateful-streaming.md) |
| [#19308](https://github.com/openai/codex/pull/19308) | Surface reasoning tokens in exec JSON usage | [PR-19308-reasoning-tokens-exec-json.md](openai-codex/PR-19308-reasoning-tokens-exec-json.md) |
| [#19294](https://github.com/openai/codex/pull/19294) | Hide unsupported MCP bearer_token from config schema | [PR-19294-mcp-bearer-token-schema-hide.md](openai-codex/PR-19294-mcp-bearer-token-schema-hide.md) |
| [#19292](https://github.com/openai/codex/pull/19292) | Reject unsupported js_repl image MIME types | [PR-19292-js-repl-image-mime-allowlist.md](openai-codex/PR-19292-js-repl-image-mime-allowlist.md) |
| [#19233](https://github.com/openai/codex/pull/19233) | Make thread archive idempotent for archived threads | [PR-19233-thread-archive-idempotent.md](openai-codex/PR-19233-thread-archive-idempotent.md) |
| [#19354](https://github.com/openai/codex/pull/19354) | chore: alias max_concurrent_threads_per_session | [PR-19354-max-threads-alias.md](openai-codex/PR-19354-max-threads-alias.md) |
| [#19216](https://github.com/openai/codex/pull/19216) | Allow any granular approval requirement | [PR-19216-granular-approval-wildcard.md](openai-codex/PR-19216-granular-approval-wildcard.md) |

---

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.
