# Review Index

197 + W17 drips (through drip-88) PR reviews across 10 OSS AI-coding-agent projects. Each review
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
- **W17 drip-12 (2026-04-24)**: 8 more — auth/identity
  surface gaps and observability noise plus a reasoning-
  content contract drift across vendor surfaces. DeepSeek
  thinking-mode `reasoning_content` injection broadened
  beyond interleaved-only models so legacy/string-content
  assistant replays stop hitting "must be passed back"
  errors; a built-in `scout` subagent that adds
  `repo_clone`/`repo_overview` tools (raises lifetime/
  egress questions about the shared on-disk repo cache);
  a three-call-site Apps-MCP gate flip from "must be
  ChatGPT token" to `uses_codex_backend` so AgentIdentity
  JWT auth finally sees connector lists; a Windows
  process-management follow-up that bounds elevated-
  runner pipe-connect, kills orphaned `codex-command-
  runner.exe` on handshake failure, and loops partial
  `WriteFile` for stdin (with a tail-drain hang risk
  that needs a child-process-signaled escape); URL-mode
  MCP elicitation routing for Codex Apps auth failures
  through the existing TUI app-link view (with a
  forward-contract concern: the client now advertises
  URL elicitation to *all* MCP servers); a TUI cache-
  staleness fix that bypasses the width-keyed render
  cache while a turn is streaming so reasoning tokens
  actually appear, plus a previously-no-op "Hide/Show
  Thinking" toggle wired up (with no persistence);
  Moonshot / Moonshot China API-key resolution for
  `MOONSHOT*_API_KEY` and `KIMI*_API_KEY` env vars
  (delivery flagged: ships a `replace` to a personal-
  account catwalk fork that should block the merge);
  a 401/403 auth-denial logger downgrade from `ERROR`
  + traceback to `WARNING` to stop expected ACL events
  from drowning real outages (PR is bundled with ~8k
  unrelated lines and needs a split); and a single-team
  DB fallback in JWT auth that infers `team_id` when the
  user belongs to exactly one team (operator-mutable
  invariant — should be opt-in behind a flag, not
  default-on).
- **W17 drip-13 (2026-04-24)**: 8 more — replay-time
  rehydration of assistant `tool_call` parts from a
  `WithParts`-derived `replayToolCalls` map plus a
  send-time guard wrapped *twice* around
  `ProviderTransform.message` for the OpenAI-compatible
  NPM api (band-aid that masks an underlying transform
  drop); per-route `Authorization` middleware on every
  experimental HttpApi group with `auth_token` modeled
  as a query-string `apiKey` security scheme (per-route
  opt-in is a default-public footgun); `/uptime` slash
  command + `processStartTime()` field on
  `/global/health` (module-load capture, dialog renders
  static-at-open snapshot — text reads like live
  counter); `time.archived = null` symmetry on the
  PATCH `/session/:id` validator wired through web,
  TUI (`ctrl+a` toggle), and event reducer (no ACL
  test on the new write path); `session_id` /
  `thread_id` header split for multi-agent v2 spawn
  flows with `current_window_id` keyed off the child
  thread (no `debug_assert_ne!` on spawn invariant,
  reopen path still on old single-id semantics); outer
  tool `call_id` propagation into
  `_meta._codex_apps.call_id` for Codex Apps MCP server
  with the `unwrap_or_default()` refactor that now
  always emits the `_codex_apps` block (broadens
  backend "presence ⇒ context" contract); two bare
  `.(float64)` / `.(string)` type-assertion panics in
  `crush logs` replaced with comma-ok form (silent
  skip + `time.Now()` fallback that hides
  log-producer bugs); and `verbose_logger.setLevel(INFO)`
  parity in the `LITELLM_LOG=INFO` branch alongside the
  router/proxy loggers (volume blast radius for
  deployments that were unintentionally relying on
   `verbose_logger` staying quiet).
- **W17 drip-15 (2026-04-24)**: 8 more — edge-case
  error-handling gaps: a CI entitlements trim that
  silently strips `keychain-access-groups` (post-build
  keychain reads now miss with no test), thread-instruction
  override flags that unconditionally clobber existing
  params with `None` instead of merging, a re-auth flow
  fix that only fires for 401 (403/transport errors still
  swallowed), retry-log helper that emits `status_code: 0`
  for transport failures and risks log-collision on
  `"message"`, LSP unavailable-cache retry-window with a
  read-then-mutate window and no integration test guarding
  the call site, `python-dotenv` CVE bump with no .env
  parser regression test, "super yollo" classifier that
  is bypassable via `bash -c` / `eval` / wrappers and
  loses block-functions on the unconditional Start path,
  shared-health-check polling that closes the silent
  fallback hole but introduces a small cache-vs-lock
  reordering race; and a fixture-only `models.json` PR
  that silently flips `default_reasoning_level` from
  `medium` to `xhigh` (cost/latency contract change with
  no release-note surface).
- **W17 drip-16 (2026-04-24)**: 8 more — desktop sidecar
  bounded-retry health check that drops the loopback gate
  and unconditionally `no_proxy()`s every host (regresses
  non-loopback callers); session-scoped permission bridge
  via a `Symbol.for`-keyed `globalThis` Map for external
  providers with `Effect.acquireRelease` cleanup; npm
  update-prompt readiness gate that verifies the platform
  `optionalDependencies` entry + tarball before announcing
  a new version (with a `VersionSource`-tagged cache so
  switching install methods invalidates correctly); typed
  `ThreadStoreConfig` enum + debug-only in-memory store
  + a regression test that asserts no rollout/sqlite
  artifacts materialize when a non-local store is
  configured; `crush` `additional_dirs` +
  `restrict_to_project` deny-outright option that resolves
  symlinks on the **leaf** path (silently breaks
  Write/Edit for new files in allowlisted dirs) and only
  gates `bash`'s working dir, not what the shell does;
  1-second cross-process session-update polling for
  `crush` TUI gated on `!isAgentBusy()` (with a fragile-
  busy-stuck failure mode); JWT proxy_admin-acting-on-
  behalf-of-a-team rate-limit fix via reordering
  `get_team_id_from_header` above `check_admin_access`
  plus a `bypass_allowed_check` for admin-scope tokens
  whose JWTs don't list teams; and the trivially-correct
  Z.AI / Zhipu AI dropdown entry that closes the
  backend-complete / UI-incomplete gap from issue
  #25482.
- **W17 drip-17 (2026-04-25)**: 8 more — WSL desktop
  mDNS workaround docs (UNC-path bypass via hostname
  resolution), ACP slash-command parser rewritten to
  preserve internal whitespace via anchored
  `^(\S+)(?:\s([\s\S]*))?$` regex, bundled openai-docs
  skill refresh for GPT-5.5 (default-model flip,
  `latest-model.md` table refit, 599→244 line
  prompting-guide trim), `LogBatchWriter` trait carve-out
  with `BufferedLogSink<W>` + `LogSinkQueueConfig` so
  tracing log sinks can plug new destinations while
  preserving fire-and-forget back-pressure shape, OSC
  99 + OSC 777 dual-emit notification backend keyed off
  `SSH_TTY` (with a load-bearing
  `RequestModeFocusEvent` move out of the smart-terminal
  branch), per-agent `model` size config in crush
  config/schema with a schema/runtime-required mismatch
  on `model` and five exposed-but-unwired Agent fields,
  GCP IAM token cache (55-min TTL, double-checked
  locking, module-level `Dict` keyed by service account)
  to stop N parallel Redis connections from each
  blocking the asyncio loop on a fresh IAM round-trip,
  and the missing
  `cache_creation_input_token_cost_above_1hr` (6e-06)
  pricing entry on both `vertex_ai/claude-sonnet-4-6`
  and its `@default` alias.
- **W17 drip-18 (2026-04-25)**: 8 more — theme is
  *"opinionated defaults vs user override"* and
  *finishing migrations cleanly*. A hardcoded
  Bedrock model allowlist that silently shadows user
  whitelists and breaks ARN-based custom models;
  the matching docs PR that *documents* the exact
  pattern the allowlist breaks (custom Claude-named
  keys with application inference profile ARNs);
  Windows-Terminal `wt.exe` plumbing through both
  Electron and Tauri with a small file-vs-dir
  normalisation drift between the two; an
  on-paste media optimization layer that quietly
  pulls in `ffmpeg`/`sips`/`magick`/`shift-ai`
  detection and a new plugin API at the same time;
  a textbook subtractive PR removing the in-tree
  JS REPL across 63 files (-9244) with all seven
  seams (flag enum, handler, router, schema,
  fixtures, CI prereqs, third-party notice) closed;
  a Crush PreToolUse hook engine wired in *before*
  permission checks with a Claude-Code-compatible
  JSON wire protocol but Windows shell story
  unspecified; 3000 lines of cross-platform PTY +
  split-pane + tab-manager scaffolding with no
  in-tree consumers and a stub Unix backend; and
  the final pass of the codex `to_legacy_sandbox_policy`
  removal that quietly tightens trust semantics for
  `Managed` profiles whose write paths don't include
  `cwd`.
- **W17 drip-19 (2026-04-25)**: 8 more — codex's continuation
  of the `SandboxPolicy → PermissionProfile` core sweep
  (`ExecApprovalRequest` rewrite, `render_decision_for_unmatched_command`
  swap, Windows-managed-readonly helper extraction); a
  one-line phase-2 memories model bump that lacks any
  consolidation-quality eval and skips the matching
  fixture refresh other model bumps in this repo carry;
  `task(background=true)` + new `task_status` polling
  tool in opencode (TaskPromptOps gains `loop`/`fork`,
  but the synchronous "started" output uses the same
  `<task_result>` envelope as the async "completed"
  message, denying the model a state disambiguator);
  a 5-line opencode question-dock textarea height clamp
  fixing a footer-pushed-out-of-viewport regression in
  web/serve mode (`Math.min(scrollHeight, 120)` + matching
  `max-height`/`overflow-y` CSS); the new `crush skills
  list` command with `--flat`/`--json`/positional-search
  and source-grouped tree output, plus an `ClassifySource`
  primitive other commands can reuse; the litellm
  consolidation of four stalled community PRs that
  together fix the `output_config` drop on Vertex Claude
  + Anthropic adapter via a leaf `output_params_utils`
  module (Vertex-unsupported keys filtered, supported
  bits forwarded); ollama Qwen-family renderer tool-payload
  XML escaping (idempotent entity-aware `escapeQwenXMLText`)
  paired with a Qwen 3.5 truncation atomicity fix that
  treats assistant-tool-call + contiguous tool messages
  as one drop unit; and an ollama Metal-tensor-API runtime
  retry path (`ShouldRetryWithMetalTensorDisabled` +
  `RunnerEnvOverrides` persistence) bundled with a
  GPU-discovery dummy-load serialization fix and a
  `StatusWriter` race repair via shared `cmd.Stdout`/`cmd.Stderr`.
- **W17 drip-20 (2026-04-25)**: 8 more — codex's continued
  `PermissionProfile` migration sweep (`from_legacy_sandbox_policy`
  becomes cwd-free / symbolic with a separate `_for_cwd`
  materializer for runtime callers; analytics + app-server
  consumers re-keyed to `permission_profile`/`permission_profile_cwd`
  with `Disabled` and `External` finally distinguishable in the
  `sandbox_policy_mode` analytics bucket); the `thread/turns/list`
  endpoint cut over from a four-fallback rollout-path resolver
  to `ThreadStore::read_thread(... include_history: true)` with
  a `thread_store_override` test seam threaded through
  `InProcessClientStartArgs`; usage-limit popup `Shown`/`CtaClicked`
  analytics emitted from the TUI via a new
  `AppEvent::TrackUsageLimitBanner` plus extended status-and-layout
  test coverage for the funnel; an event-driven rewrite of the
  `plugin_mcp_tools_are_listed` flake (helpers now return the
  full `TestCodex` so the `TempDir` cwd guard outlives stdio MCP
  startup, and the 30s polling loop becomes a single
  `McpStartupComplete` wait); CamelHumps subword editing on
  `ctrl+left/right/backspace/delete` in crush with `parseHTTPResponse`
  / `foo_barBaz` segmentation tests but missing `WithHelp` text
  for the new bindings; a 4307-line crush JSON-hooks compatibility
  PR that ships an unreleasable `replace charm.land/fantasy =>
  ../../fantasy` directive in `go.mod`, double-fires the
  `PromptSubmit` hook on first turns, and runs arbitrary shell
  commands with no timeout/allowlist/trust-model documentation —
  request-changes, not nits; and a tight opencode `.zod.parse`
  → `Schema.decodeUnknownSync` migration that hoists decoders
  to module scope across `acp/agent.ts`, `cli/cmd/import.ts`,
  `control-plane/adaptors/worktree.ts`, and `server/routes/instance/pty.ts`
  while keeping behavior identical except for malformed-JSON
  todo-output now landing in the `Result.isFailure` log branch
  instead of crashing the message-decoder loop.
- **W17 drip-21 (2026-04-25)**: 8 more — codex's `app-server`
  JSON-schema cleanup that strips sandbox-access defaults so
  generated TypeScript clients stop pinning stale fallbacks
  (merge-after-nits: needs an explicit `#[serde(default)]`
  audit on the consumer side); the third PR in the
  `PermissionProfile` migration arc, which finally derives the
  legacy `SandboxPolicy`/`AskForApproval` compat shims from
  profile data instead of duplicating the table (good direction,
  flagged a missing round-trip test for the
  `Disabled`/`External` distinction); a fresh `agent-graph-store`
  crate skeleton with `GraphStore` trait + in-memory impl that
  reads cleanly but lacks any concurrency story or persistence
  shim — merge-after-nits with a request to document the
  intended consumer; a crush SQLite fix that caps
  `MaxOpenConns(1)` to dodge `NOTADB` corruption under WAL
  contention, flagged as `needs-discussion` because it directly
  competes with already-reviewed #2690 (busy-timeout +
  `_journal_mode=WAL` PRAGMA approach) and the two cannot both
  land; a crush "session export" PR whose title promises a new
  subcommand but the diff actually adds `--markdown`/`-o` flags
  to existing `session show`/`session last` — request-changes
  for the title/scope mismatch and missing tests; a litellm
  SSO-callback fix that surfaces OAuth `error`/`error_description`
  query params back to the user instead of swallowing them into
  a generic 500 (merge-after-nits: PR title literally "Fix: add
  fix to auth error" needs a rewrite, and the error string isn't
  HTML-escaped before being rendered); and an ollama download
  stall watchdog fix where the goroutine that resets the
  `lastProgress` timestamp was never being scheduled when zero
  bytes arrived in the first interval — clean one-line fix,
  merge-as-is.
- **W17 drip-22 (2026-04-24)**: 8 more — codex's
  `PermissionProfile` promotion to canonical runtime field with
  paired `set_legacy_sandbox_policy`/`set_permission_profile`
  mutators that keep all four projection fields coherent
  (merge-after-nits: inline reconstruction in
  `codex_message_processor.rs` should call
  `Permissions::permission_profile()` instead of duplicating the
  logic, plus a round-trip property test); a bundled-skill
  upgrade-guide editorial pass that loses the
  `parallel_tool_calling`-only-when-independent caveat
  (merge-after-nits to restore it); two opencode fixes — DeepSeek
  V4 empty `reasoning_content` preservation across non-streaming
  + streaming paths with a clean split between `"reasoning_text"
  in delta` (start gate) and truthy-content (delta gate), and a
  question-dock overflow fix that replaces a global
  `document.querySelector(".scroll-view__viewport")` with a
  scoped panel-rooted traversal; a crush theming system that
  ships `Charmtone`/`Gruvbox Dark` plus a 30-slot
  `ThemePalette` and reflective struct-sync test
  (merge-after-nits: silent fallback for unknown theme names is
  the worst UX, want a startup warning); a litellm one-line
  duplicated-three-times fix that adds `verbose_logger` to the
  `LITELLM_LOG=INFO` branch (merge-as-is, plus a coordination
  note that #26397/#26401/#26429 are the same patch); a litellm
  Anthropic reasoning-family `temperature` drop fix whose 7-line
  semantic change is buried in a 2,096-line full-file rewrite
  (request-changes pending a clean reroll + committed test
  matrix); ollama's missing repeat/frequency/presence penalty
  implementation in the Go-native sampler with the asymmetric
  positive/negative-logit branch matching llama.cpp's reference
  (merge-after-nits to verify default `repeat_penalty=1.1` flows
  through `req.Options` and to preallocate the ring buffer); and
  a crush MCP `createSession` swap from `time.AfterFunc` +
  `WithCancel` to `WithTimeout` that finally lets `maybeTimeoutErr`
  distinguish deadline from cancel (needs-discussion: deleted
  `TestMCPSession_CancelOnClose` is the only defense against the
  mcp-go-sdk `ClientSession` retaining the connect context past
  `Connect`, want explicit confirmation before the wrapper
  removal lands).
- **W17 drip-24 (2026-04-25)**: 8 more — Guardian routing for
  opted-in MCP elicitations (per-server allowlist + structured
  request/response on stdin/stdout, blocked-by-default), OTel
  `gen_ai.usage.input_tokens`/`output_tokens` attributes added to
  the per-turn span (cardinality contained, sums-only), Windows
  `unified_exec` opt-in via `experimental_use_unified_exec=true`
  (rollback risk on the WinPTY → ConPTY path flagged),
  in-process `opencode run` server picking up basic-auth header
  parity with the daemon path, qdrant semantic cache JSON
  serialization fix + multimodal `content` array passthrough,
  streaming content-filter post-call now logs
  `guardrail_information` symmetrically with the non-streaming
  branch, `/api/tags` enrichment with parameter-size for
  safetensors models to close a `/api/show` parity gap, and
   `ollama stop`/`ollama show` printing usage instead of erroring
   when called with no model argument.
- **W17 drip-25 (2026-04-25)**: 8 more — codex retiring legacy
   read-only access modes from the permissions surface, codex CI
   publishing `codex-app-server` release artifacts alongside the
   main CLI binary, opencode session preserving the shell `cwd`
   across startup so the first prompt lands in the directory the
   user launched from, opencode adding a `/context` slash command
   that prints token-budget breakdown by category, litellm
   honoring `reasoning_effort=minimal` on gpt-5.5 (was being
   silently dropped by the parameter mapper), litellm CircleCI
   rerun awk preprocessor fixed for jobs whose names contain
   colons, ollama bumping mlx to 0.31.2 for the Metal kernel
   regression fix, ollama desktop sidebar no longer animating
   open on initial app load, and an ollama launch flow that
   generates `~/.codex/model.json` ahead of spawning Codex so
   per-model context-window/modality/reasoning metadata flows
   through instead of falling back to generic defaults.
- **W17 drip-26 (2026-04-25)**: 8 more — codex CI stabilisation
  pair (extracted shared `wait_for_sample_mcp_ready` helper for
  plugin MCP fixture flakes, plus split of the monolithic
  `approval_matrix_covers_all_modes` test into five focused
  per-scenario-group tests with `with_context` for failure
  diagnostics); opencode auto-enabling
  `interleaved={field:"reasoning_content"}` when a model has
  `reasoning:true` and no explicit interleaved capability set
  (one-line provider.ts fix; closes a silent reasoning-content
  drop on message replay); opencode reverting the recent
  "wildcard rules sort first" canonicalisation in
  `Permission.fromConfig` and replacing `StructWithRest` with a
  hand-rolled `InfoZod` parser via `ZodOverride` annotation, so
  permission precedence now follows literal user key order
  (semantic flip flagged as needs-discussion — existing configs
  with `bash:"allow"` after `*:"deny"` change behaviour); litellm
  reseeding spend enforcement from authoritative DB on
  Redis+memory counter miss (closes a multi-pod budget bypass
  where caller's stale in-process `team_membership.spend` was
  letting requests through; `db_spend > 0` gate semantics
  flagged); litellm Azure Sentinel always-on gzip + 950 KB batch
  splitting plus opt-in `AZURE_SENTINEL_TRUNCATE_CONTENT`
  column-level truncation with tail preservation and
  `litellm_content_truncated` metadata (request-changes:
  `len(str(...))` measures Python repr not JSON serialisation,
  no tests, doc duplication); ollama refcounted `caffeinate`/
  `systemd-inhibit` power lock around Generate/ChatHandler to
  prevent idle sleep during inference (request-changes:
  `sh -c "...&"` orphans the PID forcing `pkill -f` which kills
  user-owned inhibitors, Windows silently unsupported, dead
  `isActive` flag); and ollama F8_E4M3 safetensors import with
  block-scale companion pairing, hard-fail on missing
  block-size or scale tensor, and source-precision metadata
  propagated to the quant pipeline so default Q8_0 / mixed
  Q4_K decisions can be made FP8-aware (needs-discussion: the
  actual `decodeFP8E4M3` math + E4M3FN/FNUZ convention +
  scale-vs-scale_inv interpretation is not visible in the
  reviewable diff window).
- **W17 drip-23 (2026-04-25)**: 8 more — tool-description prompt
  diet (~66% char cut across 17 `*.txt` files in opencode, with
  open questions on bash policy relocation and per-provider cache
  audit), Polish translation polish for the opencode console
  (mixed register flagged + `error.roleRequired` semantic shift
  from "Rola" to "Stanowisko"), codex network-proxy prompt
  guidance unconditionally appended to `PermissionsInstructions`
  (gating + `NO_PROXY` mention + named permission-request tool
  flagged), crush chroma formatter extracted into shared
  `internal/ui/xchroma` package so markdown code blocks match
  diffview's foreground+background style (global registry name
  collision risk noted), litellm qdrant semantic cache `kwargs.
  get("messages") or kwargs.get("message")` defensive access plus
  `get_str_from_messages` helper for multimodal content blocks
  (tests cover list-response but not the actual bug-fix branches),
  litellm bedrock guardrail `try/except HTTPException` wrapping
  the entire streaming hook with in-band SSE error frame
  conversion (`HTTPException`-only catch leaves bare-Exception
  leak class intact, two near-identical tests parametrizable),
  Model Armor docs update adding `during_call` to the supported
  modes (the `post_call` description over-corrects — runtime
  back-fills input validation when no pre/during is configured),
  and ollama runner surfacing `prompt_cached_count` as a new
   `,omitempty` field on `CompletionResponse` (4-line additive
   change, snapshot-timing comment + OpenAI-compat layer wiring
   flagged as follow-ups).
- **W17 drip-28 (2026-04-25)**: 8 more — codex routing MCP
   elicitations through the existing Guardian review pipeline so
   server-initiated prompts inherit the same allow/deny ledger as
   tool calls (follow-up to drip-24's #19431; 23 files, schema
   wiring + UI surface flagged); a second gRPC feedback log sink
   crate landing as a standalone subscriber (1121-line additive,
   no shared abstraction with the earlier #19455 sink — dedup
   opportunity); the python SDK switching to a standalone
   `codex-app-server` runtime instead of in-process spawn
   (follow-up to drip-25's #19447; lifecycle teardown + Windows
   path handling flagged); Bedrock GPT-5.4 reasoning levels
   collapsing `xhigh` into `high` to match upstream's removed tier
   (config-only, but silent behaviour change for users who pinned
   `xhigh`); flipping `UnavailableDummyTools` from experimental to
   stable + default-on so missing-MCP scenarios degrade to typed
   stubs instead of hard failures (3-line default flip, telemetry
   for stub invocations flagged); litellm CircleCI awk preprocessor
   that rewrites `rerun-failed-tests` payloads so `local_testing`
   suite paths match the rerun matcher (3-line awk substitution,
   no test for the awk itself); litellm APScheduler job that purges
   expired UI session keys from the dashboard auth table (437-line
   addition, RBAC for the cleanup endpoint + tz-aware expiry
   comparison flagged); and ollama recognising mlx-lm's plural
   `quantization_aux_*` sibling-tensor naming at quant-metadata
   load time (supersedes #15743; 7-file refactor with regression
   test for both singular and plural conventions).
- **W17 drip-30 (2026-04-25)**: 9 more — codex removing
   ghost reasoning snapshots from Responses API turns
   (output-array trim that's correct semantically but
   changes rollout JSON wire-format — replay-tooling
   risk flagged); codex gating Windows sandbox tests
   behind `serial_test` to stop registry-key contention
   under cargo's parallel runner (test-only, clean);
   codex ChatGPT Library file upload/download adding a
   streaming multipart parser + new SDK signatures with
   thin test coverage and one breaking method rename
   (needs-discussion); litellm hardening pass-through
   target URL construction so user-supplied paths can't
   escape the configured base via `..` / scheme override
   (urllib.parse.urljoin replaced with explicit prefix
   join + suffix validation, good test matrix); litellm
   wiring `gpt-5.5` reasoning_effort flags + new
   `supports_low_reasoning_effort` capability flag through
   the model-cost JSON and capability resolver
   (additive); opencode detecting non-object frontmatter
   from gray-matter and surfacing a typed error instead
   of a downstream `Object.keys` crash (1-line guard +
   regression test); opencode lazy session error schema
   via `Schema.suspend` to break a circular import that
   was forcing eager evaluation at module load
   (Effect-Schema idiom, no behavior change); opencode
   per-model reasoning token pricing — retires the
   long-standing TODO that billed reasoning at the output
   rate, adds optional `reasoning` field through 4 cost
   schemas + the user-config merge, with both explicit
   and fallback test cases (textbook backward-compat
   shape); and crush adding inline tooling notes for
   git/gh in CONTRIBUTING (docs-only but the suggested
   `git rebase --continue --no-edit` flag combo is wrong
   — `--no-edit` isn't a rebase flag, it belongs on the
   inner commit — merge-after-nits).
- **W17 drip-32 (2026-04-25)**: eight reviews split
   across openai/codex (two slices of pakrym-oai's
   handler-streamlining chain — #19498 and #19484
   fold long if-let-Err ladders into single
   `async {}` blocks dispatching one `Result`, plus
   #19497 doing the same for the turn/realtime entry
   points; all three preserve analytics-emit ordering
   and are merge-after-nits on style-consistency
   grounds), BerriAI/litellm (#26474 collapses
   Bedrock guardrails' duplicate INPUT/OUTPUT
   asyncio.gather post-call into a single pass —
   merge-after-nits; and #26471 introduces per-
   (team, member, model) budget enforcement with a
   new `LiteLLM_TeamMemberModelSpend` table — the
   migration story is honest but the writer-side
   cache key (`team_id::...::user_id::...::model::...`)
   doesn't appear to bridge to the reader-side cache
   key (`spend:team_member:...:model:...`), so 429s
   may only fire on TTL flips rather than on the
   next request — request-changes), All-Hands
   OpenHands (#14125 hardens Bitbucket DC webhook
   parsing with `(x or {})` and isinstance guards —
   clean merge-as-is; and #14127 cuts resolver
   comment noise by editing the original "I'm on it"
   ack in place via a hidden HTML marker, with sane
   fallback to a fresh comment and a widened
   non-fatal-status set including 423/429 —
   merge-after-nits pending confirmation that the
   marker actually lands on the posted comment), and
   Aider-AI/aider (#5065 attempts to fix the
   Architect-mode prompt-injection issue #5058 by
   asking the same model to self-classify its own
   proposal as SAFE/UNSAFE, but same-model self-
   validation isn't a trust boundary — the injected
   payload travels intact into the validator prompt;
   plus the check fails open on litellm exceptions,
    so the most common error path silently bypasses
    the control — request-changes).
- **W17 drip-33 (2026-04-25)**: eight reviews across
   five repos. openai/codex contributes two more
   slices of pakrym-oai's handler-streamlining chain
   (#19496 MCP handlers — clean merge-as-is, and
   #19493 thread mutation handlers — merge-after-nits
   on test-coverage callout grounds; both extract
   `Result`-returning helpers and funnel through
   `send_result`). BerriAI/litellm #26484 hardens
   master-key auth by aliasing the master key once at
   the auth boundary so neither the raw key nor its
   hash propagates to spend logs / Prometheus labels
   / audit trails — merge-after-nits pending a
   changelog callout for the silent dashboard-filter
   migration. anomalyco/opencode contributes three:
   #24232 fixes DeepSeek/Moonshot cache-token double-
   counting by preferring `noCacheTokens` when
   present (clean merge-as-is, two sharp tests);
   #24246 preserves nix/direnv PATH across login-
   shell rc sourcing but silently flips precedence
   for all users (request-changes — needs a
   `IN_NIX_SHELL`/`DIRENV_DIR` gate or config flag);
   #24244 removes the wildcard-first sort in
   permission config and ships a "regression" test
   that codifies the new footgun where a trailing
   `*` rule overrides earlier specifics (needs-
   discussion — empty PR body, undocumented policy
   reversal). All-Hands OpenHands #14122 wires
   `TaskToolSet` behind a per-user `enable_sub_agents`
   bool but lacks defensive `getattr` for the SDK-
   pin gap and any tests (needs-discussion). #14105
   correctly fixes the empty-text guard to permit
   attachment-only chat submissions
   (merge-after-nits pending a 4-case unit test).
   Aider-AI/aider #5033 adds a truncation-detection
   heuristic that skips auto-commit, but the 92%
   threshold is unjustified, the prefill-model
   exclusion is too broad, and the `auto_commit`
    return-type change risks misinterpretation
    downstream (request-changes).
- **W17 drip-34 (2026-04-25)**: eight reviews across
   seven repos. openai/codex #19457 splits the thread-
   list state-DB handle into a two-tier abstraction
   (`completed` for paging, `exact_lookup` for per-id
   overlay) so filesystem-discovered threads still get
   git/branch/worktree fields populated when SQLite
   backfill hasn't completed (merge-after-nits — rename
   the lingering `fill_missing_*` test fn for grep
   sanity). BerriAI/litellm #26485 onboards FuturMix as
   a named OpenAI-compatible provider via a 9-line JSON
   registry entry, but ships no model_prices rows and
   no fixture, so budget tracking will silently log $0
   on first use (merge-after-nits — flag the pricing
   gap). anomalyco/opencode contributes two: #24250
   completes the DeepSeek `reasoning_content` round-
   trip by (a) defaulting `interleaved` to the field-
   injection path for any model with `reasoning`,
   (b) widening the deepseek sniff from `api.id` to
   also match `model.id` so OpenRouter routing works,
   and (c) running an unconditional second pass that
   injects empty `reasoning_content` on historical
   assistant turns loaded from the DB before reasoning
   was enabled (merge-after-nits — comment the second-
   pass override intent and add a snapshot test);
   #24251 ships a project run-configs feature
   (detect from package.json/Makefile/Cargo/Maven/
   Gradle/Go markers, scope-prefix monorepo entries,
   surface in session-header dialog), but the scan-
   then-one-click-run flow is an arbitrary-command
   surface against freshly-cloned untrusted repos and
   needs a "scanned vs accepted" trust gate (needs-
   discussion). charmbracelet/crush #2671 drops the
   `fetch`/`view` tool ceilings from 1MB to 100KB via
   six lines in `fetch.go`/`view.go` plus prose
   updates and a 7600-line cassette regen (merge-as-is;
   note that `fetch` lacks a pagination escape hatch
   so a follow-up offset knob would help). ollama
   #15713 fixes the cgroup v2 free-memory underestimate
   by adding `inactive_file` from `memory.stat` as a
   reclaimable adjustment (`FreeMemory = Total - used +
   min(inactive_file, used)`), with a path-injectable
   parser and three-row table test; v2-only and could
   tighten further with `slab_reclaimable` (merge-
   after-nits). All-Hands OpenHands #14104 restores the
   localized `ACTION_MESSAGE$THINK` fallback by
   detecting the SDK's default `think: {...}` summary
   via a narrowly-scoped regex and returning `null` so
   the upstream caller picks the localized label;
   ships positive + negative tests and links the
   tracking issue (merge-as-is). Aider-AI/aider #5024
   captures the `len(errors) < len(uniq)` predicate
   *before* joining `errors` to a string, fixing a
   silent mutate-then-test bug where the "other hunks
   applied" message was suppressed in exactly the
   cases it should have fired (merge-as-is — though a
   regression test would lock it down).
- **W17 drip-35 (2026-04-25)**: eight reviews across
   four repos — first appearance of continuedev/continue
   in the index. Aider-AI/aider #5052 adds bash/shell
   tree-sitter repomap support via a 6-line
   `function_definition` capture query plus a fixture
   covering both POSIX `name() {}` and `function name()
   {}` styles (merge-after-nits — verify `.bash`/`.zsh`
   actually route to the bash parser in the extension
   map). Aider-AI/aider #5066 appends a polyglot-
   leaderboard YAML row claiming new #1 at 93.3% on
   Claude Opus 4.7 with adaptive thinking; arithmetic
   on `pass_num/total_tests` checks out but
   `total_cost: $26.27` is suspiciously low for ~3.9M
   tokens at standard Bedrock pricing — flag for
   author confirmation before merging as new top spot
   (merge-after-nits). cline/cline #10376 closes a
   serious path-traversal hole in `ClineIgnoreController.validateAccess`
   (default no-`.clineignore` previously allowed
   `../../etc/passwd`; catch block returned `true` on
   `path.relative` errors with a literal "we are
   allowing access to all files outside cwd" comment),
   plus a companion fix to `readIncludedFile` for
   `!include ../../etc/passwd`-style abuse, with four
   new test cases including the no-clineignore default-
   deny regression (merge-after-nits — defense-in-depth
   `realpath` for symlinks; lockfile churn should be
   split). cline/cline #10377 reshapes the Plan/Act
   mode picker as a proper `role="radiogroup"` with
   roving tabindex, Space/Enter/arrow handling, and a
   webview test (merge-after-nits — note the switch→radio
   semantic downgrade in release notes). cline/cline
   #10384 adds `maxRetryAfter` (default 60s) to the
   `withRetry` decorator so multi-hour Gemini quota
   `retry-after` headers throw immediately instead of
   silently waiting; updates the synthetic-timestamp
   test to use `sinon.stub(Date, "now")` so the
   pre-existing case still asserts header-takes-
   precedence under the new default cap (merge-as-is;
   nit: rename `maxRetryAfter` → `maxRetryAfterMs` for
   unit clarity). cline/cline #10386 adds
   `whitespace-normal` to the expanded `ThinkingRow`
   button so multi-line reasoning content can wrap —
   the inner span's `whitespace-pre-wrap` was being
   defeated by the design-system button's default
   `whitespace-nowrap`; targeted regression test asserts
   both class presence and absence (merge-as-is).
   continuedev/continue #12190 moves the Gemini
   `fetchModels` API key from `?key=` URL query param
   into the `x-goog-api-key` request header, closing a
   key-in-access-logs / Referer leak primitive that
   matches the auth pattern already used elsewhere in
   the codebase, bundled with a CI-unblock model swap
   from a deprecated Anthropic haiku ID (merge-as-is).
   continuedev/continue #12198 removes
   `model.name.split("/").pop()` from `IntroMessage`
   so user-configured display names like `local/large`
   render in full instead of being trimmed to `large`;
   updates one existing test and adds a `local/large`
   regression case (merge-as-is).
- **W17 drip-36 (2026-04-25)**: eight reviews across
   four repos. openai/codex #19506 refreshes AGENTS.md
   per-turn against the effective cwd and emits a
   diff-gated user-instructions ContextualUserFragment
   carrying the new directory; the snapshot test locks
   in that turn-2 carries both original and refreshed
   blocks (merge-after-nits — equality check on
   `Option<&Arc<String>>` may false-negative; per-turn
   I/O cost worth a stat-cache). openai/codex #19495
   pushes thread-resume/fork helpers to return
   `Result<_, JSONRPCErrorError>` so `send_error` lives
   only at the request boundary, hoists the
   rollout-vs-history fork into a single match, and
   preserves error-message strings verbatim
   (merge-as-is, net -123 lines). openai/codex #19492
   does the same for `thread/start`, wrapping
   load-config + project-trust-derivation in a
   `let result = async {…}.await?` and adding a
   correct in-memory CLI-overrides fallback when
   on-disk trust persistence fails so requested
   sandbox isn't silently downgraded (merge-as-is;
   note divergence between in-memory and on-disk trust
   state in the failure-fallback path). BerriAI/litellm
   #26489 closes a credential-disclosure bug where
   `LiteLLM_ManagedVectorStore.litellm_params` (OpenAI
   `api_key`, AWS keys, `vertex_credentials`, etc.)
   was returned verbatim from `/vector_store/list`,
   `/info`, and `/update`; redaction goes through a
   new `_redact_sensitive_litellm_params` helper backed
   by `SensitiveDataMasker` (which gains the plural
   `"credentials"` key — without it Vertex would have
   slipped past), and `update` now requires per-store
   access via `_fetch_and_authorize_vector_store`
   (merge-after-nits — confirm `REDACTED_BY_LITELM_STRING`
   constant name, log on unexpected non-dict shape).
   BerriAI/litellm #26486 closes three independent
   privilege-escalation primitives in the key-mint /
   pass-through-route paths: a non-admin can no longer
   request `key_type='management'` (which expanded to
   `allowed_routes=['management_routes']` *after* the
   raw-field gate ran), can no longer set
   `metadata.allowed_passthrough_routes` (which is
   consumed by the route check *before* the role gate),
   and the pass-through route check itself now requires
   the route to be a registered pass-through endpoint
   via `InitPassThroughEndpointHelpers.is_registered_pass_through_route`
   — preventing internal admin routes like
   `/cache/settings` from being smuggled in through
   metadata. Docstrings explain *why* the prior gates
   missed each case, which is unusually good for
   maintainers (merge-after-nits — confirm both
   `/key/generate` and `/key/regenerate` invoke the
   new helpers; cache the lazy import). ollama/ollama
   #15735 introduces a content-addressed `manifest-v2/`
   layout where manifests are themselves blobs in
   `blobs/sha256-<hex>` and named-tag paths become
   relative symlinks (preferred), hardlinks, or
   byte-copies depending on FS support;
   `RemoveLayers` correctly extends the in-use map to
   include other manifests' `BlobDigest()` so
   multi-tag-to-one-manifest GC stays sound
   (needs-discussion — rollback story across
   binary downgrades, GC-vs-symlink invariant under
   crash, and `x/imagegen/manifest/` duplication
   deserve explicit answers). cline/cline #10396 makes
   the model-config "Supports images" toggle gate
   paste and drag-drop in `ChatTextArea.tsx` (not
   just file-picker), preventing wasted 3-retry cycles
   into text-only models; `useMemo` keyed correctly on
   `[apiConfiguration, mode]` and the `useCallback`
   deps array correctly adds `supportsImages`
   (merge-as-is). cline/cline #10385 fixes the Claude
   Code adapter for CLI v2.1+ which omits the `result`
   string property on success and emits
   `subtype: "error_max_turns"` for budget exhaustion;
   the `"result" in chunk` guard is dropped, the
   max-turns subtype is treated as normal end-of-turn
   (not a hard error), and error fallback uses
   `chunk.result ?? JSON.stringify(chunk)` for
   diagnostic preservation; new `_runClaudeCode` test
   injection seam typed via `Parameters<typeof …>`
   (merge-after-nits — `_runClaudeCode` leaks into
    public option type; confirm turn-loop expects a
    silent return on max-turns).
 - **W17 drip-37 (2026-04-25)**: eight reviews across
    four repos. openai/codex #19510 gates the rewind
    transcript overlay on a new `has_backtrack_target`
    predicate (cells must contain at least one
    `UserHistoryCell`) so a fresh-session double-`Esc`
    no longer opens an empty header-only view; the
    composer hint `show_esc_backtrack_hint` is
    suppressed in the same case and an info-event
    `No previous message to edit.` snapshots the
    user-visible signal (merge-as-is). openai/codex
    #19509 introduces a server→client MCP telemetry
    side-channel via `CallToolResult.meta["codex/telemetry"]["span"]`,
    allowlisting `target_id` (truncated to 256 chars
    via `truncate_str_to_char_boundary`) and
    `did_trigger_user_flow` (strict `as_bool`); the
    instrument-then-record pattern hands the span to
    `instrument(span.clone())` then calls
    `span.record(...)` after the future resolves
    (merge-after-nits — wire-format docs and a debug
    log on type-mismatch fall-throughs). openai/codex
    #19494 continues the
    `Result<_, JSONRPCErrorError>`-returning helper
    refactor for thread read/list/turn-list/summary
    handlers, collapsing per-error-arm `send_*` +
    `return` into a single `send_result` at the request
    boundary plus a `thread_read_view_error` mapper;
    net -44 LOC, no behavior diff (merge-as-is).
    BerriAI/litellm #26493 closes the residual surface
    on the `key_type='management'` escalation by
    flipping `_check_allowed_routes_caller_permission`
    to strict-by-default and threading
    `allow_safe_presets=True` only at the *post*-
    `handle_key_type` call site; also adds the missing
    `_check_allowed_routes_caller_permission` +
    `_check_passthrough_routes_caller_permission` pair
    to `/key/service-account/generate` and mirrors the
    post-handle re-check into `/key/regenerate` as
    defense-in-depth (merge-as-is). BerriAI/litellm
    #26490 drops `global_spend_tracking_routes` from
    `internal_user_routes` and
    `internal_user_view_only_routes` in `_types.py`
    to close a cross-tenant disclosure where
    `internal_user` keys could read proxy-wide spend;
    the test matrix is parametrized off
    `LiteLLMRoutes.global_spend_tracking_routes.value`
    so future additions are auto-covered, and three
    pre-existing tests that asserted the leak (the
    `True` expectation on `/global/spend/logs` for
    `internal_user`) are flipped to `False`
    (merge-as-is — flag as breaking access-control
    tightening with `permissions={"get_spend_routes": true}`
    as the documented opt-in). BerriAI/litellm #26488
    extends `/spend/logs/ui` `sort_by` whitelist with
    `model` and `ttft_ms`, with `ttft_ms` computed
    inline as
    `EXTRACT(EPOCH FROM (completionStartTime - startTime)) * 1000`
    and `NULLS LAST` so non-streaming rows always sink
    to the bottom regardless of direction; tests
    assert SQL contains `model` *and* does not contain
    `NULLS LAST` to guard the narrow special case
    (merge-after-nits — `# safe: order_column gated by
    valid_sort_fields` comment near the f-string).
    ollama/ollama #15790 is a one-line README
    integrations-list addition for Atlarix; positional
    slot is correct, marketing tone is restrained
    (merge-after-nits — verify link-liveness and
    consider deeper-link to an Ollama-setup docs page
    if available). cline/cline #10369 fixes the Ollama
    image transform in `ollama-format.ts` by stripping
    the `^data:[^;]+;base64,` prefix from
    Anthropic-style image blocks and pushing the bare
    base64 into a parallel `images: string[]` field on
    the Ollama message instead of inlining the data
    URI into `content` (merge-after-nits — confirm the
    tool-result branch's `toolResultImages` collector
    is actually attached to the outgoing wire message,
    tighten regex to `^data:image\/[^;]+;base64,`, and
    add a unit test pinning the wire shape).
  - **W17 drip-38 (2026-04-25)**: eight reviews across
     eight repos. openai/codex #19513 adds a 1-second
     composer-idle gate before any new approval modal
     via a `VecDeque<DelayedApprovalRequest>` queue
     and an `APPROVAL_PROMPT_TYPING_IDLE_DELAY` constant
     in `bottom_pane/mod.rs`, drained FIFO via
     `pop_front` + reverse-`pop_back` to preserve
     overlay display order; the activity filter
     (Press/Repeat, no CTRL/ALT, only
     Char/Backspace/Delete/Enter/Tab) lives at the
     boundary, and `cancel_approval_for_resolved_request`
     was extended to drop matching delayed entries
     (merge-after-nits). openai/codex #19490 continues
     the streamline-handlers refactor for
     plugin/apps/skills surfaces, lifting error
     construction into `internal_error`/`invalid_params`/
     `invalid_request` helpers and converting handlers
     to `Result<*Response, JSONRPCErrorError>` + single
     `send_result`; `send_marketplace_error` is deleted
     entirely (merge-as-is, -44% LOC on touched code).
     BerriAI/litellm #26491 lands the v0 Claude-Code
     compatibility matrix — CircleCI PR gate using
     `latest-minus-3-days` CLI plus a daily GH-Actions
     cron with always-latest CLI publishing JSON to
     `litellm-docs` via a `contents:write`-only GitHub
     App; trust split is real and third-party actions
     are SHA-pinned, but the 5604-LOC drop should be
     split per the PRD's #26477-#26481 slice IDs and
     the resolver step needs a
     `[ -n "$CLAUDE_CODE_VERSION" ]` guard
     (needs-discussion). sst/opencode #24258 bridges
     `/path`, `/vcs`, `/vcs/diff` through the
     experimental Effect HttpApi using a `withStatics`
     pattern that attaches a `.zod` static to each
     Effect Schema so Hono call sites keep working
     unchanged; `getVcs` runs `branch()`/`defaultBranch()`
     with `concurrency: 2` (merge-as-is). cline/cline
     #10397 adds an optional `lmStudioApiKey` plumbed
     through all eight Cline provider-key touch points
     (proto secret/config, SECRETS_KEYS, provider-keys
     map, handler, controller probe sending
     `Authorization: Bearer`, proto conversions both
     ways, UI ApiKeyField, configured-providers
     heuristic); handler falls back to `"noop"` so
     unauthenticated servers keep working
     (merge-after-nits — add a test pinning the
     `Authorization` header on the model-discovery
     probe). All-Hands-AI/OpenHands #14126 routes
     enterprise org/conversation settings reads
     through the SDK's new
     `_load_persisted_agent_settings`/
     `_load_persisted_conversation_settings` loaders
     and pins the SDK + agent-server image to commit
     `054bbd4` across `poetry.lock`, dev/prod compose,
     and pre-commit `additional_dependencies`; AGENTS.md
     gains a note about the easy-to-miss pre-commit
     mypy dep sync (merge-after-nits — request the SDK
     promote `_load_persisted_*` to public names).
     charmbracelet/crush #2663 swaps the shared
     `App.events chan tea.Msg` (competing-consumer)
     for `*pubsub.Broker[tea.Msg]` (fan-out) so each
     SSE client gets its own subscriber channel via
     `Subscribe(ctx)`; the old per-message
     `subscriberSendTimeout` drop logic is deleted
     since the broker owns backpressure, `Events()`
     now takes `ctx` (breaking signature change),
     and `app.events.Shutdown()` lands in the cleanup
     func (merge-as-is). continuedev/continue #12212
     fixes Bedrock long-lived API-key auth by adding
     `authSchemePreference: ["httpBearerAuth"]` to the
     `BedrockRuntimeClient` config inside the
     `if (this.apiKey)` branch — without it the AWS
     SDK falls back to SigV4 and 401s; however the
     change is a 2-line `as any` cast with no test
     (request-changes — replace `as any` with a typed
     extension and add a unit test). cline/cline
     #10401 adds `deepseek-v4-flash`/`deepseek-v4-pro`
     to the `deepSeekModels` catalog in
     `src/shared/api.ts`, widening the closing
     `satisfies Record<string, ModelInfo>` to
     `OpenAiCompatibleModelInfo` so the new
     `supportsReasoningEffort: true` on V4-pro
     typechecks; both entries set `inputPrice: 0`
     which feeds Cline's cost panel as `$0.00` for
     uncached input — likely a billing-shock landmine
     and almost certainly not what DeepSeek publishes
      (request-changes — cite the pricing source, fix
      `inputPrice: 0`, narrow the type widening to
      per-entry instead of map-wide).
- **W17 drip-44 (2026-04-25)**: 8 more — sst/opencode
      ecosystem-table addition (#24283) and a closed
      11k-line "fix" PR that was actually a downstream
      fork's compliance layer in disguise (#24277,
      correctly auto-closed by `needs:title`/`needs:compliance`
      bots); litellm two-clause OR fix for JSON-registered
      OpenAI-compatible providers missing `extra_body`
      wrapping (#26500), and an auth-fix PR scope-conflated
      with a 700-line gpt-5.5 pricing-catalog churn that
      needs splitting before promotion to main (#26499);
      openai/codex one-word README grammar fast-track
      (#19514, 1m open-to-merge); All-Hands V0→V1 cleanup
      stack — architecture-doc deletion (#14120) and
      third-party-runtime tree removal (#14119, with the
      `_DEFAULT_` → `_ALL_` rename that does real semantic
      work); and a browser-use "remove dead code" PR
      whose deleted branch was actually defending against
      `$ref`-inside-`type` schema shapes — closed correctly
      (#4731).
- **W17 drip-49 (2026-04-25)**: 8 more across 4 repos —
      sst/opencode large prompt-input "message
      annotations" feature with a real text-selection UX
      shift (#24311, needs-discussion) and the Effect
      Schema dedup that flips config decode to
      `propertyOrder: "original"` to keep
      first-match-wins permission key order intact
      (#24308); openai/codex final slice of the
      JSON-RPC handler streamline series (review/feedback/
      fuzzy-search/git-diff, #19498) and the
      ghost-snapshot retirement that drops the
      `GhostCommit`/`ghost_snapshot` variant from the
      Responses API surface while keeping the legacy
      `undo` features key parseable as `Stage::Removed`
      (#19481); litellm arize/langfuse_otel `_safe_get`
      fix for raw OpenAI Pydantic `CompletionUsage`
      (no `.get`) closing 252-day-old #13672 (#26506,
      bonus correctness on
      `completion_tokens_details` reasoning-token path)
      and a single-file FuturMix.ai provider registration
      (#26504, with a `max_completion_tokens →
      max_tokens` mapping to verify); browser-use new
      Astraflow/UCloud provider with zero tests, an
      orphan CN endpoint config, default
      `max_retries=10`, and missing reasoning-token
      accounting (#4726, request-changes), and a CDP
      reconnect guard that's right but doesn't cover the
      public `clear_cookies()` sibling (#4722,
      merge-after-nits).
- **W17 drip-50 (2026-04-25)**: 8 more across 4 repos —
      sst/opencode `Permission.reply()` flips silent
      no-op on unknown requestID into a typed
      `NotFoundError` mapped to HTTP 404 at the
      experimental HttpApi boundary (#24322,
      merge-as-is) and an editor lock-file resolver fix
      that filters cross-workspace VSCode IDE locks
      before they cross-inject `selection_changed`
      events into unrelated cwd sessions (#24323,
      merge-after-nits, ranks by resolved-path-string
      length not segment depth — note for future
      readers); openai/codex compaction
      `collect_user_messages` now drops empty + 
      contiguous-duplicate user turns while
      preserving repeats separated by non-user items
      (#19082, merge-as-is, single fixture covers
      both behaviors), and the apply_patch streaming
      parser rewrite from "reparse on every delta" to
      a stateful `StreamingPatchParser` with claimed
      10–15× speedup but couples three changes —
      algorithm, public-API removal of
      `parse_patch_streaming()`, and deletion of the
      largest existing test surface (#19160,
      needs-discussion, want re-asserted
      character-at-a-time fuzz before merge);
      litellm new Mavvrik third-party cost-export
      integration — +5,706 lines / 28 files / 1700+
      lines of test, GCS chunked resumable upload,
      Redis distributed-lock scheduler, 6 admin
      endpoints (#26415, needs-discussion on
      maintenance-debt scope and 45-line proxy_server.py
      delta), and a DeepSeek V4 Pro/V4 Flash metadata
      addition where PR table vs. JSON disagree by
      2.35× / 7.25× on v4-pro input/output prices
      and the new tests assert existence-only with
      zero price-value assertions (#26380,
      merge-after-nits, hard blocker until pricing
      reconciled and price-value tests added);
      browser-use `BrowserProfile.traces_dir` flips
      from silent no-op to fail-loud `ValueError` via
      `@model_validator(mode='after')` and deletes
      the misleading example (#4708, merge-as-is,
      textbook public-field-with-no-impl fix), and a
      test-only PR locking the non-obvious
      "close-before-start rotates the event_bus"
       contract that protects future close()-fast-path
       optimizations from silently breaking session
       reuse (#4707, merge-as-is).
- **W17 drip-51 (2026-04-26)**: 8 more across 4 repos —
       sst/opencode `AppFileSystem.resolve()` widens the
       `realpathSync` catch from `ENOENT`-only to a
       catch-all so `ELOOP`/`EACCES`/`ENOTDIR` no longer
       crash callers that just want a normalized path
       (#24337, merge-as-is, brings the helper into
       parity with the sibling `util/filesystem.ts`
       `normalizePath`); a 100-file barrel-removal sweep
       that flips `import { Log } from "@/util"` to
       `@/util/log` everywhere and rewrites `AGENTS.md`
       wholesale, but silently contradicts the prior
       documented self-export pattern and bundles three
       policy changes into one mechanical refactor
       (#24333, needs-discussion); a three-fix bundle
       adding 60s eviction timers to the per-file edit
       lock map and a 30s `Promise.race` timeout to the
       rwlock primitive, with a real reader-count leak
       on the timeout-after-acquisition race that should
       be fixed before merge (#24332, merge-after-nits);
       and a sweep promoting silent `catch {}` blocks to
       `log.debug`/`log.warn` across 11 modules with
       correct severity discipline plus a bonus
       `catch(e: any)` → `unknown` narrowing fix in
       `session/llm.ts` (#24331, merge-as-is, with one
       stray `Session` self-export that conflicts with
       the parallel barrel-removal PR);
       openai/codex `--cloud` mode for `exec-server`
       that registers with a cloud environments service
       and serves over the rendezvous websocket with
       sha256-derived idempotency keys, ChatGPT-only
       auth enforced at AuthManager construction
       (`enable_codex_api_key_env=false`), and 1s→30s
       reconnect backoff — but the "waiting for account
       id" error message implies wait-and-retry where
       the code actually bubbles, and there's no
       graceful drain on shutdown (#19575,
       merge-after-nits);
       litellm Fireworks AI `<think>...</think>`
       extraction into `reasoning_content` mirroring
       DeepSeek/Ollama, both non-streaming and
       streaming surfaces — but the streaming state
       machine never resets `started_reasoning_content`
       on `</think>`, so a second `<think>` block in
       the same stream (R1-style plan-then-act models)
       silently leaks into `content` (#26505,
       merge-after-nits);
       browser-use one-line `'type'` removal from
       `optimize_schema()`'s essential-fields elif —
       genuinely unreachable because an earlier
       `if key == 'type'` arm already handles it
       (#4719, merge-as-is); and a six-step Docker
       detection ladder that *makes the false-positive
       rate strictly worse* — `'overlay' in mountinfo`
       fires on `unshare`-based dev sandboxes, and
       `KUBERNETES_SERVICE_HOST` conflates K8s with
       Docker for a function whose return value gates
       Chrome launch flags, silently degrading
       interactive perf on dev laptops (#4718,
       request-changes).
- **W17 drip-52 (2026-04-26)**: 8 more across 4 repos —
       sst/opencode adds a 30s `Effect.timeoutOrElse` to
       the ripgrep bootstrap download with a defensive
       `normalizeDownloadError` so the timeout error
       isn't silently relabeled by the downstream
       `mapError` (#24345, merge-as-is); a
       `stripPlanModeReminders` flag on
       `toModelMessagesEffect` that filters synthetic
       `<system-reminder>Plan mode is active</system-reminder>`
       parts from history when the agent is no longer
       the plan agent — substring-matched on prose,
       which works today but is brittle vs. a structured
       discriminator (#24343, merge-after-nits); and a
       two-bug watcher fix that flips Windows
       `\`-paths to `/` and walks up to the nearest
       loaded ancestor on add/unlink instead of only
       checking the immediate parent (#24341,
       merge-as-is); litellm gates the implicit
       `metadata.user_id`→`user` mapping in the
       Anthropic→Responses adapter on `litellm.drop_params`
       so silent-drop callers don't get a rejected
       `user` field (#26243, merge-after-nits); and
       splits the non-streaming json+tools path into a
       new `_resolve_json_mode_non_streaming` 3-tuple
       that handles the previously-broken case of
       `RESPONSE_FORMAT_TOOL_NAME` mixed with real user
       tool calls in one response (#26222,
       merge-after-nits, with a separator nit on the
       merged content); openai/codex switches the
       Bash + PowerShell installers from the GitHub
       Releases API JSON (rate-limited) to a static
       checksums manifest + predictable download URL,
       and flattens the standalone archive layout —
       but the Bash awk parser doesn't strip the
       leading `*` that `sha256sum -b` emits, while
       PowerShell does (#18901, merge-as-is with a
       one-line awk fix recommended); and adds an
       explicit `enabled: bool` to `NetworkContext`
       so removing a network policy mid-session
       renders `<network enabled="false" />` instead
       of going silent — disambiguating "removed" from
       "unchanged" via a private `disabled()`
       constructor and a match-arm-with-guard
       (#18883, merge-as-is); browser-use uses CDP
       `screencastFrame.metadata.timestamp` to
       duplicate-fill frames so video playback runs in
       real time, but the fill loop has no upper bound
       — a 5-minute backgrounded tab inserts ~9000
       frames in a tight Python loop and stalls the
       async event loop (#4717, request-changes,
       cap+test required).
- **W17 drip-59 (2026-04-26)**: 8 more across 3 repos.
       openai/codex contributes two CI-side merges:
       #19578 bumps the Bazel job timeout from 30→45
       minutes with an honest comment naming
       BuildBuddy RBE as the eventual fix
       (merge-as-is), and #19593 disables plugin
       warmups in the
       `thread_start_with_non_local_thread_store_does_not_create_local_persistence`
       fixture so a stray top-level `.tmp` from
       plugin startup stops tripping the
       persistence-artifact assertion on Windows
       (merge-as-is). BerriAI/litellm #26524 closes a
       privilege gap where any valid API key could
       `DELETE /v1/vector_stores/{id}` or
       `DELETE /azure_ai/indexes/{name}` — the
       OpenAI-shaped route had no role check and the
       Azure pass-through fell through to the upstream
       on a missing-from-registry index; the fix wires
       `_is_proxy_admin` gates at all three sites with
       403 responses, but the registry-miss tail path
       lacks a regression test and the create/modify
       paths weren't audited in the same PR
       (merge-after-nits). anomalyco/opencode lands
       five: #24395 introduces `agent_memory` (SQLite
       table + Effect-based `AgentMemory.save/search/
       consolidate` service) bundled with a brand-new
       `packages/memory-tools` Supabase cloud-sync
       plugin — the privacy/egress story isn't
       discussed and the schema migration is
       interleaved with the optional plugin, so this
       wants to split (needs-discussion); #24382 adds
       a vision-model fallback in `session/llm.ts`
       that runs `generateText` through a
       cross-provider vision model and replaces image
       parts with a `[Image description by ${id}]:`
       text block when the active model lacks vision
       capability — fail-soft on errors, but silently
       so, and multi-image positional grounding is
       lost (merge-after-nits); #24378 syncs every
       locale's env-var docs table with source —
       structural sort + add-missing-rows, mostly a
       generator output, native-speaker pass on the
       new translated descriptions wanted
       (merge-after-nits); and ecosystem-table
       additions #24390 (`opencode-claude-code-plugin`,
       merge-after-nits) and #24388
       (`opencode-local-ollama`, merge-as-is, closes
       #24389).


See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.

## Aider-AI/aider

| PR | Title | File |
|---|---|---|
| [#5050](https://github.com/Aider-AI/aider/pull/5050) | fix: fall back to ASCII in tool_output | [Aider-AI-aider-pr-5050.md](2026-W17/drip-56/Aider-AI-aider-pr-5050.md) |
| [#5024](https://github.com/Aider-AI/aider/pull/5024) | fix: check len(errors) before string conversion in udiff_coder.py | [Aider-AI-aider-pr-5024.md](2026-W17/drip-34/Aider-AI-aider-pr-5024.md) |
| [#5066](https://github.com/Aider-AI/aider/pull/5066) | Polyglot benchmark: Claude Opus 4.7 new #1 at 93.3% | [Aider-AI-aider-pr-5066.md](2026-W17/drip-35/Aider-AI-aider-pr-5066.md) |
| [#5052](https://github.com/Aider-AI/aider/pull/5052) | Add bash/shell repomap support | [Aider-AI-aider-pr-5052.md](2026-W17/drip-35/Aider-AI-aider-pr-5052.md) |
| [#5033](https://github.com/Aider-AI/aider/pull/5033) | feat: skip auto-commit when LLM response is truncated | [Aider-AI-aider-pr-5033.md](2026-W17/drip-33/Aider-AI-aider-pr-5033.md) |
| [#5065](https://github.com/Aider-AI/aider/pull/5065) | Fix prompt injection vulnerability in Architect mode (Issue #5058) | [Aider-AI-aider-pr-5065.md](2026-W17/drip-32/Aider-AI-aider-pr-5065.md) |
| [#5031](https://github.com/Aider-AI/aider/pull/5031) | fix(io): tool_output falls back to ASCII on UnicodeEncodeError (#5029) | [Aider-AI-aider-pr-5031.md](2026-W17/drip-30/Aider-AI-aider-pr-5031.md) |
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
| [#14116](https://github.com/All-Hands-AI/OpenHands/pull/14116) | fix: normalize legacy MCP config in migration 108 | [All-Hands-AI-OpenHands-pr-14116.md](2026-W17/drip-48/All-Hands-AI-OpenHands-pr-14116.md) |
| [#14086](https://github.com/All-Hands-AI/OpenHands/pull/14086) | fix: don't persist Gemini model-specific endpoint as base_url | [All-Hands-AI-OpenHands-pr-14086.md](2026-W17/drip-47/All-Hands-AI-OpenHands-pr-14086.md) |
| [#14125](https://github.com/All-Hands-AI/OpenHands/pull/14125) | fix(integrations): guard Bitbucket Data Center against null nested fields | [All-Hands-AI-OpenHands-pr-14125.md](2026-W17/drip-46/All-Hands-AI-OpenHands-pr-14125.md) |
| [#14132](https://github.com/All-Hands-AI/OpenHands/pull/14132) | fix: support pagination for branch search | [All-Hands-AI-OpenHands-pr-14132.md](2026-W17/drip-45/All-Hands-AI-OpenHands-pr-14132.md) |
| [#14120](https://github.com/All-Hands-AI/OpenHands/pull/14120) | Removed Architecture diagrams | [All-Hands-AI-OpenHands-pr-14120.md](2026-W17/drip-44/All-Hands-AI-OpenHands-pr-14120.md) |
| [#14119](https://github.com/All-Hands-AI/OpenHands/pull/14119) | Removed V0 third party runtimes | [All-Hands-AI-OpenHands-pr-14119.md](2026-W17/drip-44/All-Hands-AI-OpenHands-pr-14119.md) |
| [#14118](https://github.com/All-Hands-AI/OpenHands/pull/14118) | feat(enterprise): Add GitLab event forwarding to automation service | [All-Hands-AI-OpenHands-pr-14118.md](2026-W17/drip-43/All-Hands-AI-OpenHands-pr-14118.md) |
| [#14122](https://github.com/All-Hands-AI/OpenHands/pull/14122) | feat: enable sub-agent delegation via TaskToolSet in app server | [All-Hands-AI-OpenHands-pr-14122.md](2026-W17/drip-42/All-Hands-AI-OpenHands-pr-14122.md) |
| [#14128](https://github.com/All-Hands-AI/OpenHands/pull/14128) | Support pagination for branch search | [All-Hands-AI-OpenHands-pr-14128.md](2026-W17/drip-41/All-Hands-AI-OpenHands-pr-14128.md) |
| [#14099](https://github.com/All-Hands-AI/OpenHands/pull/14099) | fix(git): support pagination for branch search queries | [All-Hands-AI-OpenHands-pr-14099.md](2026-W17/drip-40/All-Hands-AI-OpenHands-pr-14099.md) |
| [#14101](https://github.com/All-Hands-AI/OpenHands/pull/14101) | fix(app): deliver pending messages queued during startup | [All-Hands-AI-OpenHands-pr-14101.md](2026-W17/drip-39/All-Hands-AI-OpenHands-pr-14101.md) |
| [#14126](https://github.com/All-Hands-AI/OpenHands/pull/14126) | feat(settings): use from_persisted for stored settings | [All-Hands-AI-OpenHands-pr-14126.md](2026-W17/drip-38/All-Hands-AI-OpenHands-pr-14126.md) |
| [#14104](https://github.com/All-Hands-AI/OpenHands/pull/14104) | fix(frontend): restore think title fallback | [All-Hands-AI-OpenHands-pr-14104.md](2026-W17/drip-34/All-Hands-AI-OpenHands-pr-14104.md) |
| [#14122](https://github.com/All-Hands-AI/OpenHands/pull/14122) | feat: enable sub-agent delegation via TaskToolSet in app server | [All-Hands-AI-OpenHands-pr-14122.md](2026-W17/drip-33/All-Hands-AI-OpenHands-pr-14122.md) |
| [#14105](https://github.com/All-Hands-AI/OpenHands/pull/14105) | fix(frontend): allow send when only attachments are present | [All-Hands-AI-OpenHands-pr-14105.md](2026-W17/drip-33/All-Hands-AI-OpenHands-pr-14105.md) |
| [#14127](https://github.com/All-Hands-AI/OpenHands/pull/14127) | Reduce GitHub resolver comment noise by editing acknowledgement comment | [All-Hands-AI-OpenHands-pr-14127.md](2026-W17/drip-32/All-Hands-AI-OpenHands-pr-14127.md) |
| [#14125](https://github.com/All-Hands-AI/OpenHands/pull/14125) | fix(integrations): Bitbucket Data Center webhook null guards | [All-Hands-AI-OpenHands-pr-14125.md](2026-W17/drip-32/All-Hands-AI-OpenHands-pr-14125.md) |
| [#14114](https://github.com/All-Hands-AI/OpenHands/pull/14114) | fix(runtime): shrink BashSession tmux pane from 1000x1000 to 256x50 | [All-Hands-AI-OpenHands-pr-14114.md](2026-W17/drip-30/All-Hands-AI-OpenHands-pr-14114.md) |
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
| [#24392](https://github.com/anomalyco/opencode/pull/24392) | chore: add changelog sync workflow and changelog | [anomalyco-opencode-pr-24392.md](2026-W17/drip-58/anomalyco-opencode-pr-24392.md) |
| [#24387](https://github.com/anomalyco/opencode/pull/24387) | feat(httpapi): bridge config update endpoint | [anomalyco-opencode-pr-24387.md](2026-W17/drip-58/anomalyco-opencode-pr-24387.md) |
| [#24386](https://github.com/anomalyco/opencode/pull/24386) | fix(provider): preserve Azure API version | [anomalyco-opencode-pr-24386.md](2026-W17/drip-58/anomalyco-opencode-pr-24386.md) |
| [#24384](https://github.com/anomalyco/opencode/pull/24384) | fix(provider): respect configured output limit | [anomalyco-opencode-pr-24384.md](2026-W17/drip-58/anomalyco-opencode-pr-24384.md) |
| [#24383](https://github.com/anomalyco/opencode/pull/24383) | fix: move session roots filter from client-side to SQL layer | [anomalyco-opencode-pr-24383.md](2026-W17/drip-58/anomalyco-opencode-pr-24383.md) |
| [#24379](https://github.com/anomalyco/opencode/pull/24379) | fix(session): use transcript position instead of lexical ID compare in prompt loop | [anomalyco-opencode-pr-24379.md](2026-W17/drip-57/anomalyco-opencode-pr-24379.md) |
| [#24369](https://github.com/anomalyco/opencode/pull/24369) | feat(processor): add model fallback chain when retries are exhausted | [anomalyco-opencode-pr-24369.md](2026-W17/drip-57/anomalyco-opencode-pr-24369.md) |
| [#24367](https://github.com/anomalyco/opencode/pull/24367) | fix(zen): stop double-counting reasoning_tokens in oa-compat usage | [anomalyco-opencode-pr-24367.md](2026-W17/drip-57/anomalyco-opencode-pr-24367.md) |
| [#24319](https://github.com/sst/opencode/pull/24319) | fix(file): show linked directories in file list | [sst-opencode-pr-24319.md](2026-W17/drip-55/sst-opencode-pr-24319.md) |
| [#24330](https://github.com/sst/opencode/pull/24330) | fix: correct broken CI workflows and infra migration | [sst-opencode-pr-24330.md](2026-W17/drip-55/sst-opencode-pr-24330.md) |
| [#24351](https://github.com/sst/opencode/pull/24351) | fix(app): keep question dock buttons visible on mobile | [sst-opencode-pr-24351.md](2026-W17/drip-55/sst-opencode-pr-24351.md) |
| [#24363](https://github.com/sst/opencode/pull/24363) | fix(agent): accept common color names | [sst-opencode-pr-24363.md](2026-W17/drip-53/sst-opencode-pr-24363.md) |
| [#24350](https://github.com/sst/opencode/pull/24350) | fix: deepseek when using messages api | [sst-opencode-pr-24350.md](2026-W17/drip-53/sst-opencode-pr-24350.md) |
| [#24340](https://github.com/sst/opencode/pull/24340) | fix(acp): expose variant config option | [sst-opencode-pr-24340.md](2026-W17/drip-53/sst-opencode-pr-24340.md) |
| [#24336](https://github.com/sst/opencode/pull/24336) | fix(session): clamp token usage counts | [sst-opencode-pr-24336.md](2026-W17/drip-53/sst-opencode-pr-24336.md) |
| [#24345](https://github.com/sst/opencode/pull/24345) | fix(ripgrep): time out binary download | [sst-opencode-pr-24345.md](2026-W17/drip-52/sst-opencode-pr-24345.md) |
| [#24343](https://github.com/sst/opencode/pull/24343) | fix(session): drop stale plan reminders | [sst-opencode-pr-24343.md](2026-W17/drip-52/sst-opencode-pr-24343.md) |
| [#24341](https://github.com/sst/opencode/pull/24341) | fix(app): normalize watcher paths | [sst-opencode-pr-24341.md](2026-W17/drip-52/sst-opencode-pr-24341.md) |
| [#24337](https://github.com/sst/opencode/pull/24337) | fix(filesystem): tolerate unresolved paths | [sst-opencode-pr-24337.md](2026-W17/drip-51/sst-opencode-pr-24337.md) |
| [#24333](https://github.com/sst/opencode/pull/24333) | refactor: remove barrel index.ts and flatten export namespace | [sst-opencode-pr-24333.md](2026-W17/drip-51/sst-opencode-pr-24333.md) |
| [#24332](https://github.com/sst/opencode/pull/24332) | fix: add lock timeout/eviction and fix concurrency issues | [sst-opencode-pr-24332.md](2026-W17/drip-51/sst-opencode-pr-24332.md) |
| [#24331](https://github.com/sst/opencode/pull/24331) | fix: add logging to silent catch blocks across core modules | [sst-opencode-pr-24331.md](2026-W17/drip-51/sst-opencode-pr-24331.md) |
| [#24323](https://github.com/sst/opencode/pull/24323) | fix(editor): reject lock files with no workspace match for cwd | [sst-opencode-pr-24323.md](2026-W17/drip-50/sst-opencode-pr-24323.md) |
| [#24322](https://github.com/sst/opencode/pull/24322) | fix(permission): reject stale permission replies | [sst-opencode-pr-24322.md](2026-W17/drip-50/sst-opencode-pr-24322.md) |
| [#24311](https://github.com/sst/opencode/pull/24311) | feat(app): support message annotations | [sst-opencode-pr-24311.md](2026-W17/drip-49/sst-opencode-pr-24311.md) |
| [#24308](https://github.com/sst/opencode/pull/24308) | fix(config): preserve permission order with Effect decode | [sst-opencode-pr-24308.md](2026-W17/drip-49/sst-opencode-pr-24308.md) |
| [#24297](https://github.com/sst/opencode/pull/24297) | fix(opencode): resolve heap unlimited + orphan processes on Linux | [sst-opencode-pr-24297.md](2026-W17/drip-47/sst-opencode-pr-24297.md) |
| [#24296](https://github.com/sst/opencode/pull/24296) | docs: sync env vars with source code | [sst-opencode-pr-24296.md](2026-W17/drip-47/sst-opencode-pr-24296.md) |
| [#24293](https://github.com/sst/opencode/pull/24293) | fix(task): propagate parent session permissions to sub-agents | [sst-opencode-pr-24293.md](2026-W17/drip-46/sst-opencode-pr-24293.md) |
| [#24278](https://github.com/sst/opencode/pull/24278) | fix(provider): prevent duplicate OAuth requests to Google when using Vertex AI provider | [sst-opencode-pr-24278.md](2026-W17/drip-46/sst-opencode-pr-24278.md) |
| [#24290](https://github.com/sst/opencode/pull/24290) | fix(session): skip tool calls during summary instead of throwing | [sst-opencode-pr-24290.md](2026-W17/drip-45/sst-opencode-pr-24290.md) |
| [#24289](https://github.com/sst/opencode/pull/24289) | fix: Repair truncated JSON tool inputs in LLM session | [sst-opencode-pr-24289.md](2026-W17/drip-45/sst-opencode-pr-24289.md) |
| [#24283](https://github.com/sst/opencode/pull/24283) | docs: add opencode-provider-alias to ecosystem | [sst-opencode-pr-24283.md](2026-W17/drip-44/sst-opencode-pr-24283.md) |
| [#24277](https://github.com/sst/opencode/pull/24277) | [codex] Fix team run preflight aborts (closed; scope mismatch) | [sst-opencode-pr-24277.md](2026-W17/drip-44/sst-opencode-pr-24277.md) |
| [#24285](https://github.com/anomalyco/opencode/pull/24285) | fix(tui): add copy action for native question prompts | [anomalyco-opencode-pr-24285.md](2026-W17/drip-43/anomalyco-opencode-pr-24285.md) |
| [#24279](https://github.com/anomalyco/opencode/pull/24279) | fix(app): align usage chart with local timezone | [anomalyco-opencode-pr-24279.md](2026-W17/drip-43/anomalyco-opencode-pr-24279.md) |
| [#24272](https://github.com/anomalyco/opencode/pull/24272) | docs: add Mongolian README documentation | [anomalyco-opencode-pr-24272.md](2026-W17/drip-42/anomalyco-opencode-pr-24272.md) |
| [#24271](https://github.com/anomalyco/opencode/pull/24271) | Set active server before navigation and use replace navigation | [anomalyco-opencode-pr-24271.md](2026-W17/drip-42/anomalyco-opencode-pr-24271.md) |
| [#24273](https://github.com/anomalyco/opencode/pull/24273) | docs: correct compaction prune default | [anomalyco-opencode-pr-24273.md](2026-W17/drip-41/anomalyco-opencode-pr-24273.md) |
| [#24259](https://github.com/sst/opencode/pull/24259) | docs: add opencode-simple-notify to ecosystem | [sst-opencode-pr-24259.md](2026-W17/drip-40/sst-opencode-pr-24259.md) |
| [#24262](https://github.com/sst/opencode/pull/24262) | fix(provider): inject chat_template_kwargs for Nvidia NIM deepseek-v4 models | [sst-opencode-pr-24262.md](2026-W17/drip-39/sst-opencode-pr-24262.md) |
| [#24258](https://github.com/sst/opencode/pull/24258) | feat(httpapi): bridge instance read endpoints | [sst-opencode-pr-24258.md](2026-W17/drip-38/sst-opencode-pr-24258.md) |
| [#24251](https://github.com/anomalyco/opencode/pull/24251) | feat: add project run configs to the web app | [anomalyco-opencode-pr-24251.md](2026-W17/drip-34/anomalyco-opencode-pr-24251.md) |
| [#24250](https://github.com/anomalyco/opencode/pull/24250) | fix(provider): complete DeepSeek reasoning_content round-trip for multi-turn conversations | [anomalyco-opencode-pr-24250.md](2026-W17/drip-34/anomalyco-opencode-pr-24250.md) |
| [#24246](https://github.com/anomalyco/opencode/pull/24246) | fix: preserve nix/direnv PATH in login shell for ! commands | [anomalyco-opencode-pr-24246.md](2026-W17/drip-33/anomalyco-opencode-pr-24246.md) |
| [#24244](https://github.com/anomalyco/opencode/pull/24244) | fix: remove top-level wildcard-first sort in permission config | [anomalyco-opencode-pr-24244.md](2026-W17/drip-33/anomalyco-opencode-pr-24244.md) |
| [#24232](https://github.com/anomalyco/opencode/pull/24232) | fix(session): honor noCacheTokens in usage accounting | [anomalyco-opencode-pr-24232.md](2026-W17/drip-33/anomalyco-opencode-pr-24232.md) |
| [#24241](https://github.com/anomalyco/opencode/pull/24241) | fix(tui): clean zero-width agent display labels | [anomalyco-opencode-pr-24241.md](2026-W17/drip-30/anomalyco-opencode-pr-24241.md) |
| [#24234](https://github.com/anomalyco/opencode/pull/24234) | fix(agent): detect non-object frontmatter from gray-matter | [anomalyco-opencode-pr-24234.md](2026-W17/drip-30/anomalyco-opencode-pr-24234.md) |
| [#24233](https://github.com/anomalyco/opencode/pull/24233) | fix(provider): honor per-model reasoning token pricing | [anomalyco-opencode-pr-24233.md](2026-W17/drip-30/anomalyco-opencode-pr-24233.md) |
| [#24229](https://github.com/anomalyco/opencode/pull/24229) | fix(session): lazy session error schema via Schema.suspend | [anomalyco-opencode-pr-24229.md](2026-W17/drip-30/anomalyco-opencode-pr-24229.md) |
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
| [#24150](https://github.com/sst/opencode/pull/24150) | fix(transform): inject reasoning_content for ALL assistant msgs to fix DeepSeek thinking mode | [PR-24150-deepseek-reasoning-content-all-msgs.md](anomalyco-opencode/PR-24150-deepseek-reasoning-content-all-msgs.md) |
| [#24149](https://github.com/sst/opencode/pull/24149) | feat(scout): add scout agent for repo research | [PR-24149-scout-agent-repo-research.md](anomalyco-opencode/PR-24149-scout-agent-repo-research.md) |
| [#24170](https://github.com/sst/opencode/pull/24170) | fix: preserve assistant tool_calls in openai-compatible replay | [anomalyco-opencode-pr-24170.md](2026-W17/anomalyco-opencode-pr-24170.md) |
| [#24168](https://github.com/sst/opencode/pull/24168) | Refactor HttpApi auth middleware wiring | [anomalyco-opencode-pr-24168.md](2026-W17/anomalyco-opencode-pr-24168.md) |
| [#24161](https://github.com/sst/opencode/pull/24161) | feat: add /uptime slash command | [anomalyco-opencode-pr-24161.md](2026-W17/anomalyco-opencode-pr-24161.md) |
| [#24154](https://github.com/sst/opencode/pull/24154) | feat: add unarchive/restore for archived sessions | [anomalyco-opencode-pr-24154.md](2026-W17/anomalyco-opencode-pr-24154.md) |
| [#24169](https://github.com/sst/opencode/pull/24169) | refactor(schema): decode effect schemas directly | [sst-opencode-pr-24169.md](2026-W17/drip-20/sst-opencode-pr-24169.md) |
| [#24179](https://github.com/sst/opencode/pull/24179) | feat: expose a session-scoped permission bridge for external providers | [sst-opencode-pr-24179.md](2026-W17/sst-opencode-pr-24179.md) |
| [#24162](https://github.com/sst/opencode/pull/24162) | fix(desktop): add retry logic with exponential backoff to health check system | [sst-opencode-pr-24162.md](2026-W17/sst-opencode-pr-24162.md) |
| [#24144](https://github.com/sst/opencode/pull/24144) | docs: add MDNS workaround for WSL path issues in windows-wsl.mdx | [sst-opencode-pr-24144.md](2026-W17/drip-17/sst-opencode-pr-24144.md) |
| [#24008](https://github.com/sst/opencode/pull/24008) | fix: preserve newlines in ACP command argument parsing | [sst-opencode-pr-24008.md](2026-W17/drip-17/sst-opencode-pr-24008.md) |
| [#24194](https://github.com/sst/opencode/pull/24194) | restrict amazon-bedrock provider to curated model allowlist | [sst-opencode-pr-24194.md](2026-W17/drip-17/sst-opencode-pr-24194.md) |
| [#24193](https://github.com/sst/opencode/pull/24193) | docs: clarify bedrock inference profile setup | [sst-opencode-pr-24193.md](2026-W17/drip-17/sst-opencode-pr-24193.md) |
| [#24136](https://github.com/sst/opencode/pull/24136) | feat(desktop): add support for Open in Windows Terminal in the header menu | [sst-opencode-pr-24136.md](2026-W17/drip-17/sst-opencode-pr-24136.md) |
| [#24128](https://github.com/sst/opencode/pull/24128) | feat: optimize media attachments on paste in TUI | [sst-opencode-pr-24128.md](2026-W17/drip-17/sst-opencode-pr-24128.md) |
| [#24174](https://github.com/sst/opencode/pull/24174) | feat(core): add background subagent support | [sst-opencode-pr-24174.md](2026-W17/drip-19/sst-opencode-pr-24174.md) |
| [#24107](https://github.com/sst/opencode/pull/24107) | fix: prevent question custom input from hiding submit button | [sst-opencode-pr-24107.md](2026-W17/drip-19/sst-opencode-pr-24107.md) |
| [#24047](https://github.com/sst/opencode/pull/24047) | docs: add agent architecture audit guide | [sst-opencode-pr-24047.md](2026-W17/drip-21/sst-opencode-pr-24047.md) |
| [#24200](https://github.com/sst/opencode/pull/24200) | fix: preserve empty reasoning_content for DeepSeek V4 in non-streaming and streaming paths | [sst-opencode-pr-24200.md](2026-W17/drip-22/sst-opencode-pr-24200.md) |
| [#24022](https://github.com/sst/opencode/pull/24022) | fix(app): prevent question dock overflow | [sst-opencode-pr-24022.md](2026-W17/drip-22/sst-opencode-pr-24022.md) |
| [#24202](https://github.com/sst/opencode/pull/24202) | perf(tool): condense tool descriptions ~66% to cut system-prompt tokens | [sst-opencode-pr-24202.md](2026-W17/drip-23/sst-opencode-pr-24202.md) |
| [#24196](https://github.com/sst/opencode/pull/24196) | feat(console): improve Polish translations | [sst-opencode-pr-24196.md](2026-W17/drip-23/sst-opencode-pr-24196.md) |
| [#24205](https://github.com/anomalyco/opencode/pull/24205) | fix(cli): authenticate run in-process server requests | [anomalyco-opencode-pr-24205.md](2026-W17/drip-24/anomalyco-opencode-pr-24205.md) |
| [#24215](https://github.com/anomalyco/opencode/pull/24215) | fix(session): preserve shell cwd after startup | [anomalyco-opencode-pr-24215.md](2026-W17/drip-25/anomalyco-opencode-pr-24215.md) |
| [#24210](https://github.com/anomalyco/opencode/pull/24210) | feat(opencode): add /context command | [anomalyco-opencode-pr-24210.md](2026-W17/drip-25/anomalyco-opencode-pr-24210.md) |
| [#24218](https://github.com/anomalyco/opencode/pull/24218) | fix(provider): auto-enable interleaved for reasoning models | [anomalyco-opencode-pr-24218.md](2026-W17/drip-26/anomalyco-opencode-pr-24218.md) |
| [#24222](https://github.com/anomalyco/opencode/pull/24222) | fix permission config order | [anomalyco-opencode-pr-24222.md](2026-W17/drip-26/anomalyco-opencode-pr-24222.md) |
| [#24220](https://github.com/anomalyco/opencode/pull/24220) | Fix session event typechecks and shell cwd | [anomalyco-opencode-pr-24220.md](2026-W17/drip-27/anomalyco-opencode-pr-24220.md) |
| [#24219](https://github.com/anomalyco/opencode/pull/24219) | docs(effect): add generated http route inventory | [anomalyco-opencode-pr-24219.md](2026-W17/drip-27/anomalyco-opencode-pr-24219.md) |
| [#24395](https://github.com/anomalyco/opencode/pull/24395) | feat(memory): add agent_memory table and memory-tools plugin | [anomalyco-opencode-pr-24395.md](2026-W17/drip-59/anomalyco-opencode-pr-24395.md) |
| [#24390](https://github.com/anomalyco/opencode/pull/24390) | docs: add opencode-claude-code-plugin to ecosystem plugins | [anomalyco-opencode-pr-24390.md](2026-W17/drip-59/anomalyco-opencode-pr-24390.md) |
| [#24388](https://github.com/anomalyco/opencode/pull/24388) | docs: add opencode-local-ollama to ecosystem plugins | [anomalyco-opencode-pr-24388.md](2026-W17/drip-59/anomalyco-opencode-pr-24388.md) |
| [#24382](https://github.com/anomalyco/opencode/pull/24382) | feat(llm): auto-describe images via vision fallback when active model lacks vision support | [anomalyco-opencode-pr-24382.md](2026-W17/drip-59/anomalyco-opencode-pr-24382.md) |
| [#24378](https://github.com/anomalyco/opencode/pull/24378) | docs: sync env vars with source code | [anomalyco-opencode-pr-24378.md](2026-W17/drip-59/anomalyco-opencode-pr-24378.md) |

## BerriAI/litellm

| PR | Title | File |
|---|---|---|
| [#26524](https://github.com/BerriAI/litellm/pull/26524) | fix(vector_stores): restrict vector store and index deletion to proxy admins | [BerriAI-litellm-pr-26524.md](2026-W17/drip-59/BerriAI-litellm-pr-26524.md) |
| [#26525](https://github.com/BerriAI/litellm/pull/26525) | [Fix] broaden RAG ingestion credential cleanup to AWS endpoint/identity fields | [BerriAI-litellm-pr-26525.md](2026-W17/drip-58/BerriAI-litellm-pr-26525.md) |
| [#26520](https://github.com/BerriAI/litellm/pull/26520) | [Feat] Add "My User" tab to team info page | [BerriAI-litellm-pr-26520.md](2026-W17/drip-58/BerriAI-litellm-pr-26520.md) |
| [#26518](https://github.com/BerriAI/litellm/pull/26518) | chore(auth): tighten clientside api_base handling | [BerriAI-litellm-pr-26518.md](2026-W17/drip-57/BerriAI-litellm-pr-26518.md) |
| [#26498](https://github.com/BerriAI/litellm/pull/26498) | fix(auth): apply temp_budget_increase on cache-hit path | [BerriAI-litellm-pr-26498.md](2026-W17/drip-57/BerriAI-litellm-pr-26498.md) |
| [#26517](https://github.com/BerriAI/litellm/pull/26517) | feat(mcp): non-admin MCP self-service (disconnect, approval filter, BYOK admin guard) | [BerriAI-litellm-pr-26517.md](2026-W17/drip-56/BerriAI-litellm-pr-26517.md) |
| [#26515](https://github.com/BerriAI/litellm/pull/26515) | fix(ui): render dict-shaped fallback entries in router settings | [BerriAI-litellm-pr-26515.md](2026-W17/drip-56/BerriAI-litellm-pr-26515.md) |
| [#26499](https://github.com/BerriAI/litellm/pull/26499) | fix(auth): join team-member budget so rpm/tpm limits are enforced | [BerriAI-litellm-pr-26499.md](2026-W17/drip-56/BerriAI-litellm-pr-26499.md) |
| [#26513](https://github.com/BerriAI/litellm/pull/26513) | [Fix] Harden /model/info redaction for plural credential field names | [BerriAI-litellm-pr-26513.md](2026-W17/drip-55/BerriAI-litellm-pr-26513.md) |
| [#26512](https://github.com/BerriAI/litellm/pull/26512) | [Fix] bind RAG ingestion config to stored credential values | [BerriAI-litellm-pr-26512.md](2026-W17/drip-55/BerriAI-litellm-pr-26512.md) |
| [#26373](https://github.com/BerriAI/litellm/pull/26373) | fix(azure): propagate llm_provider- response headers on Responses API cancel | [BerriAI-litellm-pr-26373.md](2026-W17/drip-55/BerriAI-litellm-pr-26373.md) |
| [#26508](https://github.com/BerriAI/litellm/pull/26508) | Litellm oss staging 04 25 2026 (Cloudflare response_text + Z.AI provider) | [BerriAI-litellm-pr-26508.md](2026-W17/drip-53/BerriAI-litellm-pr-26508.md) |
| [#26374](https://github.com/BerriAI/litellm/pull/26374) | fix(mcp): do not prefix tool names when listing via scoped /mcp/{server_name} | [BerriAI-litellm-pr-26374.md](2026-W17/drip-53/BerriAI-litellm-pr-26374.md) |
| [#26243](https://github.com/BerriAI/litellm/pull/26243) | fix: respect drop_params when mapping metadata.user_id to user in Responses adapter | [BerriAI-litellm-pr-26243.md](2026-W17/drip-52/BerriAI-litellm-pr-26243.md) |
| [#26222](https://github.com/BerriAI/litellm/pull/26222) | fix(anthropic): json response_format + user tools non-streaming | [BerriAI-litellm-pr-26222.md](2026-W17/drip-52/BerriAI-litellm-pr-26222.md) |
| [#26505](https://github.com/BerriAI/litellm/pull/26505) | feat(fireworks): extract think tags into reasoning_content | [BerriAI-litellm-pr-26505.md](2026-W17/drip-51/BerriAI-litellm-pr-26505.md) |
| [#26415](https://github.com/BerriAI/litellm/pull/26415) | feat(mavvrik): add Mavvrik integration | [BerriAI-litellm-pr-26415.md](2026-W17/drip-50/BerriAI-litellm-pr-26415.md) |
| [#26380](https://github.com/BerriAI/litellm/pull/26380) | feat(deepseek): add DeepSeek V4 Pro and V4 Flash model metadata | [BerriAI-litellm-pr-26380.md](2026-W17/drip-50/BerriAI-litellm-pr-26380.md) |
| [#26506](https://github.com/BerriAI/litellm/pull/26506) | fix(arize): _set_usage_outputs handles raw OpenAI Pydantic CompletionUsage | [BerriAI-litellm-pr-26506.md](2026-W17/drip-49/BerriAI-litellm-pr-26506.md) |
| [#26504](https://github.com/BerriAI/litellm/pull/26504) | feat: add FuturMix as named OpenAI-compatible provider | [BerriAI-litellm-pr-26504.md](2026-W17/drip-49/BerriAI-litellm-pr-26504.md) |
| [#26492](https://github.com/BerriAI/litellm/pull/26492) | [Fix] Tighten caller-permission checks on key route fields | [BerriAI-litellm-pr-26492.md](2026-W17/drip-48/BerriAI-litellm-pr-26492.md) |
| [#26503](https://github.com/BerriAI/litellm/pull/26503) | [Fix] Enforce key.models / user.models on Bedrock passthrough routes | [BerriAI-litellm-pr-26503.md](2026-W17/drip-47/BerriAI-litellm-pr-26503.md) |
| [#26468](https://github.com/BerriAI/litellm/pull/26468) | [WIP] Add endpoint for bulk key updates for team | [BerriAI-litellm-pr-26468.md](2026-W17/drip-46/BerriAI-litellm-pr-26468.md) |
| [#26500](https://github.com/BerriAI/litellm/pull/26500) | [Fix] Wrap extra_body for JSON-configured OpenAI-compatible providers | [BerriAI-litellm-pr-26500.md](2026-W17/drip-44/BerriAI-litellm-pr-26500.md) |
| [#26499](https://github.com/BerriAI/litellm/pull/26499) | fix(auth): join team-member budget so rpm/tpm limits are enforced | [BerriAI-litellm-pr-26499.md](2026-W17/drip-44/BerriAI-litellm-pr-26499.md) |
| [#26455](https://github.com/BerriAI/litellm/pull/26455) | feat: per-model team member budgets (project routes + GPT-5.5 rebase) | [BerriAI-litellm-pr-26455.md](2026-W17/drip-43/BerriAI-litellm-pr-26455.md) |
| [#26449](https://github.com/BerriAI/litellm/pull/26449) | [Feat] Day-0 support for GPT-5.5 and GPT-5.5 Pro | [BerriAI-litellm-pr-26449.md](2026-W17/drip-43/BerriAI-litellm-pr-26449.md) |
| [#26498](https://github.com/BerriAI/litellm/pull/26498) | fix(auth): apply temp_budget_increase on cache-hit path | [BerriAI-litellm-pr-26498.md](2026-W17/drip-42/BerriAI-litellm-pr-26498.md) |
| [#26495](https://github.com/BerriAI/litellm/pull/26495) | fix(health_check): drop max_tokens from non-chat handler params (closes #26406) | [BerriAI-litellm-pr-26495.md](2026-W17/drip-40/BerriAI-litellm-pr-26495.md) |
| [#26497](https://github.com/BerriAI/litellm/pull/26497) | fix(chatgpt): preserve text and parallel_tool_calls in responses | [BerriAI-litellm-pr-26497.md](2026-W17/drip-39/BerriAI-litellm-pr-26497.md) |
| [#26469](https://github.com/BerriAI/litellm/pull/26469) | [WIP] Cache LiteLLM_Config param reads in DualCache and batch | [BerriAI-litellm-pr-26469.md](2026-W17/drip-39/BerriAI-litellm-pr-26469.md) |
| [#26491](https://github.com/BerriAI/litellm/pull/26491) | [WIP] feat(tests): Claude Code Compatibility Matrix v0 (PRD #26476) | [BerriAI-litellm-pr-26491.md](2026-W17/drip-38/BerriAI-litellm-pr-26491.md) |
| [#26493](https://github.com/BerriAI/litellm/pull/26493) | [Fix] Extend caller-permission checks to service-account + tighten raw-body acceptance | [BerriAI-litellm-pr-26493.md](2026-W17/drip-37/BerriAI-litellm-pr-26493.md) |
| [#26490](https://github.com/BerriAI/litellm/pull/26490) | [Fix] Restrict /global/spend/* routes to admin roles | [BerriAI-litellm-pr-26490.md](2026-W17/drip-37/BerriAI-litellm-pr-26490.md) |
| [#26488](https://github.com/BerriAI/litellm/pull/26488) | [Feature] UI - Spend Logs: sortable Model and TTFT columns | [BerriAI-litellm-pr-26488.md](2026-W17/drip-37/BerriAI-litellm-pr-26488.md) |
| [#26489](https://github.com/BerriAI/litellm/pull/26489) | chore(vector-stores): redact credentials in list/info/update; gate update by per-store access | [BerriAI-litellm-pr-26489.md](2026-W17/drip-36/BerriAI-litellm-pr-26489.md) |
| [#26486](https://github.com/BerriAI/litellm/pull/26486) | chore(auth): admin-only gate on key_type, allowed_passthrough_routes, and key/regenerate grant fields | [BerriAI-litellm-pr-26486.md](2026-W17/drip-36/BerriAI-litellm-pr-26486.md) |
| [#26485](https://github.com/BerriAI/litellm/pull/26485) | feat: add FuturMix as named OpenAI-compatible provider | [BerriAI-litellm-pr-26485.md](2026-W17/drip-34/BerriAI-litellm-pr-26485.md) |
| [#26484](https://github.com/BerriAI/litellm/pull/26484) | chore(auth): substitute alias for master key on UserAPIKeyAuth | [BerriAI-litellm-pr-26484.md](2026-W17/drip-33/BerriAI-litellm-pr-26484.md) |
| [#26474](https://github.com/BerriAI/litellm/pull/26474) | fix(bedrock guardrails): collapse duplicate INPUT/OUTPUT post-call passes | [BerriAI-litellm-pr-26474.md](2026-W17/drip-32/BerriAI-litellm-pr-26474.md) |
| [#26471](https://github.com/BerriAI/litellm/pull/26471) | feat(teams): per-model team member budgets | [BerriAI-litellm-pr-26471.md](2026-W17/drip-32/BerriAI-litellm-pr-26471.md) |
| [#26472](https://github.com/BerriAI/litellm/pull/26472) | fix(bedrock): avoid duplicate post-call guardrail logs on streaming | [BerriAI-litellm-pr-26472.md](2026-W17/drip-30/BerriAI-litellm-pr-26472.md) |
| [#26470](https://github.com/BerriAI/litellm/pull/26470) | [Fix] Prevent atexit flush hangs and guard proxy_server_request header lookup | [BerriAI-litellm-pr-26470.md](2026-W17/drip-30/BerriAI-litellm-pr-26470.md) |
| [#26467](https://github.com/BerriAI/litellm/pull/26467) | fix(pass-through): harden target URL construction | [BerriAI-litellm-pr-26467.md](2026-W17/drip-30/BerriAI-litellm-pr-26467.md) |
| [#26456](https://github.com/BerriAI/litellm/pull/26456) | feat(models): gpt-5.5 reasoning_effort flags + supports_low_reasoning_effort | [BerriAI-litellm-pr-26456.md](2026-W17/drip-30/BerriAI-litellm-pr-26456.md) |
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
| [#26425](https://github.com/BerriAI/litellm/pull/26425) | fix(proxy): downgrade 401/403 auth denials from ERROR to WARNING in auth_exception_handler | [PR-26425-auth-401-403-warning-downgrade.md](BerriAI-litellm/PR-26425-auth-401-403-warning-downgrade.md) |
| [#26418](https://github.com/BerriAI/litellm/pull/26418) | fix(proxy): single-team DB fallback when JWT has no team_id | [PR-26418-jwt-single-team-db-fallback.md](BerriAI-litellm/PR-26418-jwt-single-team-db-fallback.md) |
| [#26426](https://github.com/BerriAI/litellm/pull/26426) | fix(vertex_ai/anthropic): stop stripping output_config | [BerriAI-litellm-pr-26426.md](2026-W17/BerriAI-litellm-pr-26426.md) |
| [#26427](https://github.com/BerriAI/litellm/pull/26427) | fix(model_management): refresh router after POST /model/update | [BerriAI-litellm-pr-26427.md](2026-W17/BerriAI-litellm-pr-26427.md) |
| [#26429](https://github.com/BerriAI/litellm/pull/26429) | fix: set verbose_logger level when LITELLM_LOG=INFO | [BerriAI-litellm-pr-26429.md](2026-W17/BerriAI-litellm-pr-26429.md) |
| [#26435](https://github.com/BerriAI/litellm/pull/26435) | fix: bump python-dotenv to 1.2.2 to fix CVE-2026-28684 | [PR-26435-python-dotenv-cve-bump.md](BerriAI-litellm/PR-26435-python-dotenv-cve-bump.md) |
| [#26434](https://github.com/BerriAI/litellm/pull/26434) | Fix/shared health check polling | [PR-26434-shared-health-check-polling.md](BerriAI-litellm/PR-26434-shared-health-check-polling.md) |
| [#26438](https://github.com/BerriAI/litellm/pull/26438) | fix(jwt-auth): apply team TPM/RPM to proxy_admin users acting on behalf of a team | [BerriAI-litellm-pr-26438.md](2026-W17/BerriAI-litellm-pr-26438.md) |
| [#26419](https://github.com/BerriAI/litellm/pull/26419) | fix(ui): add missing 'zai' (Z.AI / Zhipu AI) provider to Add-Model dropdown | [BerriAI-litellm-pr-26419.md](2026-W17/BerriAI-litellm-pr-26419.md) |
| [#26441](https://github.com/BerriAI/litellm/pull/26441) | fix(redis): cache GCP IAM token to prevent async event loop blocking | [BerriAI-litellm-pr-26441.md](2026-W17/drip-17/BerriAI-litellm-pr-26441.md) |
| [#26410](https://github.com/BerriAI/litellm/pull/26410) | fix: add vertex sonnet 4.6 1h cache pricing | [BerriAI-litellm-pr-26410.md](2026-W17/drip-17/BerriAI-litellm-pr-26410.md) |
| [#26442](https://github.com/BerriAI/litellm/pull/26442) | feat: restrict org admins from creating keys, teams, models via UI settings | [BerriAI-litellm-pr-26442.md](2026-W17/drip-17/BerriAI-litellm-pr-26442.md) |
| [#26439](https://github.com/BerriAI/litellm/pull/26439) | fix(adapters,vertex): pass output_config through to backends that accept it | [BerriAI-litellm-pr-26439.md](2026-W17/drip-19/BerriAI-litellm-pr-26439.md) |
| [#26405](https://github.com/BerriAI/litellm/pull/26405) | Fix: add fix to auth error (SSO callback OAuth-error surfacing) | [BerriAI-litellm-pr-26405.md](2026-W17/drip-21/BerriAI-litellm-pr-26405.md) |
| [#26445](https://github.com/BerriAI/litellm/pull/26445) | fix(anthropic): drop temperature from reasoning-family supported params | [BerriAI-litellm-pr-26445.md](2026-W17/drip-22/BerriAI-litellm-pr-26445.md) |
| [#26397](https://github.com/BerriAI/litellm/pull/26397) | fix(proxy): add verbose_logger to LITELLM_LOG=INFO branch | [BerriAI-litellm-pr-26397.md](2026-W17/drip-22/BerriAI-litellm-pr-26397.md) |
| [#26446](https://github.com/BerriAI/litellm/pull/26446) | fix(caching): handle list-based responses and message key variations in qdrant semantic cache | [BerriAI-litellm-pr-26446.md](2026-W17/drip-23/BerriAI-litellm-pr-26446.md) |
| [#26388](https://github.com/BerriAI/litellm/pull/26388) | fix: bedrock guardrail sse streaming exception | [BerriAI-litellm-pr-26388.md](2026-W17/drip-23/BerriAI-litellm-pr-26388.md) |
| [#26394](https://github.com/BerriAI/litellm/pull/26394) | docs(guardrails): add during_call mode to Model Armor guardrail docs | [BerriAI-litellm-pr-26394.md](2026-W17/drip-23/BerriAI-litellm-pr-26394.md) |
| [#26448](https://github.com/BerriAI/litellm/pull/26448) | fix(content_filter): log guardrail_information on streaming post-call | [BerriAI-litellm-pr-26448.md](2026-W17/drip-24/BerriAI-litellm-pr-26448.md) |
| [#26453](https://github.com/BerriAI/litellm/pull/26453) | Fix qdrant semantic cache | [BerriAI-litellm-pr-26453.md](2026-W17/drip-24/BerriAI-litellm-pr-26453.md) |
| [#26456](https://github.com/BerriAI/litellm/pull/26456) | fix: gpt-5.5 reasoning_effort=minimal silently dropped | [BerriAI-litellm-pr-26456.md](2026-W17/drip-25/BerriAI-litellm-pr-26456.md) |
| [#26452](https://github.com/BerriAI/litellm/pull/26452) | fix(ci): CircleCI rerun awk preprocessor | [BerriAI-litellm-pr-26452.md](2026-W17/drip-25/BerriAI-litellm-pr-26452.md) |
| [#26459](https://github.com/BerriAI/litellm/pull/26459) | [Fix] Reseed enforcement read path from DB on counter miss | [BerriAI-litellm-pr-26459.md](2026-W17/drip-26/BerriAI-litellm-pr-26459.md) |
| [#26451](https://github.com/BerriAI/litellm/pull/26451) | Azure Sentinel truncation + gzip + batch splitting | [BerriAI-litellm-pr-26451.md](2026-W17/drip-26/BerriAI-litellm-pr-26451.md) |
| [#26447](https://github.com/BerriAI/litellm/pull/26447) | New Relic AI Monitoring integration | [BerriAI-litellm-pr-26447.md](2026-W17/drip-27/BerriAI-litellm-pr-26447.md) |
| [#26461](https://github.com/BerriAI/litellm/pull/26461) | fix(ci): support CircleCI rerun failed tests for local_testing jobs | [BerriAI-litellm-pr-26461.md](2026-W17/drip-28/BerriAI-litellm-pr-26461.md) |
| [#26460](https://github.com/BerriAI/litellm/pull/26460) | feat(proxy): Add cleanup job for expired LiteLLM dashboard session keys | [BerriAI-litellm-pr-26460.md](2026-W17/drip-28/BerriAI-litellm-pr-26460.md) |
| [#26466](https://github.com/BerriAI/litellm/pull/26466) | fix(guardrails): team-level guardrails auto-apply alongside global policy guardrails | [BerriAI-litellm-pr-26466.md](2026-W17/drip-29/BerriAI-litellm-pr-26466.md) |
| [#26464](https://github.com/BerriAI/litellm/pull/26464) | Harden team metadata handling in /team/new and /team/update | [BerriAI-litellm-pr-26464.md](2026-W17/drip-29/BerriAI-litellm-pr-26464.md) |
| [#26463](https://github.com/BerriAI/litellm/pull/26463) | fix(mcp): tighten public-route detection and OAuth2 fallback gating | [BerriAI-litellm-pr-26463.md](2026-W17/drip-29/BerriAI-litellm-pr-26463.md) |
| [#26511](https://github.com/BerriAI/litellm/pull/26511) | ci: add supply-chain guard to block fork PRs that modify dependencies | [BerriAI-litellm-pr-26511.md](2026-W17/drip-54/BerriAI-litellm-pr-26511.md) |

## browser-use/browser-use

| PR | Title | File |
|---|---|---|
| [#4717](https://github.com/browser-use/browser-use/pull/4717) | fix: video recording speed by implementing frame duplication | [browser-use-browser-use-pr-4717.md](2026-W17/drip-52/browser-use-browser-use-pr-4717.md) |
| [#4719](https://github.com/browser-use/browser-use/pull/4719) | refactor(schema): remove unreachable type branch entry | [browser-use-browser-use-pr-4719.md](2026-W17/drip-51/browser-use-browser-use-pr-4719.md) |
| [#4718](https://github.com/browser-use/browser-use/pull/4718) | fix: robust Docker detection using multi-signal heuristics | [browser-use-browser-use-pr-4718.md](2026-W17/drip-51/browser-use-browser-use-pr-4718.md) |
| [#4708](https://github.com/browser-use/browser-use/pull/4708) | fix: fail fast for unimplemented traces_dir | [browser-use-browser-use-pr-4708.md](2026-W17/drip-50/browser-use-browser-use-pr-4708.md) |
| [#4707](https://github.com/browser-use/browser-use/pull/4707) | test: cover BrowserSession.close before start | [browser-use-browser-use-pr-4707.md](2026-W17/drip-50/browser-use-browser-use-pr-4707.md) |
| [#4726](https://github.com/browser-use/browser-use/pull/4726) | feat: add Astraflow provider support | [browser-use-browser-use-pr-4726.md](2026-W17/drip-49/browser-use-browser-use-pr-4726.md) |
| [#4722](https://github.com/browser-use/browser-use/pull/4722) | Guard direct CDP helpers during reconnect | [browser-use-browser-use-pr-4722.md](2026-W17/drip-49/browser-use-browser-use-pr-4722.md) |
| [#4734](https://github.com/browser-use/browser-use/pull/4734) | fix(schema): remove unreachable 'type' key from validation fields list | [browser-use-browser-use-pr-4734.md](2026-W17/drip-48/browser-use-browser-use-pr-4734.md) |
| [#4723](https://github.com/browser-use/browser-use/pull/4723) | security: verify init template integrity | [browser-use-browser-use-pr-4723.md](2026-W17/drip-47/browser-use-browser-use-pr-4723.md) |
| [#4724](https://github.com/browser-use/browser-use/pull/4724) | fix: implement retry logic for captureScreenshot to avoid timeouts | [browser-use-browser-use-pr-4724.md](2026-W17/drip-46/browser-use-browser-use-pr-4724.md) |
| [#4741](https://github.com/browser-use/browser-use/pull/4741) | fix(anthropic-serializer): type tool_calls + raise on malformed data URL | [browser-use-pr-4741.md](2026-W17/drip-45/browser-use-pr-4741.md) |
| [#4727](https://github.com/browser-use/browser-use/pull/4727) | fix(browser): replace deprecated asyncio.get_event_loop() with get_running_loop() | [browser-use-pr-4727.md](2026-W17/drip-45/browser-use-pr-4727.md) |
| [#4731](https://github.com/browser-use/browser-use/pull/4731) | fix: remove dead code in optimize_schema() (closed; not actually dead) | [browser-use-browser-use-pr-4731.md](2026-W17/drip-44/browser-use-browser-use-pr-4731.md) |
| [#4728](https://github.com/browser-use/browser-use/pull/4728) | feat(cli): add --proxy-url flag for local Chromium sessions | [browser-use-browser-use-pr-4728.md](2026-W17/drip-43/browser-use-browser-use-pr-4728.md) |
| [#4735](https://github.com/browser-use/browser-use/pull/4735) | Harden init template integrity and improve cloud tunnel recovery | [browser-use-browser-use-pr-4735.md](2026-W17/drip-42/browser-use-browser-use-pr-4735.md) |
| [#4732](https://github.com/browser-use/browser-use/pull/4732) | feat: self-healing element recovery engine | [browser-use-browser-use-pr-4732.md](2026-W17/drip-42/browser-use-browser-use-pr-4732.md) |
| [#4737](https://github.com/browser-use/browser-use/pull/4737) | Fix Windows CLI session probe timeouts | [browser-use-browser-use-pr-4737.md](2026-W17/drip-41/browser-use-browser-use-pr-4737.md) |
| [#4736](https://github.com/browser-use/browser-use/pull/4736) | fix(element): redact sensitive values in fill() debug logging | [browser-use-browser-use-pr-4736.md](2026-W17/drip-41/browser-use-browser-use-pr-4736.md) |

## charmbracelet/crush

| PR | Title | File |
|---|---|---|
| [#2714](https://github.com/charmbracelet/crush/pull/2714) | update terminal notifier for macos | [charmbracelet-crush-pr-2714.md](2026-W17/drip-58/charmbracelet-crush-pr-2714.md) |
| [#2711](https://github.com/charmbracelet/crush/pull/2711) | fix(ui): refresh todo pills on every session update | [charmbracelet-crush-pr-2711.md](2026-W17/drip-57/charmbracelet-crush-pr-2711.md) |
| [#2710](https://github.com/charmbracelet/crush/pull/2710) | fix(agent): pass max output tokens to summary stream | [charmbracelet-crush-pr-2710.md](2026-W17/drip-56/charmbracelet-crush-pr-2710.md) |
| [#2709](https://github.com/charmbracelet/crush/pull/2709) | fix(agent,ui): persist terminal finish for Run and Summarize + scoped spinner stall-guard | [charmbracelet-crush-pr-2709.md](2026-W17/drip-56/charmbracelet-crush-pr-2709.md) |
| [#2693](https://github.com/charmbracelet/crush/pull/2693) | fix(mcp): expand environment variables in stdio MCP server args | [charmbracelet-crush-pr-2693.md](2026-W17/drip-40/charmbracelet-crush-pr-2693.md) |
| [#2702](https://github.com/charmbracelet/crush/pull/2702) | feat: super yollo (split yolo into yolo / super-yolo permission modes) | [charmbracelet-crush-pr-2702.md](2026-W17/drip-39/charmbracelet-crush-pr-2702.md) |
| [#2663](https://github.com/charmbracelet/crush/pull/2663) | fix(app): replace single events channel with pubsub.Broker for fan-out | [charmbracelet-crush-pr-2663.md](2026-W17/drip-38/charmbracelet-crush-pr-2663.md) |
| [#2706](https://github.com/charmbracelet/crush/pull/2706) | docs(contributing): inline tooling notes for git/gh | [charmbracelet-crush-pr-2706.md](2026-W17/drip-30/charmbracelet-crush-pr-2706.md) |
| [#2693](https://github.com/charmbracelet/crush/pull/2693) | fix(mcp): expand environment variables in stdio MCP server args | [charmbracelet-crush-pr-2693.md](2026-W17/drip-30/charmbracelet-crush-pr-2693.md) |
| [#2671](https://github.com/charmbracelet/crush/pull/2671) | fix: reduce `fetch` and `view` tools truncation size to 100KB | [charmbracelet-crush-pr-2671.md](2026-W17/drip-34/charmbracelet-crush-pr-2671.md) |
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
| [#2643](https://github.com/charmbracelet/crush/pull/2643) | fix: enable real-time reasoning display and implement missing toggle handler | [PR-2643-realtime-reasoning-display-toggle.md](charmbracelet-crush/PR-2643-realtime-reasoning-display-toggle.md) |
| [#2686](https://github.com/charmbracelet/crush/pull/2686) | feat: support Moonshot and Moonshot China API keys in config | [PR-2686-moonshot-cn-api-keys.md](charmbracelet-crush/PR-2686-moonshot-cn-api-keys.md) |
| [#2674](https://github.com/charmbracelet/crush/pull/2674) | config: APIURL/APIKEY env overrides with CRUSH_PROVIDER targeting | [charmbracelet-crush-pr-2674.md](2026-W17/charmbracelet-crush-pr-2674.md) |
| [#2679](https://github.com/charmbracelet/crush/pull/2679) | fix: reduce token usage, use short tool descriptions by default | [charmbracelet-crush-pr-2679.md](2026-W17/charmbracelet-crush-pr-2679.md) |
| [#2634](https://github.com/charmbracelet/crush/pull/2634) | fix(logs): guard against unsafe type assertions in printLogLine | [charmbracelet-crush-pr-2634.md](2026-W17/charmbracelet-crush-pr-2634.md) |
| [#2703](https://github.com/charmbracelet/crush/pull/2703) | fix(hyper): re-authorization flow not triggering on certain conditions | [PR-2703-reauth-flow-trigger.md](charmbracelet-crush/PR-2703-reauth-flow-trigger.md) |
| [#2498](https://github.com/charmbracelet/crush/pull/2498) | fix(lsp): replace sticky unavailable cache with retry backoff | [PR-2498-lsp-unavailable-retry-backoff.md](charmbracelet-crush/PR-2498-lsp-unavailable-retry-backoff.md) |
| [#2702](https://github.com/charmbracelet/crush/pull/2702) | feat: super yollo (dangerous-command warning in YOLO mode) | [PR-2702-super-yollo-dangerous-command-prompt.md](charmbracelet-crush/PR-2702-super-yollo-dangerous-command-prompt.md) |
| [#2605](https://github.com/charmbracelet/crush/pull/2605) | feat(config): add additional_dirs option for tool access | [charmbracelet-crush-pr-2605.md](2026-W17/charmbracelet-crush-pr-2605.md) |
| [#2601](https://github.com/charmbracelet/crush/pull/2601) | fix: refresh TUI when session is updated by an external process | [charmbracelet-crush-pr-2601.md](2026-W17/charmbracelet-crush-pr-2601.md) |
| [#2590](https://github.com/charmbracelet/crush/pull/2590) | feat(notification): notify using OSC sequences for ssh terminal | [charmbracelet-crush-pr-2590.md](2026-W17/drip-17/charmbracelet-crush-pr-2590.md) |
| [#2584](https://github.com/charmbracelet/crush/pull/2584) | feat(agent): allow user to configure agent model size | [charmbracelet-crush-pr-2584.md](2026-W17/drip-17/charmbracelet-crush-pr-2584.md) |
| [#2598](https://github.com/charmbracelet/crush/pull/2598) | feat: PreToolUse hook | [charmbracelet-crush-pr-2598.md](2026-W17/drip-17/charmbracelet-crush-pr-2598.md) |
| [#2606](https://github.com/charmbracelet/crush/pull/2606) | feat: split-pane tree, tab manager, and cross-platform PTY | [charmbracelet-crush-pr-2606.md](2026-W17/drip-17/charmbracelet-crush-pr-2606.md) |
| [#2620](https://github.com/charmbracelet/crush/pull/2620) | feat(cmd): add `crush skills list` command with group-by-source support | [charmbracelet-crush-pr-2620.md](2026-W17/drip-19/charmbracelet-crush-pr-2620.md) |
| [#2646](https://github.com/charmbracelet/crush/pull/2646) | feat(ui): add CamelHumps editing for ctrl word shortcuts | [charmbracelet-crush-pr-2646.md](2026-W17/drip-20/charmbracelet-crush-pr-2646.md) |
| [#2612](https://github.com/charmbracelet/crush/pull/2612) | feat(hooks): implement JSON-based compatibility layer and lifecycle hooks | [charmbracelet-crush-pr-2612.md](2026-W17/drip-20/charmbracelet-crush-pr-2612.md) |
| [#2691](https://github.com/charmbracelet/crush/pull/2691) | fix(db): cap SQLite pool to one writer to prevent NOTADB corruption | [charmbracelet-crush-pr-2691.md](2026-W17/drip-21/charmbracelet-crush-pr-2691.md) |
| [#2609](https://github.com/charmbracelet/crush/pull/2609) | feat(session): add `session export` command for markdown/JSON export | [charmbracelet-crush-pr-2609.md](2026-W17/drip-21/charmbracelet-crush-pr-2609.md) |
| [#2593](https://github.com/charmbracelet/crush/pull/2593) | feat: add theme support with Charmtone default and Gruvbox Dark | [charmbracelet-crush-pr-2593.md](2026-W17/drip-22/charmbracelet-crush-pr-2593.md) |
| [#2575](https://github.com/charmbracelet/crush/pull/2575) | fix: correctly identify context fate in mcp createSession | [charmbracelet-crush-pr-2575.md](2026-W17/drip-22/charmbracelet-crush-pr-2575.md) |
| [#2656](https://github.com/charmbracelet/crush/pull/2656) | fix: use same chroma formatter as diffview for markdown | [charmbracelet-crush-pr-2656.md](2026-W17/drip-23/charmbracelet-crush-pr-2656.md) |
| [#2555](https://github.com/charmbracelet/crush/pull/2555) | fix: use small model for task agent instead of large | [charmbracelet-crush-pr-2555.md](2026-W17/drip-27/charmbracelet-crush-pr-2555.md) |
| [#2699](https://github.com/charmbracelet/crush/pull/2699) | fix(lsp): enforce workspace boundary for workspace edits | [charmbracelet-crush-pr-2699.md](2026-W17/drip-29/charmbracelet-crush-pr-2699.md) |

## cline/cline

| PR | Title | File |
|---|---|---|
| [#10382](https://github.com/cline/cline/pull/10382) | fix(hooks): Use shell escapes on JSON literals in hooks templates | [cline-cline-pr-10382.md](2026-W17/drip-48/cline-cline-pr-10382.md) |
| [#10400](https://github.com/cline/cline/pull/10400) | [Aikido] Fix 26 security issues in node-forge, @xmldom/xmldom, basic-ftp and 7 more | [cline-cline-pr-10400.md](2026-W17/drip-47/cline-cline-pr-10400.md) |
| [#10404](https://github.com/cline/cline/pull/10404) | feat(deepseek): deepseek-v4-pro supports reasoning effort control | [cline-cline-pr-10404.md](2026-W17/drip-46/cline-cline-pr-10404.md) |
| [#10406](https://github.com/cline/cline/pull/10406) | docs: Add FuturMix AI Gateway setup guide | [cline-cline-pr-10406.md](2026-W17/drip-45/cline-cline-pr-10406.md) |
| [#10380](https://github.com/cline/cline/pull/10380) | [Aikido] Fix security issue in dompurify via minor version upgrade from 3.3.3 to 3.4.1 in webview-ui | [cline-cline-pr-10380.md](2026-W17/drip-42/cline-cline-pr-10380.md) |
| [#10403](https://github.com/cline/cline/pull/10403) | feat: add Abliteration.ai provider | [cline-cline-pr-10403.md](2026-W17/drip-40/cline-cline-pr-10403.md) |
| [#10384](https://github.com/cline/cline/pull/10384) | fix: cap retry-after delay to prevent silent multi-hour hangs | [cline-cline-pr-10384.md](2026-W17/drip-39/cline-cline-pr-10384.md) |
| [#10397](https://github.com/cline/cline/pull/10397) | feat: add API key field to LM Studio provider | [cline-cline-pr-10397.md](2026-W17/drip-38/cline-cline-pr-10397.md) |
| [#10401](https://github.com/cline/cline/pull/10401) | feat(deepseek): Add deepseek-v4-flash and deepseek-v4-pro support | [cline-cline-pr-10401.md](2026-W17/drip-38/cline-cline-pr-10401.md) |
| [#10369](https://github.com/cline/cline/pull/10369) | fix(ollama): strip data URI prefix from images for Ollama API compatibility | [cline-cline-pr-10369.md](2026-W17/drip-37/cline-cline-pr-10369.md) |
| [#10396](https://github.com/cline/cline/pull/10396) | fix: respect image support toggle for paste and drag-drop operations | [cline-cline-pr-10396.md](2026-W17/drip-36/cline-cline-pr-10396.md) |
| [#10385](https://github.com/cline/cline/pull/10385) | fix(claude-code): handle CLI v2.1+ result chunks and error_max_turns | [cline-cline-pr-10385.md](2026-W17/drip-36/cline-cline-pr-10385.md) |
| [#10210](https://github.com/cline/cline/pull/10210) | Remove `/deep-planning` built-in slash command | [PR-10210.md](cline-cline/PR-10210.md) |
| [#10254](https://github.com/cline/cline/pull/10254) | fix: use deterministic keys for MCP server tool routing | [PR-10254.md](cline-cline/PR-10254.md) |
| [#10266](https://github.com/cline/cline/pull/10266) | Fix cache reflection for Cline and Vercel handlers | [PR-10266.md](cline-cline/PR-10266.md) |
| [#10269](https://github.com/cline/cline/pull/10269) | fix: unblock stuck command_output ask when terminal command ends | [PR-10269.md](cline-cline/PR-10269.md) |
| [#10283](https://github.com/cline/cline/pull/10283) | feat: wire up remote globalSkills with enterprise UI and architectural fixes | [PR-10283.md](cline-cline/PR-10283.md) |
| [#10329](https://github.com/cline/cline/pull/10329) | feat(cost-control): enforce third-party API spend limits | [PR-10329.md](cline-cline/PR-10329.md) |
| [#10286](https://github.com/cline/cline/pull/10286) | feat(models): prepare Claude Opus 4.7 provider support | [PR-10286.md](cline-cline/PR-10286.md) |
| [#10291](https://github.com/cline/cline/pull/10291) | fix: stabilize flaky Windows CI test paths | [PR-10291.md](cline-cline/PR-10291.md) |
| [#10343](https://github.com/cline/cline/pull/10343) | feat(memory-observability): add periodic memory logging to cline-core | [PR-10343.md](cline-cline/PR-10343.md) |
| [#10376](https://github.com/cline/cline/pull/10376) | fix: prevent path traversal in ClineIgnoreController.validateAccess | [cline-cline-pr-10376.md](2026-W17/drip-35/cline-cline-pr-10376.md) |
| [#10377](https://github.com/cline/cline/pull/10377) | fix: expose Plan/Act mode as an accessible radio group | [cline-cline-pr-10377.md](2026-W17/drip-35/cline-cline-pr-10377.md) |
| [#10384](https://github.com/cline/cline/pull/10384) | fix: cap retry-after delay to prevent silent multi-hour hangs | [cline-cline-pr-10384.md](2026-W17/drip-35/cline-cline-pr-10384.md) |
| [#10386](https://github.com/cline/cline/pull/10386) | fix: preserve multiline thinking blocks | [cline-cline-pr-10386.md](2026-W17/drip-35/cline-cline-pr-10386.md) |
| [#10242](https://github.com/cline/cline/pull/10242) | fix: quote file mention paths containing spaces | [cline-cline-pr-10242.md](2026-W17/drip-54/cline-cline-pr-10242.md) |
| [#10243](https://github.com/cline/cline/pull/10243) | fix: DiffViewProvider — await scroll, accurate line count, BOM preservation | [cline-cline-pr-10243.md](2026-W17/drip-54/cline-cline-pr-10243.md) |
| [#10331](https://github.com/cline/cline/pull/10331) | feat(ui): redesign chat toolbar with popup panels | [cline-cline-pr-10331.md](2026-W17/drip-54/cline-cline-pr-10331.md) |

## continuedev/continue

| PR | Title | File |
|---|---|---|
| [#12200](https://github.com/continuedev/continue/pull/12200) | Added new supported models, and removed deprecated models for Ask Sage | [continuedev-continue-pr-12200.md](2026-W17/drip-48/continuedev-continue-pr-12200.md) |
| [#12201](https://github.com/continuedev/continue/pull/12201) | chore(deps): bump dompurify from 3.3.3 to 3.4.1 in /gui | [continuedev-continue-pr-12201.md](2026-W17/drip-47/continuedev-continue-pr-12201.md) |
| [#12158](https://github.com/continuedev/continue/pull/12158) | fix(ollama): add Gemma to tool support heuristic for Ollama and LM Studio | [continuedev-continue-pr-12158.md](2026-W17/drip-46/continuedev-continue-pr-12158.md) |
| [#12151](https://github.com/continuedev/continue/pull/12151) | fix(gui): ignore stale provider model fetch results after provider switch | [continuedev-continue-pr-12151.md](2026-W17/drip-46/continuedev-continue-pr-12151.md) |
| [#12202](https://github.com/continuedev/continue/pull/12202) | update broken documentation references | [continuedev-continue-pr-12202.md](2026-W17/drip-45/continuedev-continue-pr-12202.md) |
| [#12220](https://github.com/continuedev/continue/pull/12220) | feat: add FuturMix as a model provider | [continuedev-continue-pr-12220.md](2026-W17/drip-42/continuedev-continue-pr-12220.md) |
| [#12219](https://github.com/continuedev/continue/pull/12219) | feat(llm): add Doubao (Volcengine Ark) as an LLM provider | [continuedev-continue-pr-12219.md](2026-W17/drip-41/continuedev-continue-pr-12219.md) |
| [#12206](https://github.com/continuedev/continue/pull/12206) | fix: When AGENTS.md does not exist, the traversal will be interrupted | [continuedev-continue-pr-12206.md](2026-W17/drip-40/continuedev-continue-pr-12206.md) |
| [#12216](https://github.com/continuedev/continue/pull/12216) | docs(openrouter): document automatic identification headers | [continuedev-continue-pr-12216.md](2026-W17/drip-39/continuedev-continue-pr-12216.md) |
| [#12212](https://github.com/continuedev/continue/pull/12212) | fix(bedrock): opt into httpBearerAuth when apiKey is set | [continuedev-continue-pr-12212.md](2026-W17/drip-38/continuedev-continue-pr-12212.md) |
| [#12190](https://github.com/continuedev/continue/pull/12190) | fix: use x-goog-api-key header instead of URL query param for Gemini API | [continuedev-continue-pr-12190.md](2026-W17/drip-35/continuedev-continue-pr-12190.md) |
| [#12198](https://github.com/continuedev/continue/pull/12198) | fix(cli): display full model name in TUI without trimming at slash delimiter | [continuedev-continue-pr-12198.md](2026-W17/drip-35/continuedev-continue-pr-12198.md) |

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
| [#19593](https://github.com/openai/codex/pull/19593) | test: isolate remote thread store regression from plugin warmups | [openai-codex-pr-19593.md](2026-W17/drip-59/openai-codex-pr-19593.md) |
| [#19578](https://github.com/openai/codex/pull/19578) | fix: increase Bazel timeout to 45 minutes | [openai-codex-pr-19578.md](2026-W17/drip-59/openai-codex-pr-19578.md) |
| [#19597](https://github.com/openai/codex/pull/19597) | Fix TUI attach fallback test stack overflow | [openai-codex-pr-19597.md](2026-W17/drip-57/openai-codex-pr-19597.md) |
| [#19595](https://github.com/openai/codex/pull/19595) | [codex] Bypass managed network for escalated exec | [openai-codex-pr-19595.md](2026-W17/drip-57/openai-codex-pr-19595.md) |
| [#19591](https://github.com/openai/codex/pull/19591) | Fix TUI resume performance regression | [openai-codex-pr-19591.md](2026-W17/drip-56/openai-codex-pr-19591.md) |
| [#19589](https://github.com/openai/codex/pull/19589) | Fix request_permissions tool flake in core tests | [openai-codex-pr-19589.md](2026-W17/drip-56/openai-codex-pr-19589.md) |
| [#18901](https://github.com/openai/codex/pull/18901) | Install standalone archives with checksum verification | [openai-codex-pr-18901.md](2026-W17/drip-52/openai-codex-pr-18901.md) |
| [#18883](https://github.com/openai/codex/pull/18883) | [codex] fix network context removal | [openai-codex-pr-18883.md](2026-W17/drip-52/openai-codex-pr-18883.md) |
| [#19575](https://github.com/openai/codex/pull/19575) | Add cloud executor registration to exec-server | [openai-codex-pr-19575.md](2026-W17/drip-51/openai-codex-pr-19575.md) |
| [#19160](https://github.com/openai/codex/pull/19160) | Make apply_patch streaming parser stateful | [openai-codex-pr-19160.md](2026-W17/drip-50/openai-codex-pr-19160.md) |
| [#19082](https://github.com/openai/codex/pull/19082) | Drop duplicate contiguous user messages during compaction | [openai-codex-pr-19082.md](2026-W17/drip-50/openai-codex-pr-19082.md) |
| [#19498](https://github.com/openai/codex/pull/19498) | Streamline review and feedback handlers | [openai-codex-pr-19498.md](2026-W17/drip-49/openai-codex-pr-19498.md) |
| [#19481](https://github.com/openai/codex/pull/19481) | Remove ghost snapshots | [openai-codex-pr-19481.md](2026-W17/drip-49/openai-codex-pr-19481.md) |
| [#19470](https://github.com/openai/codex/pull/19470) | Add turn start timestamp to turn metadata | [openai-codex-pr-19470.md](2026-W17/drip-48/openai-codex-pr-19470.md) |
| [#19468](https://github.com/openai/codex/pull/19468) | Fix Bazel cargo_bin runfiles paths | [openai-codex-pr-19468.md](2026-W17/drip-48/openai-codex-pr-19468.md) |
| [#19442](https://github.com/openai/codex/pull/19442) | feat: apply provider capability disables through config layers | [openai-codex-pr-19442.md](2026-W17/drip-47/openai-codex-pr-19442.md) |
| [#19456](https://github.com/openai/codex/pull/19456) | Add remote plugin uninstall API | [openai-codex-pr-19456.md](2026-W17/drip-46/openai-codex-pr-19456.md) |
| [#19514](https://github.com/openai/codex/pull/19514) | Fix codex-rs README grammar | [openai-codex-pr-19514.md](2026-W17/drip-44/openai-codex-pr-19514.md) |
| [#19491](https://github.com/openai/codex/pull/19491) | Streamline account and command handlers | [openai-codex-pr-19491.md](2026-W17/drip-43/openai-codex-pr-19491.md) |
| [#19537](https://github.com/openai/codex/pull/19537) | Add plugin MCP policy persistence | [openai-codex-pr-19537.md](2026-W17/drip-41/openai-codex-pr-19537.md) |
| [#19526](https://github.com/openai/codex/pull/19526) | [codex] Order codex-mcp items by visibility | [openai-codex-pr-19526.md](2026-W17/drip-40/openai-codex-pr-19526.md) |
| [#19524](https://github.com/openai/codex/pull/19524) | [codex] Minimize codex-mcp public surface | [openai-codex-pr-19524.md](2026-W17/drip-39/openai-codex-pr-19524.md) |
| [#19511](https://github.com/openai/codex/pull/19511) | Keep slash command popup columns stable while scrolling | [openai-codex-pr-19511.md](2026-W17/drip-39/openai-codex-pr-19511.md) |
| [#19513](https://github.com/openai/codex/pull/19513) | Delay approval prompts while typing | [openai-codex-pr-19513.md](2026-W17/drip-38/openai-codex-pr-19513.md) |
| [#19490](https://github.com/openai/codex/pull/19490) | Streamline plugin, apps, and skills handlers | [openai-codex-pr-19490.md](2026-W17/drip-38/openai-codex-pr-19490.md) |
| [#19510](https://github.com/openai/codex/pull/19510) | Hide rewind preview when no user message exists | [openai-codex-pr-19510.md](2026-W17/drip-37/openai-codex-pr-19510.md) |
| [#19509](https://github.com/openai/codex/pull/19509) | Record MCP tool result telemetry on spans | [openai-codex-pr-19509.md](2026-W17/drip-37/openai-codex-pr-19509.md) |
| [#19494](https://github.com/openai/codex/pull/19494) | Streamline thread read handlers | [openai-codex-pr-19494.md](2026-W17/drip-37/openai-codex-pr-19494.md) |
| [#19506](https://github.com/openai/codex/pull/19506) | [codex] Refresh AGENTS.md on cwd changes | [openai-codex-pr-19506.md](2026-W17/drip-36/openai-codex-pr-19506.md) |
| [#19495](https://github.com/openai/codex/pull/19495) | Streamline thread resume and fork handlers | [openai-codex-pr-19495.md](2026-W17/drip-36/openai-codex-pr-19495.md) |
| [#19492](https://github.com/openai/codex/pull/19492) | Streamline thread start handler | [openai-codex-pr-19492.md](2026-W17/drip-36/openai-codex-pr-19492.md) |
| [#19496](https://github.com/openai/codex/pull/19496) | Streamline MCP handlers | [openai-codex-pr-19496.md](2026-W17/drip-33/openai-codex-pr-19496.md) |
| [#19493](https://github.com/openai/codex/pull/19493) | Streamline thread mutation handlers | [openai-codex-pr-19493.md](2026-W17/drip-33/openai-codex-pr-19493.md) |
| [#19457](https://github.com/openai/codex/pull/19457) | Centralize thread git metadata overlays | [openai-codex-pr-19457.md](2026-W17/drip-34/openai-codex-pr-19457.md) |
| [#19498](https://github.com/openai/codex/pull/19498) | Streamline conversation init and resume handlers | [openai-codex-pr-19498.md](2026-W17/drip-32/openai-codex-pr-19498.md) |
| [#19497](https://github.com/openai/codex/pull/19497) | Streamline turn and realtime handlers | [openai-codex-pr-19497.md](2026-W17/drip-32/openai-codex-pr-19497.md) |
| [#19484](https://github.com/openai/codex/pull/19484) | Centralize JsonRpc error_code constructors and trim leaf handlers | [openai-codex-pr-19484.md](2026-W17/drip-32/openai-codex-pr-19484.md) |
| [#19487](https://github.com/openai/codex/pull/19487) | [codex] Move config loading into codex-config | [openai-codex-pr-19487.md](2026-W17/drip-30/openai-codex-pr-19487.md) |
| [#19481](https://github.com/openai/codex/pull/19481) | Remove ghost reasoning snapshots from Responses API turns | [openai-codex-pr-19481.md](2026-W17/drip-30/openai-codex-pr-19481.md) |
| [#19471](https://github.com/openai/codex/pull/19471) | test: gate Windows sandbox tests behind serial_test | [openai-codex-pr-19471.md](2026-W17/drip-30/openai-codex-pr-19471.md) |
| [#19458](https://github.com/openai/codex/pull/19458) | feat(chatgpt-library): file upload/download hooks | [openai-codex-pr-19458.md](2026-W17/drip-30/openai-codex-pr-19458.md) |
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
| [#19240](https://github.com/openai/codex/pull/19240) | fix: allow AgentIdentity through Apps MCP gates | [PR-19240-agent-identity-apps-mcp-gates.md](openai-codex/PR-19240-agent-identity-apps-mcp-gates.md) |
| [#19211](https://github.com/openai/codex/pull/19211) | [codex] Fix Windows process management edge cases | [PR-19211-windows-process-edge-cases.md](openai-codex/PR-19211-windows-process-edge-cases.md) |
| [#19193](https://github.com/openai/codex/pull/19193) | Support Codex Apps auth elicitations | [PR-19193-codex-apps-auth-elicitations.md](openai-codex/PR-19193-codex-apps-auth-elicitations.md) |
| [#19218](https://github.com/openai/codex/pull/19218) | cli: add macOS seatbelt flags for Mach services and Apple events | [openai-codex-pr-19218.md](2026-W17/openai-codex-pr-19218.md) |
| [#19246](https://github.com/openai/codex/pull/19246) | Increase app-server WebSocket outbound buffer | [openai-codex-pr-19246.md](2026-W17/openai-codex-pr-19246.md) |
| [#19351](https://github.com/openai/codex/pull/19351) | Add agents.interrupt_message for interruption markers | [openai-codex-pr-19351.md](2026-W17/openai-codex-pr-19351.md) |
| [#19360](https://github.com/openai/codex/pull/19360) | feat: surface multi-agent thread limit in spawn description | [openai-codex-pr-19360.md](2026-W17/openai-codex-pr-19360.md) |
| [#19377](https://github.com/openai/codex/pull/19377) | feat: separate session_id and thread_id in Responses requests | [openai-codex-pr-19377.md](2026-W17/openai-codex-pr-19377.md) |
| [#19207](https://github.com/openai/codex/pull/19207) | Forward Codex Apps tool call IDs to backend metadata | [openai-codex-pr-19207.md](2026-W17/openai-codex-pr-19207.md) |
| [#19350](https://github.com/openai/codex/pull/19350) | fix alpha build (entitlements trim) | [PR-19350-fix-alpha-build-entitlements.md](openai-codex/PR-19350-fix-alpha-build-entitlements.md) |
| [#19236](https://github.com/openai/codex/pull/19236) | Add instruction params to codex-app-server-test-client | [PR-19236-app-server-test-client-instructions.md](openai-codex/PR-19236-app-server-test-client-instructions.md) |
| [#19323](https://github.com/openai/codex/pull/19323) | Update models.json and related fixtures (default_reasoning_level shift) | [PR-19323-models-json-fixtures-update.md](openai-codex/PR-19323-models-json-fixtures-update.md) |
| [#19389](https://github.com/openai/codex/pull/19389) | Guard npm update prompt on registry readiness | [openai-codex-pr-19389.md](2026-W17/openai-codex-pr-19389.md) |
| [#19266](https://github.com/openai/codex/pull/19266) | [codex] add non-local thread store regression harness | [openai-codex-pr-19266.md](2026-W17/openai-codex-pr-19266.md) |
| [#19407](https://github.com/openai/codex/pull/19407) | Update bundled OpenAI Docs skill for GPT-5.5 | [openai-codex-pr-19407.md](2026-W17/drip-17/openai-codex-pr-19407.md) |
| [#19234](https://github.com/openai/codex/pull/19234) | Carve out log DB interfaces for new sinks | [openai-codex-pr-19234.md](2026-W17/drip-17/openai-codex-pr-19234.md) |
| [#19410](https://github.com/openai/codex/pull/19410) | Remove js_repl feature | [openai-codex-pr-19410.md](2026-W17/drip-17/openai-codex-pr-19410.md) |
| [#19395](https://github.com/openai/codex/pull/19395) | permissions: finish profile-backed app surfaces | [openai-codex-pr-19395.md](2026-W17/drip-17/openai-codex-pr-19395.md) |
| [#19394](https://github.com/openai/codex/pull/19394) | permissions: remove core legacy policy round trips | [openai-codex-pr-19394.md](2026-W17/drip-19/openai-codex-pr-19394.md) |
| [#19224](https://github.com/openai/codex/pull/19224) | Bump phase-two default model to gpt-5.5 | [openai-codex-pr-19224.md](2026-W17/drip-19/openai-codex-pr-19224.md) |
| [#19414](https://github.com/openai/codex/pull/19414) | permissions: make legacy profile conversion cwd-free | [openai-codex-pr-19414.md](2026-W17/drip-20/openai-codex-pr-19414.md) |
| [#19393](https://github.com/openai/codex/pull/19393) | permissions: migrate approval and sandbox consumers to profiles | [openai-codex-pr-19393.md](2026-W17/drip-20/openai-codex-pr-19393.md) |
| [#19280](https://github.com/openai/codex/pull/19280) | [codex] Migrate thread turns list to thread store | [openai-codex-pr-19280.md](2026-W17/drip-20/openai-codex-pr-19280.md) |
| [#19209](https://github.com/openai/codex/pull/19209) | [codex] Emit usage-limit prompt analytics from TUI | [openai-codex-pr-19209.md](2026-W17/drip-20/openai-codex-pr-19209.md) |
| [#19169](https://github.com/openai/codex/pull/19169) | Stabilize plugin MCP tool listing test | [openai-codex-pr-19169.md](2026-W17/drip-20/openai-codex-pr-19169.md) |
| [#19424](https://github.com/openai/codex/pull/19424) | Strip sandbox access defaults from app-server JSON schema | [openai-codex-pr-19424.md](2026-W17/drip-21/openai-codex-pr-19424.md) |
| [#19392](https://github.com/openai/codex/pull/19392) | permissions: derive compatibility policies from profiles | [openai-codex-pr-19392.md](2026-W17/drip-21/openai-codex-pr-19392.md) |
| [#19229](https://github.com/openai/codex/pull/19229) | Add agent graph store interface | [openai-codex-pr-19229.md](2026-W17/drip-21/openai-codex-pr-19229.md) |
| [#19391](https://github.com/openai/codex/pull/19391) | permissions: runtime config profile-backed | [openai-codex-pr-19391.md](2026-W17/drip-22/openai-codex-pr-19391.md) |
| [#19422](https://github.com/openai/codex/pull/19422) | Clarify bundled OpenAI Docs upgrade guide wording | [openai-codex-pr-19422.md](2026-W17/drip-22/openai-codex-pr-19422.md) |
| [#19176](https://github.com/openai/codex/pull/19176) | Add network proxy prompt guidance | [openai-codex-pr-19176.md](2026-W17/drip-23/openai-codex-pr-19176.md) |
| [#19431](https://github.com/openai/codex/pull/19431) | Route opted-in MCP elicitations through Guardian | [openai-codex-pr-19431.md](2026-W17/drip-24/openai-codex-pr-19431.md) |
| [#19432](https://github.com/openai/codex/pull/19432) | Add token usage to turn tracing spans | [openai-codex-pr-19432.md](2026-W17/drip-24/openai-codex-pr-19432.md) |
| [#19435](https://github.com/openai/codex/pull/19435) | Allow manual unified_exec opt-in on Windows | [openai-codex-pr-19435.md](2026-W17/drip-24/openai-codex-pr-19435.md) |
| [#19449](https://github.com/openai/codex/pull/19449) | permissions: remove legacy read-only access modes | [openai-codex-pr-19449.md](2026-W17/drip-25/openai-codex-pr-19449.md) |
| [#19447](https://github.com/openai/codex/pull/19447) | ci: publish codex-app-server release artifacts | [openai-codex-pr-19447.md](2026-W17/drip-25/openai-codex-pr-19447.md) |
| [#19452](https://github.com/openai/codex/pull/19452) | Stabilize plugin MCP fixture tests | [openai-codex-pr-19452.md](2026-W17/drip-26/openai-codex-pr-19452.md) |
| [#19454](https://github.com/openai/codex/pull/19454) | Split approval matrix test groups | [openai-codex-pr-19454.md](2026-W17/drip-26/openai-codex-pr-19454.md) |
| [#19453](https://github.com/openai/codex/pull/19453) | Serialize legacy Windows PowerShell sandbox tests | [openai-codex-pr-19453.md](2026-W17/drip-27/openai-codex-pr-19453.md) |
| [#19455](https://github.com/openai/codex/pull/19455) | Add gRPC feedback log sink | [openai-codex-pr-19455.md](2026-W17/drip-27/openai-codex-pr-19455.md) |
| [#19467](https://github.com/openai/codex/pull/19467) | feat: route MCP elicitations through guardian review | [openai-codex-pr-19467.md](2026-W17/drip-28/openai-codex-pr-19467.md) |
| [#19465](https://github.com/openai/codex/pull/19465) | Add gRPC feedback log sink | [openai-codex-pr-19465.md](2026-W17/drip-28/openai-codex-pr-19465.md) |
| [#19462](https://github.com/openai/codex/pull/19462) | sdk/python: use standalone codex-app-server runtime | [openai-codex-pr-19462.md](2026-W17/drip-28/openai-codex-pr-19462.md) |
| [#19461](https://github.com/openai/codex/pull/19461) | fix: Bedrock GPT-5.4 reasoning levels | [openai-codex-pr-19461.md](2026-W17/drip-28/openai-codex-pr-19461.md) |
| [#19459](https://github.com/openai/codex/pull/19459) | Enable unavailable dummy tools by default | [openai-codex-pr-19459.md](2026-W17/drip-28/openai-codex-pr-19459.md) |
| [#19474](https://github.com/openai/codex/pull/19474) | Make thread store process-scoped | [openai-codex-pr-19474.md](2026-W17/drip-29/openai-codex-pr-19474.md) |
| [#19473](https://github.com/openai/codex/pull/19473) | Add turn start timestamp to turn metadata | [openai-codex-pr-19473.md](2026-W17/drip-29/openai-codex-pr-19473.md) |

## ollama/ollama

| PR | Title | File |
|---|---|---|
| [#15717](https://github.com/ollama/ollama/pull/15717) | Add Spring AI Playground to Chat Interfaces (Desktop) | [ollama-ollama-pr-15717.md](2026-W17/drip-55/ollama-ollama-pr-15717.md) |
| [#15581](https://github.com/ollama/ollama/pull/15581) | ggml-metal: fix mixed bf16/f16 cooperative tensor operand order | [ollama-ollama-pr-15581.md](2026-W17/drip-55/ollama-ollama-pr-15581.md) |
| [#15814](https://github.com/ollama/ollama/pull/15814) | Model support for batching | [ollama-ollama-pr-15814.md](2026-W17/drip-53/ollama-ollama-pr-15814.md) |
| [#15726](https://github.com/ollama/ollama/pull/15726) | fix: resolve gateway launch timeout on Windows | [ollama-ollama-pr-15726.md](2026-W17/drip-53/ollama-ollama-pr-15726.md) |
| [#15787](https://github.com/ollama/ollama/pull/15787) | api: accept "max" as a think value | [ollama-ollama-pr-15787.md](2026-W17/drip-48/ollama-ollama-pr-15787.md) |
| [#15736](https://github.com/ollama/ollama/pull/15736) | mlxrunner: batch the sampler across multiple sequences | [ollama-ollama-pr-15736.md](2026-W17/drip-47/ollama-ollama-pr-15736.md) |
| [#15811](https://github.com/ollama/ollama/pull/15811) | docs: add RDNA4 / gfx1201 ROCm build instructions to development.md | [ollama-ollama-pr-15811.md](2026-W17/drip-45/ollama-ollama-pr-15811.md) |
| [#15710](https://github.com/ollama/ollama/pull/15710) | docs: fix tar decompression flags | [ollama-ollama-pr-15710.md](2026-W17/drip-43/ollama-ollama-pr-15710.md) |
| [#15809](https://github.com/ollama/ollama/pull/15809) | create: prune imported blobs and startup invalid leftovers | [ollama-ollama-pr-15809.md](2026-W17/drip-41/ollama-ollama-pr-15809.md) |
| [#15789](https://github.com/ollama/ollama/pull/15789) | openai: map responses reasoning effort to think | [ollama-ollama-pr-15789.md](2026-W17/drip-41/ollama-ollama-pr-15789.md) |
| [#15808](https://github.com/ollama/ollama/pull/15808) | fix: improve error handling for model loading in Scheduler | [ollama-ollama-pr-15808.md](2026-W17/drip-40/ollama-ollama-pr-15808.md) |
| [#15790](https://github.com/ollama/ollama/pull/15790) | Add Atlarix to Code Editors & Development integrations | [ollama-ollama-pr-15790.md](2026-W17/drip-37/ollama-ollama-pr-15790.md) |
| [#15735](https://github.com/ollama/ollama/pull/15735) | server: add v2 manifest path | [ollama-ollama-pr-15735.md](2026-W17/drip-36/ollama-ollama-pr-15735.md) |
| [#15805](https://github.com/ollama/ollama/pull/15805) | Add `ollama launch qwen` support for Qwen Code CLI | [ollama-ollama-pr-15805.md](2026-W17/drip-30/ollama-ollama-pr-15805.md) |
| [#15713](https://github.com/ollama/ollama/pull/15713) | fix: use MemAvailable equivalent in cgroup memory check | [ollama-ollama-pr-15713.md](2026-W17/drip-34/ollama-ollama-pr-15713.md) |
| [#15774](https://github.com/ollama/ollama/pull/15774) | Harden Qwen-family tool payload rendering and fix Qwen 3.5 tool-block truncation integrity | [ollama-ollama-pr-15774.md](2026-W17/drip-19/ollama-ollama-pr-15774.md) |
| [#15755](https://github.com/ollama/ollama/pull/15755) | metal: harden for ggml initialization failures | [ollama-ollama-pr-15755.md](2026-W17/drip-19/ollama-ollama-pr-15755.md) |
| [#15716](https://github.com/ollama/ollama/pull/15716) | server: fix download stall watchdog not firing when no bytes arrive | [ollama-ollama-pr-15716.md](2026-W17/drip-21/ollama-ollama-pr-15716.md) |
| [#15784](https://github.com/ollama/ollama/pull/15784) | Go sampler: implement repeat/freq/presence penalties | [ollama-ollama-pr-15784.md](2026-W17/drip-22/ollama-ollama-pr-15784.md) |
| [#15768](https://github.com/ollama/ollama/pull/15768) | feat(runner): expose prompt cache hit count in completion response | [ollama-ollama-pr-15768.md](2026-W17/drip-23/ollama-ollama-pr-15768.md) |
| [#15766](https://github.com/ollama/ollama/pull/15766) | server: populate parameter_size in /api/tags for safetensors models | [ollama-ollama-pr-15766.md](2026-W17/drip-24/ollama-ollama-pr-15766.md) |
| [#15773](https://github.com/ollama/ollama/pull/15773) | cmd: show help when stop and show are called without a model argument | [ollama-ollama-pr-15773.md](2026-W17/drip-24/ollama-ollama-pr-15773.md) |
| [#15793](https://github.com/ollama/ollama/pull/15793) | mlx: update to 0.31.2 | [ollama-ollama-pr-15793.md](2026-W17/drip-25/ollama-ollama-pr-15793.md) |
| [#15756](https://github.com/ollama/ollama/pull/15756) | app/ui: fix sidebar animating on initial load | [ollama-ollama-pr-15756.md](2026-W17/drip-25/ollama-ollama-pr-15756.md) |
| [#15795](https://github.com/ollama/ollama/pull/15795) | launch: add codex model metadata catalog | [ollama-ollama-pr-15795.md](2026-W17/drip-25/ollama-ollama-pr-15795.md) |
| [#15763](https://github.com/ollama/ollama/pull/15763) | Prevent system sleep during inference (fixes #4072) | [ollama-ollama-pr-15763.md](2026-W17/drip-26/ollama-ollama-pr-15763.md) |
| [#15724](https://github.com/ollama/ollama/pull/15724) | convert: support fp8 safetensors import | [ollama-ollama-pr-15724.md](2026-W17/drip-26/ollama-ollama-pr-15724.md) |
| [#15761](https://github.com/ollama/ollama/pull/15761) | x/mlxrunner/model: add baseline test coverage for quant metadata scanning | [ollama-ollama-pr-15761.md](2026-W17/drip-27/ollama-ollama-pr-15761.md) |
| [#15760](https://github.com/ollama/ollama/pull/15760) | x/mlxrunner: apply config.json per-tensor quant overrides for mixed-precision MoE | [ollama-ollama-pr-15760.md](2026-W17/drip-27/ollama-ollama-pr-15760.md) |
| [#15765](https://github.com/ollama/ollama/pull/15765) | docs: add HuggingFace Hub direct GGUF pull pattern to import guide | [ollama-ollama-pr-15765.md](2026-W17/drip-27/ollama-ollama-pr-15765.md) |
| [#15759](https://github.com/ollama/ollama/pull/15759) | x/mlxrunner: recognise mlx-lm plural aux naming at load time | [ollama-ollama-pr-15759.md](2026-W17/drip-28/ollama-ollama-pr-15759.md) |
| [#15705](https://github.com/ollama/ollama/pull/15705) | model/parsers/qwen3coder: prevent tag regex from matching across newlines | [ollama-ollama-pr-15705.md](2026-W17/drip-29/ollama-ollama-pr-15705.md) |
| [#15703](https://github.com/ollama/ollama/pull/15703) | model/renderers/gemma4: allow reserved JSON Schema keys as parameter names | [ollama-ollama-pr-15703.md](2026-W17/drip-29/ollama-ollama-pr-15703.md) |
| [#15664](https://github.com/ollama/ollama/pull/15664) | openai: honor reasoning_effort in /v1/responses endpoint | [ollama-ollama-pr-15664.md](2026-W17/drip-54/ollama-ollama-pr-15664.md) |
| [#15683](https://github.com/ollama/ollama/pull/15683) | server: preserve thinking in /api/generate; populate parameter_size in /api/tags | [ollama-ollama-pr-15683.md](2026-W17/drip-54/ollama-ollama-pr-15683.md) |
| [#15691](https://github.com/ollama/ollama/pull/15691) | add token calculation support with UI display | [ollama-ollama-pr-15691.md](2026-W17/drip-54/ollama-ollama-pr-15691.md) |
| [#15706](https://github.com/ollama/ollama/pull/15706) | Add LumaKit to Community Integrations | [ollama-ollama-pr-15706.md](2026-W17/drip-54/ollama-ollama-pr-15706.md) |

### W17 drip-60 (2026-04-26) — guardrail logging, provider-config preservation, schema-cleanup scope

8-PR sweep covering: a regression fix that suppressed duplicate (Success+Failure) spend logs when a post-call guardrail blocks; provider-config plumbing for Azure Cognitive Services `api-version` and configured-output-token caps in opencode; a UI fix in crush so disabled skills stop rendering as "online"; and three larger PRs flagged for scope creep (a Vertex `output_config` sanitizer bundled with unrelated Arize and price-table churn; a CI-infra PR carrying production guardrail rewrites; and a "stabilize Windows path tests" PR that also renames a production sandbox helper).

| PR | Title | Path |
|---|---|---|
| [#26528](https://github.com/BerriAI/litellm/pull/26528) | fix(proxy): suppress deferred success log when post-call guardrail blocks | [BerriAI-litellm/PR-26528.md](BerriAI-litellm/PR-26528.md) |
| [#26530](https://github.com/BerriAI/litellm/pull/26530) | fix(adapters,vertex): pass Anthropic `output_config` through to backends that accept it | [BerriAI-litellm/PR-26530.md](BerriAI-litellm/PR-26530.md) |
| [#26527](https://github.com/BerriAI/litellm/pull/26527) | test(ci): add dockerized E2E coverage for Anthropic-style client ↔ LiteLLM proxy | [BerriAI-litellm/PR-26527.md](BerriAI-litellm/PR-26527.md) |
| [#19605](https://github.com/openai/codex/pull/19605) | Delete unused ResponseItem::Message.end_turn | [openai-codex/PR-19605-delete-end-turn.md](openai-codex/PR-19605-delete-end-turn.md) |
| [#19604](https://github.com/openai/codex/pull/19604) | test: stabilize app-server path assertions on Windows | [openai-codex/PR-19604-windows-path-asserts.md](openai-codex/PR-19604-windows-path-asserts.md) |
| [#2712](https://github.com/charmbracelet/crush/pull/2712) | fix(ui): Skill display as loaded despite being disabled in config | [charmbracelet-crush/PR-2712.md](charmbracelet-crush/PR-2712.md) |
| [#24386](https://github.com/sst/opencode/pull/24386) | fix(provider): preserve Azure API version | [sst-opencode/PR-24386.md](sst-opencode/PR-24386.md) |
| [#24384](https://github.com/sst/opencode/pull/24384) | fix(provider): respect configured output limit | [sst-opencode/PR-24384.md](sst-opencode/PR-24384.md) |

### W17 drip-61 (2026-04-26) — MCP tool-name normalisation, dict-shaped UI inputs, shell-spawn pgroup safety

8-PR sweep across five repos (litellm × 3, opencode × 2, goose × 2, codex × 1) covering: a meaningful expansion of the litellm MCP semantic tool filter to recognise LibreChat-style `<canonical><sep><uid>` suffix mangling; a dashboard fix that stops React error #31 when fallback chain entries arrive as override-dicts (with secret-leak surface area flagged for confirmation); JSON-registered providers finally getting the same `extra_body` wrapping as the static `openai_compatible_providers` list; two opencode defensive-coding fixes (nullish guards on tool output, and an explicit allow-list for image MIME types); two goose subprocess fixes (drop interactive-shell SIGTTOU race; allow quoted-numeric YAML config values); and an `end_turn` plumbing PR in codex that is otherwise correct but ships with an author FIXME in the only human-visible CLI output path.

| PR | Title | Path |
|---|---|---|
| [#26533](https://github.com/BerriAI/litellm/pull/26533) | fix(proxy): handle client-side unique-ID suffixes in MCP semantic tool filter | [BerriAI-litellm/PR-26533.md](BerriAI-litellm/PR-26533.md) |
| [#26515](https://github.com/BerriAI/litellm/pull/26515) | fix(ui): render dict-shaped fallback entries in router settings | [BerriAI-litellm/PR-26515.md](BerriAI-litellm/PR-26515.md) |
| [#26500](https://github.com/BerriAI/litellm/pull/26500) | fix: wrap extra_body for JSON-configured OpenAI-compatible providers | [BerriAI-litellm/PR-26500.md](BerriAI-litellm/PR-26500.md) |
| [#24401](https://github.com/sst/opencode/pull/24401) | fix: guard against undefined MCP tool output causing output.split crash | [sst-opencode/PR-24401.md](sst-opencode/PR-24401.md) |
| [#24364](https://github.com/sst/opencode/pull/24364) | fix(provider): reject unsupported image mime types | [sst-opencode/PR-24364.md](sst-opencode/PR-24364.md) |
| [#8836](https://github.com/block/goose/pull/8836) | fix: drop -i flag from login shell PATH resolution to prevent SIGTTOU | [block-goose/PR-8836.md](block-goose/PR-8836.md) |
| [#8844](https://github.com/block/goose/pull/8844) | fix: convert quoted numeric config values to numbers if needed | [block-goose/PR-8844.md](block-goose/PR-8844.md) |
| [#19610](https://github.com/openai/codex/pull/19610) | Support end_turn in response.completed | [openai-codex/PR-19610-end-turn-completed.md](openai-codex/PR-19610-end-turn-completed.md) |

Verdict mix: 4 merge-as-is, 3 merge-after-nits, 1 request-changes (codex#19610 — author FIXME blocking).

### W17 drip-62 (2026-04-26) — Pydantic-coerced span attrs, Bedrock reasoning round-trip, panic→error in local-inference, profile-backed sandbox refactor

8-PR sweep across five repos (litellm × 1, opencode × 2, goose × 3, crush × 1, codex × 1). Themes: an Arize observability rewrite that fixes two real bugs (Pydantic responses being skipped; dict-shaped Responses-API output items silently dropped) but bundles them with a +500-line image-rendering rewrite worth splitting; a 2-line SDK fix that swaps `import default` for `createRequire` to survive Bun's no-default-interop ESM loader for `cross-spawn`; a TUI feature PR that ships Spanish strings and an unhashed lockfile entry alongside legitimate reconnecting-state plumbing; a careful Bedrock `ReasoningContent` round-trip that preserves the SDK's `non_exhaustive` boundary as a hard error rather than silent truncation; a panic→error refactor on the corrupted-permission-config path that still leaves an `expect()` on the directory-create branch and silently revokes prior tool approvals; a one-call-site `Result` propagation that removes a `panic!` from goose-server startup; an OAuth dialog clipboard delegation to a shared `CopyToClipboard` helper for non-OSC52 terminals; and a +1383/-655 codex permission-profile consolidation that's directionally right but renames `DangerFullAccess → Disabled` (telemetry-visible) and introduces a `compatibility_*` helper without a deprecation horizon.

| PR | Title | Path |
|---|---|---|
| [#26526](https://github.com/BerriAI/litellm/pull/26526) | fix arize observability bugs | [BerriAI-litellm/PR-26526.md](BerriAI-litellm/PR-26526.md) |
| [#24374](https://github.com/sst/opencode/pull/24374) | fix(sdk): load cross-spawn through require | [sst-opencode/PR-24374.md](sst-opencode/PR-24374.md) |
| [#24406](https://github.com/sst/opencode/pull/24406) | feat(tui): add unified task state color convention with icons | [sst-opencode/PR-24406.md](sst-opencode/PR-24406.md) |
| [#8843](https://github.com/block/goose/pull/8843) | fix(bedrock): handle ReasoningContent blocks gracefully | [block-goose/PR-8843.md](block-goose/PR-8843.md) |
| [#8813](https://github.com/block/goose/pull/8813) | fix: gracefully handle corrupted permission config instead of panicking | [block-goose/PR-8813.md](block-goose/PR-8813.md) |
| [#8814](https://github.com/block/goose/pull/8814) | fix: return error instead of panicking on llama backend init failure | [block-goose/PR-8814.md](block-goose/PR-8814.md) |
| [#2642](https://github.com/charmbracelet/crush/pull/2642) | fix(oauth): fix copy to clipboard on terminals that don't support osc52 | [charmbracelet-crush/PR-2642.md](charmbracelet-crush/PR-2642.md) |
| [#19606](https://github.com/openai/codex/pull/19606) | permissions: make runtime config profile-backed | [openai-codex/PR-19606-permissions-profile-backed.md](openai-codex/PR-19606-permissions-profile-backed.md) |

Verdict mix: 3 merge-as-is (opencode#24374, goose#8843, goose#8814), 3 merge-after-nits (litellm#26526, goose#8813, crush#2642), 1 request-changes (opencode#24406 — Spanish strings + unhashed lockfile + scope), 1 needs-discussion (codex#19606 — `Disabled` rename + `compatibility_*` helper deprecation horizon).

### W17 drip-63 (2026-04-26) — startup stdin buffering, Kilo reasoning_details typing, exp/iat double-gate, codex shell history + logo

8-PR sweep across three repos (opencode × 3, codex × 2, goose × 3). Themes: a TUI fix that buffers `process.stdin` between launch and React mount via a disposable factory plumbed through `PromptRefProvider`, with priority-ordering `route.prompt > args.prompt > buffered`; a provider-options branch in `normalizeMessages` that stops synthesizing string `reasoning_details` from concatenated `part.text` for openaiCompatible reasoning models (Kimi K2.6) and instead forwards the structured per-part array; a typed-HttpApi bridge for `POST /experimental/console/switch` and `GET /experimental/tool` that maps `account.use` failures to `BadRequest` and produces JSON-schema parameters via `EffectZod.toJsonSchema`; an OIDC-proxy security fix that flips `if/else if` to two independent gates so configuring `MAX_TOKEN_AGE_SECONDS` no longer disables `exp` enforcement (with 530 lines of new gate tests covering all four corner cases); a codex TUI feature that adds an 8-line ASCII codex logo with light/dark gradient adaptation, gated on terminal width with graceful fallback to the 56-column text-only header; a one-line moim whitelist addition for "Trimmed trailing whitespace from assistant message"; a context-window indicator that tiers Info → Warning → Error at 75% / 90% fill; and a codex shell-history persistence fix that emits `Op::AddToHistory` only on `QueueDrain::Stop` and renames "Bash mode" → "Shell mode" across the composer.

| PR | Title | Path |
|---|---|---|
| [#24412](https://github.com/sst/opencode/pull/24412) | fix: Buffer stdin before prompt UI appears | [2026-W17/sst-opencode-pr-24412.md](2026-W17/sst-opencode-pr-24412.md) |
| [#24411](https://github.com/sst/opencode/pull/24411) | fix(opencode): avoid invalid Kilo reasoning details | [2026-W17/sst-opencode-pr-24411.md](2026-W17/sst-opencode-pr-24411.md) |
| [#24407](https://github.com/sst/opencode/pull/24407) | feat(httpapi): bridge experimental tool routes | [2026-W17/sst-opencode-pr-24407.md](2026-W17/sst-opencode-pr-24407.md) |
| [#19618](https://github.com/openai/codex/pull/19618) | Persist shell mode commands in prompt history | [2026-W17/openai-codex-pr-19618.md](2026-W17/openai-codex-pr-19618.md) |
| [#19617](https://github.com/openai/codex/pull/19617) | feat(tui): add codex logo to session header | [2026-W17/openai-codex-pr-19617.md](2026-W17/openai-codex-pr-19617.md) |
| [#8851](https://github.com/block/goose/pull/8851) | colorize context window indicator | [2026-W17/block-goose-pr-8851.md](2026-W17/block-goose-pr-8851.md) |
| [#8847](https://github.com/block/goose/pull/8847) | Add "Trimmed trailing whitespace" message to moim whitelist | [2026-W17/block-goose-pr-8847.md](2026-W17/block-goose-pr-8847.md) |
| [#8839](https://github.com/block/goose/pull/8839) | fix(oidc-proxy): enforce exp independently of MAX_TOKEN_AGE_SECONDS | [2026-W17/block-goose-pr-8839.md](2026-W17/block-goose-pr-8839.md) |

Verdict mix: 3 merge-as-is (opencode#24401-style — wait, this batch: goose#8851, goose#8847, no third), 5 merge-after-nits (opencode#24412, opencode#24411, opencode#24407, codex#19618, codex#19617, goose#8839). Actual: 2 merge-as-is (goose#8847, goose#8851), 6 merge-after-nits.

### W17 drip-64 (2026-04-26) — ASCII-escaped headers, Prisma `Json?` jsonification, OAuth2 self-service auth, ACP+ system/git migrations, lifecycle hooks, SSE reconnect cap removal, Windows dev loop

8-PR sweep across three repos (codex × 1, litellm × 2, goose × 5). Themes: a custom `serde_json::Formatter` (`AsciiJsonFormatter` overriding only `write_string_fragment`) that escapes non-ASCII as `\uXXXX` so turn-metadata HTTP headers stay parseable JSON without dropping unicode workspace paths or values; a `_serialize_metadata_for_prisma` helper that unconditionally `json.dumps`-encodes `metadata` payloads (dict/list/scalar) on `/v1/memory` to work around prisma-client-python's `DataError` on `Json?` columns, with the `metadata: null` semantics flipped from "clear column" to "no-op" to match the rest of the proxy and a new 400 when `null` is the only field; an MCP OAuth2 self-service feature that extends `has_user_credential` resolution to `auth_type == "oauth2"` and adds a header-OR-cookie `_authorize_user_auth` fallback to support browser redirects (with cookie-CSRF threat-modeling open as a discussion item); a 7-RPC `_goose/system/*` ACP+ namespace covering home_dir, path_exists, list_directory_entries, inspect_attachment_paths, list_files_for_mentions, read_image_attachment, write_file (replacing equivalent Tauri commands); a 9-RPC `_goose/git/*` ACP+ migration covering state, changed_files, switch_branch, stash, init, fetch (--prune), pull (--ff-only), create_branch, create_worktree, with mutating ops returning `EmptyResponse` and forcing a follow-up state-read round-trip; a config-driven 6-event lifecycle hooks system (`before_tool_call`/`after_tool_call`/`before_reply` etc.) with regex matchers and command/url handlers — `block: true` short-circuits tool dispatch, but `after_tool_call` is documented as fire-and-forget while actually being awaited, `tool_result` is never populated, and zero tests ship for what is a security-sensitive subsystem; a SSE reconnect-cap removal that drops `MAX_CONSECUTIVE_ERRORS = 10` and the synthetic terminal `Error` broadcast, replacing it with infinite reconnect-with-backoff (correct because `Last-Event-ID` replay protocol handles recovery, the cap was conflating transport failure with protocol-fatal); and a Windows dev-loop fix bundling UTF-8 console reconfig, a `_resolve_goose_cmd()` that picks `$GOOSE_BIN` → prebuilt → `cargo run`, `stderr=DEVNULL` to avoid `cargo run` chatter deadlocking the JSON-RPC stdio loop, and Tauri `beforeDevCommand` rewrites that drop `&&`/`exec` and add `--host 127.0.0.1` for Vite 7's IPv6-only-on-Windows default.

| PR | Title | Path |
|---|---|---|
| [#19620](https://github.com/openai/codex/pull/19620) | Escape turn metadata headers as ASCII JSON | [2026-W17/openai-codex-pr-19620.md](2026-W17/openai-codex-pr-19620.md) |
| [#26536](https://github.com/BerriAI/litellm/pull/26536) | fix(memory): jsonify metadata before Prisma writes on /v1/memory | [2026-W17/BerriAI-litellm-pr-26536.md](2026-W17/BerriAI-litellm-pr-26536.md) |
| [#26531](https://github.com/BerriAI/litellm/pull/26531) | feat(mcp): OAuth2 self-service for internal users (LIT-2503) | [2026-W17/BerriAI-litellm-pr-26531.md](2026-W17/BerriAI-litellm-pr-26531.md) |
| [#8849](https://github.com/block/goose/pull/8849) | feat(acp): migrate system/file Tauri commands to ACP+ | [2026-W17/block-goose-pr-8849.md](2026-W17/block-goose-pr-8849.md) |
| [#8848](https://github.com/block/goose/pull/8848) | feat(goose2): migrate git commands from Tauri to ACP+ (#8698) | [2026-W17/block-goose-pr-8848.md](2026-W17/block-goose-pr-8848.md) |
| [#8846](https://github.com/block/goose/pull/8846) | fix(ui): keep SSE reconnect loop alive on long disconnects (#8717) | [2026-W17/block-goose-pr-8846.md](2026-W17/block-goose-pr-8846.md) |
| [#8842](https://github.com/block/goose/pull/8842) | feat: lifecycle hooks system | [2026-W17/block-goose-pr-8842.md](2026-W17/block-goose-pr-8842.md) |
| [#8834](https://github.com/block/goose/pull/8834) | Fix Windows dev loop: beforeDevCommand script + Vite IPv4 bind + test_acp_client.py stdin-flush | [2026-W17/block-goose-pr-8834.md](2026-W17/block-goose-pr-8834.md) |

Verdict mix: 1 merge-as-is (goose#8846), 6 merge-after-nits (codex#19620, litellm#26536, goose#8849, goose#8848, goose#8834 — and goose#8842 elevated below), 1 needs-discussion (litellm#26531 — cookie-fallback CSRF/SameSite story on a redirect endpoint), 1 request-changes (goose#8842 — `after_tool_call` docstring/impl disagreement, `tool_result` never populated, `command`/`url` mutual-exclusivity unenforced, zero tests for a security-sensitive subsystem).

### W17 drip-65 (2026-04-26) — Fireworks chat-transform deletions risk, MCP suffix-match symmetry, Arize observability rebuild w/ no tests, Bun ESM cross-spawn require, YAML quoted-numeric coercion, Bedrock ReasoningContent replay fidelity, TUI exit-time keyboard reset triad, codex `end_turn` Option-typed signal w/ author FIXME

8-PR sweep across four repos (litellm × 3, sst/opencode × 1, goose × 2, codex × 2). Themes: a Fireworks chat-completions modernization that bundles three undocumented `map_openai_params` deletions (the `tool_choice="required"→"any"` workaround from #4416, the `response_format`+`tools` coexistence carve-out, and the `max_completion_tokens→max_tokens` rewrite) with a legitimately-improved `get_models` that paginates and a pair of new Anthropic-Messages and OpenAI-Responses subclasses — request-changes for the bundle-and-no-justification shape; a symmetric suffix-match branch in `SemanticMCPToolFilter._name_matches_canonical` that recognises LibreChat's `<canonical><sep><uid>` pattern alongside opencode's prefix `<alias><sep><canonical>`, with the spurious-match guard "remainder contains no further separator" exhaustively pinned by 6 behavioral + 1 contract-level test; a ~640-line Arize/Phoenix rebuild that fixes Pydantic response coercion, the IMAGE_URL-key path Phoenix actually renders against, a 32 KB inline cap with sha256 omission notice, dict-shaped Responses-API `output[]` items, parent-vs-child token double-counting, and SDK-mode guardrail-span orphaning — but ships with **zero tests** for any of it; the `cross-spawn` CJS-default-under-Bun-ESM fix using `createRequire(import.meta.url)` with `import type` sidecar; a `Config::get_param` typed-coercion fallback that routes through the existing `parse_env_value` helper so `OLLAMA_TIMEOUT: '1200'` (quoted YAML string) deserialises into `u64`, with a four-test matrix covering the bug, the no-regression-on-string-consumers, and the failure path; a Bedrock `ReasoningContent` replay-fidelity rewrite that turns the catch-all `bail!` into a typed `Result<Option<MessageContent>>` (Err for known-but-unsupported, None for SDK `non_exhaustive::Unknown`), preserves `Thinking` text+signature on outbound, base64-decodes `RedactedThinking` with warn-and-drop fallback for cross-provider conversation histories — but no tests; a TUI exit-path triad (`PopKeyboardEnhancementFlags` + `ResetKeyboardEnhancementFlags` + `DisableModifyOtherKeys`) gated behind a new `restore_after_exit()` helper, with the keyboard module split out cleanly and the `restore_common` interior refactored to a two-axis `RawModeRestore × KeyboardRestore` enum; and the `end_turn: Option<bool>` plumbing through SSE → event enum → channel → turn loop with `needs_follow_up |= !end_turn` composition (don't override existing follow-up requests) — but with the author's own literal `FIXME(andrey): Make this actually work + test it, before merging` left in `responses_cmd.rs:81`, an unambiguous "do not merge" signal.

| PR | Title | Path |
|---|---|---|
| [#26538](https://github.com/BerriAI/litellm/pull/26538) | fix(fireworks_ai): modernize chat transforms, add Messages + Responses | [2026-W17/BerriAI-litellm-pr-26538.md](2026-W17/BerriAI-litellm-pr-26538.md) |
| [#26533](https://github.com/BerriAI/litellm/pull/26533) | fix(proxy): handle client-side unique-ID suffixes in MCP semantic tool filter | [2026-W17/BerriAI-litellm-pr-26533.md](2026-W17/BerriAI-litellm-pr-26533.md) |
| [#26526](https://github.com/BerriAI/litellm/pull/26526) | fix arize observability bugs | [2026-W17/BerriAI-litellm-pr-26526.md](2026-W17/BerriAI-litellm-pr-26526.md) |
| [#24374](https://github.com/sst/opencode/pull/24374) | fix(sdk): load cross-spawn through require | [2026-W17/sst-opencode-pr-24374.md](2026-W17/sst-opencode-pr-24374.md) |
| [#8844](https://github.com/block/goose/pull/8844) | fix: convert quoted numeric config values to numbers if needed | [2026-W17/block-goose-pr-8844.md](2026-W17/block-goose-pr-8844.md) |
| [#8843](https://github.com/block/goose/pull/8843) | fix(bedrock): handle ReasoningContent blocks gracefully | [2026-W17/block-goose-pr-8843.md](2026-W17/block-goose-pr-8843.md) |
| [#19625](https://github.com/openai/codex/pull/19625) | Reset TUI keyboard reporting on exit | [2026-W17/openai-codex-pr-19625.md](2026-W17/openai-codex-pr-19625.md) |
| [#19610](https://github.com/openai/codex/pull/19610) | Support end_turn in response.completed | [2026-W17/openai-codex-pr-19610.md](2026-W17/openai-codex-pr-19610.md) |

Verdict mix: 2 merge-as-is (sst/opencode#24374, goose#8844), 3 merge-after-nits (litellm#26533, goose#8843, codex#19625), 3 request-changes (litellm#26538 — bundles three undocumented Fireworks param-mapping deletions with a legitimate refactor; litellm#26526 — ~640 lines of new observability code with zero tests; codex#19610 — author's own FIXME marker in `responses_cmd.rs:81` is an unambiguous do-not-merge).

### W17 drip-66 (2026-04-26) — Slash-command provenance on user messages, MCP tool-order determinism + concurrency-drop, codex assistant-phase round-trip parity, vendored PII guardrail w/ pre/post cache asymmetry, per-model team-budget reimpl bundling schema+global-cache+pricing changes, goose logging-helper extraction losing OTLP `force` gate

8-PR sweep across four repos (sst/opencode × 4, openai/codex × 1, BerriAI/litellm × 2, block/goose × 1). Themes: sst/opencode adds an optional `command: { name, arguments, source: "command"|"mcp"|"skill" }` to the v2 `User` schema and threads it through `PromptInput` so `/plan`-style invocations are persisted with their originating slash-command (the schema is duplicated across `message-v2.ts:391` and `prompt.ts:1730` — extract a shared `CommandRef`); a one-character typo fix in `nix/opencode.nix:67` (`dunning` → `running`); a one-row docs addition for `opencode-toon-config-plugin` in `ecosystem.mdx:55`; a deterministic-MCP-tool-order fix that sorts both intra-server (`defs():157`) and inter-server (`layer():629`) via `localeCompare`, with a single behavioral test pinning `["a-server_alpha", "a-server_beta", "z-server_zeta"]` — but the same diff silently drops `Effect.forEach(..., { concurrency: "unbounded" })` for a sequential `for...of` (a separate observable change deserving its own PR-body line); a codex protocol-symmetry fix that adds `phase: Option<MessagePhase>` to `ResponseInputItem::Message` (wire-compatible via `serde(default, skip_serializing_if = Option::is_none)` + `ts(optional)`) and threads it through 8 call-sites so assistant `Commentary` survives replay/inject/persist round-trips, with one conversion test pinning the seam (`models.rs:1598`) and `InterAgentCommunication::into_response_input_item()` opting cross-agent traffic into `Commentary` by default; a vendor-contributed peyeeye PII redaction+rehydration guardrail with stateful/stateless modes that has a real cache-handle asymmetry — pre-call writes to the per-request `DualCache` arg (`peyeeye.py:202`), post-call reads from the global `litellm.cache` (`peyeeye.py:227`), so any deployment without a configured global cache silently fails to rehydrate placeholders into the user-facing response (and 8 well-written tests miss this exact path because they monkey-patch `litellm.cache = MagicMock()`); a 2500-line "clean reimplementation" of per-model team-member budgets that correctly pivots from JSON RMW to a dedicated `LiteLLM_TeamMemberModelSpend(user_id, team_id, model)` table with composite PK supporting atomic upsert-increment, but bundles a paired no-op + real migration (bisect/rollback hazard), an `nx` kwarg added to the global `InMemoryCache.set_cache` with no return value to distinguish set-vs-skip, and an unrelated `azure/gpt-5.5` pricing JSON entry — and offers no migration path for deployments already on the predecessor #26471's JSON column shape; and a goose logging refactor that pulls `goose-cli` and `goose-server` duplication into a shared `goose::logging::build_logging_subscriber(&LoggingConfig)` helper with `LoggingConfig { component, name, extra_directives, console }` — but silently drops the CLI's previous `if !force` gate on OTLP layer initialization (so `cargo test` runs with OTLP env vars now spin up an exporter), and the original `test_default_filter_avoids_debug_by_default` literal-filter assertion is reduced to a smoke-only `is_ok()` check (the contract still exists in `build_env_filter` defaults but is no longer pinned anywhere).

| PR | Title | Path |
|---|---|---|
| [#24422](https://github.com/sst/opencode/pull/24422) | feat(session): persist invoking slash command on user messages | [2026-W17/sst-opencode-pr-24422.md](2026-W17/sst-opencode-pr-24422.md) |
| [#24420](https://github.com/sst/opencode/pull/24420) | fix: correct typo in comment | [2026-W17/sst-opencode-pr-24420.md](2026-W17/sst-opencode-pr-24420.md) |
| [#24397](https://github.com/sst/opencode/pull/24397) | docs: add toon-config-plugin to ecosystem plugins | [2026-W17/sst-opencode-pr-24397.md](2026-W17/sst-opencode-pr-24397.md) |
| [#24339](https://github.com/sst/opencode/pull/24339) | fix(mcp): stabilize tool ordering | [2026-W17/sst-opencode-pr-24339.md](2026-W17/sst-opencode-pr-24339.md) |
| [#19626](https://github.com/openai/codex/pull/19626) | Preserve assistant phase for replayed messages | [2026-W17/openai-codex-pr-19626.md](2026-W17/openai-codex-pr-19626.md) |
| [#26546](https://github.com/BerriAI/litellm/pull/26546) | feat(guardrails): add peyeeye PII redaction & rehydration guardrail | [2026-W17/BerriAI-litellm-pr-26546.md](2026-W17/BerriAI-litellm-pr-26546.md) |
| [#26523](https://github.com/BerriAI/litellm/pull/26523) | feat(teams): per-model team member budgets (clean reimplementation) | [2026-W17/BerriAI-litellm-pr-26523.md](2026-W17/BerriAI-litellm-pr-26523.md) |
| [#8817](https://github.com/block/goose/pull/8817) | refactor(logging): consolidate logging setup into shared helper in goose crate | [2026-W17/block-goose-pr-8817.md](2026-W17/block-goose-pr-8817.md) |

Verdict mix: 2 merge-as-is (sst/opencode#24420, sst/opencode#24397), 5 merge-after-nits (sst/opencode#24422, sst/opencode#24339, codex#19626, litellm#26546, goose#8817), 1 needs-discussion (litellm#26523 — bundled schema+global-cache-contract+unrelated-pricing changes plus an unaddressed migration path for deployments already on #26471).

### W17 drip-67 (2026-04-26) — TUI theme accent on statusline, ShutdownComplete double-write, actionable skills budget warning, profile-backed permissions migration, lazy-loaded proxy routers w/ silent-404 risk, DeepSeek 2nd-pass reasoning preservation, transcript-position vs lex ID compare in prompt loop, llama backend init `panic!` → `Result`

8-PR sweep across four repos (openai/codex × 4, BerriAI/litellm × 1, sst/opencode × 2, block/goose × 1). Themes: a TUI statusline restyle that derives per-segment foreground accents from the active syntax theme via TextMate scope lookups, drops blanket `.dim()` in favor of selectively dimming only chrome (separators, agent label), and adds `AppEvent::SyntaxThemePreviewed` so theme picker preview/cancel/save all refresh cached colors — visual-contrast change ships globally for every user, release-notes-worthy; a 2-line shutdown-handler fix that splits the terminal `ShutdownComplete` event into a `record_protocol_event` + `deliver_event_raw` pair so it doesn't try to append to the just-closed `LiveThread` writer (regression from #18882's `ThreadStore` routing), with a `#[cfg(debug_assertions)]` test pinning `..Default::default()` rest-pattern that asserts zero post-`shutdown_thread` store calls; a skills-budget warning rewrite that swaps aggregate-stats text ("7 skills not included") for actionable enumeration ("Omitted skills: imagegen, openai-docs, ..., and 2 more"), capped by `MAX_WARNING_SKILL_NAMES = 5`, with `budget_warning_prefix` simplified from `replacen` string-mangling to a `match` returning `&'static str` — log-grep-contract change for downstream scrapers; a 4836-line / 62-file refactor making `PermissionProfile` (Managed/Disabled/External + filesystem rules) canonical on `Permissions` and `SessionConfiguration` with `sandbox_policy` demoted to a *projection* via `compatibility_sandbox_policy_for_permission_profile(...)`, replacing `from_legacy_sandbox_policy(&SandboxPolicy::DangerFullAccess)` with the explicit `from_runtime_permissions_with_enforcement(enforcement, ...)` constructor — needs-discussion on whether there's a single canonical setter enforcing projection sync, plus regression tests for the deny-read + `glob_scan_max_depth` preservation claim and parity tests for new `read_only()`/`workspace_write()` presets; a 316-line `LazyFeatureMiddleware` for litellm proxy that lazy-imports 17 optional feature routers (MCP, agents, vector_stores, scim, cloudzero, vantage, policies, guardrails, claude-code-marketplace, etc.) on first matching path-prefix request — saves ~700MB but ships with a silent-permanent-404-on-transient-import-failure path, deletes import-time side effects (`global_mcp_tool_registry`, `global_agent_registry`) without explicit handling, and the prefix-overlap discrimination (`/policy/` vs `/policies` vs `/policies/resolve`) is a fragile order-and-trailing-slash convention with no test; a DeepSeek 2nd-pass `reasoning_content` preservation fix (`existingField = msg.providerOptions?.[sdk]?.[field]; resolvedText = reasoningText || existingField || ""`) routing through dynamic `sdkKey()` instead of hardcoded `openaiCompatible`, plus a `model.capabilities.reasoning && !interleaved` historical-message empty-injection block with duplicated array-vs-string-content branches — zero tests in the diff slice; the canonical fix for #23490 in sst/opencode prompt loop replacing `lastUser.id < lastAssistant.id` lex-compare with `lastUserIndex < lastAssistantIndex` array-position compare during the same backward walk, with a regression test using `MessageID.make("msg_zzzzzz...")` to pin caller-supplied IDs that lex-sort greater than `MessageID.ascending()` output; and a goose `local_inference.rs` panic-to-Result migration changing `InferenceRuntime::get_or_init() -> Arc<Self>` to `-> Result<Arc<Self>>` so `goose-server` startup on hardware without working llama backend produces an actionable error rather than a process panic, with the `Weak`-based caching meaning every caller retries init-on-failure (deserves a comment).

| PR | Title | Path |
|---|---|---|
| [#19631](https://github.com/openai/codex/pull/19631) | Color TUI statusline from active theme | [2026-W17/openai-codex-pr-19631.md](2026-W17/openai-codex-pr-19631.md) |
| [#19630](https://github.com/openai/codex/pull/19630) | Avoid persisting ShutdownComplete after thread shutdown | [2026-W17/openai-codex-pr-19630.md](2026-W17/openai-codex-pr-19630.md) |
| [#19624](https://github.com/openai/codex/pull/19624) | fix(skills): make budget warning actionable | [2026-W17/openai-codex-pr-19624.md](2026-W17/openai-codex-pr-19624.md) |
| [#19606](https://github.com/openai/codex/pull/19606) | permissions: make runtime config profile-backed | [2026-W17/openai-codex-pr-19606.md](2026-W17/openai-codex-pr-19606.md) |
| [#26534](https://github.com/BerriAI/litellm/pull/26534) | [WIP] Lazy-load optional feature routers on first request | [2026-W17/BerriAI-litellm-pr-26534.md](2026-W17/BerriAI-litellm-pr-26534.md) |
| [#24428](https://github.com/sst/opencode/pull/24428) | fix: preserve existing reasoning_content on second interleaved pass (#24146 follow-up) | [2026-W17/sst-opencode-pr-24428.md](2026-W17/sst-opencode-pr-24428.md) |
| [#24379](https://github.com/sst/opencode/pull/24379) | fix(session): use transcript position instead of lexical ID compare in prompt loop | [2026-W17/sst-opencode-pr-24379.md](2026-W17/sst-opencode-pr-24379.md) |
| [#8814](https://github.com/block/goose/pull/8814) | fix: return error instead of panicking on llama backend init failure | [2026-W17/block-goose-pr-8814.md](2026-W17/block-goose-pr-8814.md) |

Verdict mix: 3 merge-as-is (codex#19630, sst/opencode#24379, goose#8814), 4 merge-after-nits (codex#19631, codex#19624, sst/opencode#24428, and — wait, recount), 1 needs-discussion (codex#19606 — 4836 lines / 62 files demands four pre-merge guarantees on projection-sync setter, deny-read preservation tests, preset parity tests, and a grep-audit), 1 request-changes (litellm#26534 — silent-permanent-404 on transient import failure, deleted import-time side-effects without handling, fragile prefix-overlap convention, zero tests, marked WIP). Recount: 3 merge-as-is + 3 merge-after-nits (codex#19631, codex#19624, sst/opencode#24428) + 1 needs-discussion + 1 request-changes = 8.

### W17 drip-68 (2026-04-26) — two dependabot bumps, vendor-claimed Vertex `output_format` parity in a 5-changes-in-one staging roll-up, vestigial `end_turn` field deletion, per-message TUI token counts, unsupported-image-MIME pre-filter, login-shell `-i` flag SIGTTOU regression-fix, vendor-self-listed FuturMix declarative provider w/ likely wrong `base_url`

8-PR sweep across four repos (BerriAI/litellm × 3, openai/codex × 1, sst/opencode × 1, anomalyco/opencode × 1, block/goose × 2). Themes: two dependabot lockfile-only patch bumps targeted at the internal staging branch (postcss 8.5.6 → 8.5.10 in `ui/litellm-dashboard/`, gitpython 3.1.46 → 3.1.47 in `uv.lock`) — both single-purpose and integrity-hash-coherent with the published registry records; a 5-in-one staging roll-up whose headline is the `output_config` passthrough fix (closes #23380, supersedes four prior attempts) introducing a leaf-module `sanitize_vertex_anthropic_output_params` helper to break a CodeQL-flagged cyclic import + an `ANTHROPIC_ONLY_REQUEST_KEYS = frozenset({"output_config"})` gate to stop the adapter from leaking Anthropic-shaped keys past translation, but flipping the test assertion `test_vertex_ai_claude_sonnet_4_5_structured_output_fix` from "stripped" to "forwarded" on the strength of one negative data point with the author's own request for live-Vertex validation still open — needs-discussion until that gate clears, plus a real `extra_kwargs or {} → extra_kwargs if extra_kwargs is not None else {}` semantic fix that preserves explicit-empty-dict callers; a vestigial `ResponseItem::Message.end_turn: Option<bool>` field deletion (annotated since introduction as "Do not use directly, no available consistently across all providers") that ripples through 51 files as mechanical struct-literal cleanups, with wire compat preserved by the original `skip_serializing_if = "Option::is_none"` annotation; a 12-line additive TUI surfacing of per-message input/output token counts in two places (sidebar context plugin footer + assistant-message metadata footer) using `↓`/`↑` glyphs and a `<Show when={... > 0}>` zero-state gate, but with a minor `Locale.number(...)` vs `.toLocaleString()` formatter inconsistency between the two sites and a "cache reads counted as input + reasoning counted as output" framing worth tooltip-flagging for billing-comparison clarity; an unsupported-image-MIME pre-filter at `transform.ts:309` substituting a model-actionable text part `"ERROR: Cannot read X (unsupported image format Y). Supported image formats are JPEG, PNG, GIF, and WebP. Inform the user."` for any payload outside `{jpeg, png, gif, webp}` before provider submission, with both branches of the filename-vs-MIME ternary covered by tests but missing a positive-case regression test and inconsistent `mime.toLowerCase()` handling between the new gate and the existing `mimeToModality` startsWith check; a 14-line goose login-shell regression-fix dropping the `-i` flag from `zsh -l -i -c "echo $PATH"` after #8631 introduced an `interactive`-shell `tcsetpgrp()` race that demoted goose to background and triggered SIGTTOU on rustyline's `tcsetattr` — comment block correctly identifies the kernel-level cause (not just the symptom) but the "PATH-from-`.zprofile`-only" assumption may regress users who export PATH from `~/.zshrc`; and a vendor-self-listed FuturMix declarative provider config (5 models: Claude Sonnet 4, GPT-4o, Gemini 2.5 Pro, DeepSeek Chat, Claude Haiku 4) where `base_url: "https://futurmix.ai/v1/chat/completions"` likely double-appends the path under the OpenAI engine convention used by sibling configs, plus a docs link pointing at `aaif-goose/goose` instead of the canonical `block/goose` repo.

| PR | Title | Path |
|---|---|---|
| [#26540](https://github.com/BerriAI/litellm/pull/26540) | chore(deps-dev): bump postcss from 8.5.6 to 8.5.10 in /ui/litellm-dashboard | [2026-W17/drip-68/BerriAI-litellm-pr-26540.md](2026-W17/drip-68/BerriAI-litellm-pr-26540.md) |
| [#26539](https://github.com/BerriAI/litellm/pull/26539) | chore(deps): bump gitpython from 3.1.46 to 3.1.47 | [2026-W17/drip-68/BerriAI-litellm-pr-26539.md](2026-W17/drip-68/BerriAI-litellm-pr-26539.md) |
| [#26530](https://github.com/BerriAI/litellm/pull/26530) | chore(staging): roll oss_staging_04_25_2026 into internal staging (output_config fix + 4 upstream sync fixes) | [2026-W17/drip-68/BerriAI-litellm-pr-26530.md](2026-W17/drip-68/BerriAI-litellm-pr-26530.md) |
| [#19605](https://github.com/openai/codex/pull/19605) | Delete unused ResponseItem::Message.end_turn | [2026-W17/drip-68/openai-codex-pr-19605.md](2026-W17/drip-68/openai-codex-pr-19605.md) |
| [#24434](https://github.com/sst/opencode/pull/24434) | feat(tui): show per-message input/output token counts | [2026-W17/drip-68/sst-opencode-pr-24434.md](2026-W17/drip-68/sst-opencode-pr-24434.md) |
| [#24364](https://github.com/anomalyco/opencode/pull/24364) | fix(provider): reject unsupported image mime types | [2026-W17/drip-68/anomalyco-opencode-pr-24364.md](2026-W17/drip-68/anomalyco-opencode-pr-24364.md) |
| [#8836](https://github.com/block/goose/pull/8836) | fix: drop -i flag from login shell PATH resolution to prevent SIGTTOU | [2026-W17/drip-68/block-goose-pr-8836.md](2026-W17/drip-68/block-goose-pr-8836.md) |
| [#8840](https://github.com/block/goose/pull/8840) | Add FuturMix provider | [2026-W17/drip-68/block-goose-pr-8840.md](2026-W17/drip-68/block-goose-pr-8840.md) |

Verdict mix: 3 merge-as-is (litellm#26540, litellm#26539, codex#19605), 3 merge-after-nits (sst/opencode#24434 — formatter inconsistency + cache-reads-as-input release-note nit; anomalyco/opencode#24364 — missing positive-case test + MIME-case lowercasing inconsistency; goose#8836 — `-l`-only PATH sourcing may regress `~/.zshrc`-only users), 1 needs-discussion (litellm#26530 — Vertex `output_format` passthrough flipped on one data point with author's own live-endpoint validation gate still open), 1 request-changes (goose#8840 — `base_url` likely double-appends `/chat/completions` under the OpenAI-engine convention + docs link points to wrong fork).

### W17 drip-69 (2026-04-26)

8 fresh PRs across four repos — heavy DeepSeek-family theme (cline v4-pro,
opencode openrouter SDK bump, qwen-code sglang/vllm gate, qwen-code strict
OpenAI-compat tool-media split). Telemetry, skills lazy-load, slash-command
input handling, env-var timeout override, and worktree-wide session listing
round it out.

| PR | Title | File |
| --- | --- | --- |
| [#3630](https://github.com/QwenLM/qwen-code/pull/3630) | fix(telemetry): use safeJsonStringify in FileExporter to avoid circular reference crash | [2026-W17/drip-69/QwenLM-qwen-code-pr-3630.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3630.md) |
| [#3620](https://github.com/QwenLM/qwen-code/pull/3620) | fix(core): match DeepSeek provider by model name for sglang/vllm (#3613) | [2026-W17/drip-69/QwenLM-qwen-code-pr-3620.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3620.md) |
| [#3617](https://github.com/QwenLM/qwen-code/pull/3617) | fix(core): split tool-result media into follow-up user message for strict OpenAI compat | [2026-W17/drip-69/QwenLM-qwen-code-pr-3617.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3617.md) |
| [#3604](https://github.com/QwenLM/qwen-code/pull/3604) | feat(skills): parallelize loading + add path-conditional activation | [2026-W17/drip-69/QwenLM-qwen-code-pr-3604.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3604.md) |
| [#3618](https://github.com/QwenLM/qwen-code/pull/3618) | fix(vscode-companion): fill slash commands into input on Enter instead of auto-submitting | [2026-W17/drip-69/QwenLM-qwen-code-pr-3618.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3618.md) |
| [#3629](https://github.com/QwenLM/qwen-code/pull/3629) | feat(config): support API timeout env override | [2026-W17/drip-69/QwenLM-qwen-code-pr-3629.md](2026-W17/drip-69/QwenLM-qwen-code-pr-3629.md) |
| [#10414](https://github.com/cline/cline/pull/10414) | Update: Deepseek reasoner check to include v4-pro | [2026-W17/drip-69/cline-cline-pr-10414.md](2026-W17/drip-69/cline-cline-pr-10414.md) |
| [#24359](https://github.com/sst/opencode/pull/24359) | fix(app): show project sessions across worktrees | [2026-W17/drip-69/sst-opencode-pr-24359.md](2026-W17/drip-69/sst-opencode-pr-24359.md) |

Verdict mix: 1 merge-as-is (qwen#3630), 5 merge-after-nits (qwen#3620 substring-match comment; qwen#3604 path-source in activation reminder; qwen#3618 demo recording + `/login` changelog; qwen#3629 warn-on-bad-env; opencode#24359 consumer audit + layout call-site doc), 1 needs-discussion (qwen#3617 — flip-default vs per-provider flag), 1 request-changes (cline#10414 — missing comma syntax error + diff doesn't match described scope).

### W17 drip-70 (2026-04-26)

8 fresh PRs across six repos — billing-correctness fix in OAI-compat usage
normalization (sst/opencode), dead-subcommand removal + agent-identity
runtime auth fix (openai/codex × 2), two dependabot bumps with mixed
risk profiles (block/goose × 2 — winreg patch + rcgen minor with backend
swap), one lockfile-only patch bump (cline/cline), opt-in cost-estimation
feature with scope-creep (QwenLM/qwen-code), a new SAST workflow
(Aider-AI/aider), and a frontend spinner consolidation with a11y upgrade
(All-Hands-AI/OpenHands).

| PR | Title | File |
| --- | --- | --- |
| [#24441](https://github.com/sst/opencode/pull/24441) | fix(zen): stop double-counting reasoning_tokens in oa-compat usage | [2026-W17/drip-70/sst-opencode-pr-24441.md](2026-W17/drip-70/sst-opencode-pr-24441.md) |
| [#19640](https://github.com/openai/codex/pull/19640) | [codex] remove responses command | [2026-W17/drip-70/openai-codex-pr-19640.md](2026-W17/drip-70/openai-codex-pr-19640.md) |
| [#19635](https://github.com/openai/codex/pull/19635) | Fix agent identity runtime auth flow | [2026-W17/drip-70/openai-codex-pr-19635.md](2026-W17/drip-70/openai-codex-pr-19635.md) |
| [#8829](https://github.com/block/goose/pull/8829) | chore(deps): bump winreg from 0.55.0 to 0.56.0 | [2026-W17/drip-70/block-goose-pr-8829.md](2026-W17/drip-70/block-goose-pr-8829.md) |
| [#8827](https://github.com/block/goose/pull/8827) | chore(deps): bump rcgen from 0.13.2 to 0.14.7 | [2026-W17/drip-70/block-goose-pr-8827.md](2026-W17/drip-70/block-goose-pr-8827.md) |
| [#10389](https://github.com/cline/cline/pull/10389) | chore(deps): bump hono from 4.12.9 to 4.12.15 | [2026-W17/drip-70/cline-cline-pr-10389.md](2026-W17/drip-70/cline-cline-pr-10389.md) |
| [#3631](https://github.com/QwenLM/qwen-code/pull/3631) | Feat/stats model cost estimation | [2026-W17/drip-70/QwenLM-qwen-code-pr-3631.md](2026-W17/drip-70/QwenLM-qwen-code-pr-3631.md) |
| [#5073](https://github.com/Aider-AI/aider/pull/5073) | feat: add SAST scanning workflow with Semgrep and OSV-Scanner | [2026-W17/drip-70/Aider-AI-aider-pr-5073.md](2026-W17/drip-70/Aider-AI-aider-pr-5073.md) |
| [#14138](https://github.com/All-Hands-AI/OpenHands/pull/14138) | refactor(frontend): consolidate loading states around shared spinner | [2026-W17/drip-70/All-Hands-AI-OpenHands-pr-14138.md](2026-W17/drip-70/All-Hands-AI-OpenHands-pr-14138.md) |

Verdict mix: 2 merge-as-is (codex#19640, cline#10389), 5 merge-after-nits (sst/opencode#24441 — parity check with native OpenAI helper + no-reasoning baseline test; goose#8829 — explain `itertools` rider; codex#19635 — split into 4 commits + add env-override test; aider#5073 — Semgrep `--baseline-ref` + EOF newline + `concurrency:` group; OpenHands#14138 — visual screenshot pair + `className`/`spinnerClassName` doc), 2 needs-discussion (qwen#3631 — split out unrelated `QWEN_CODE_API_TIMEOUT_MS` resolver changes + acknowledge cache-read pricing limitation; goose#8827 — rcgen 0.13→0.14 backend swap needs human-tested CI confirmation).

### W17 drip-71 (2026-04-26)

8 fresh PRs across five repos — DeepSeek reasoning_content second-pass
preservation (sst/opencode), project-relative permission matching in
Read tool (sst/opencode), two job-control / robustness fixes in
goose-server (login-shell PATH probe SIGTTIN suspension, CSP
`.unwrap()` panic → 400), two security/correctness fixes in OpenHands
(MCP config secret redaction, Gemini full-URL doubled-path 404),
shell line-continuation handling in qwen-code splitter, and an Ollama
unreachable abort path for aider --exit batch mode.

| PR | Title | File |
| --- | --- | --- |
| [#24443](https://github.com/sst/opencode/pull/24443) | fix(provider): preserve reasoning_content on second interleaved pass (#24146 follow-up) | [2026-W17/drip-71/sst-opencode-pr-24443.md](2026-W17/drip-71/sst-opencode-pr-24443.md) |
| [#24320](https://github.com/sst/opencode/pull/24320) | fix(read): match project-relative permissions | [2026-W17/drip-71/sst-opencode-pr-24320.md](2026-W17/drip-71/sst-opencode-pr-24320.md) |
| [#8804](https://github.com/block/goose/pull/8804) | fix(shell): prevent login-shell PATH probe from suspending goose on startup | [2026-W17/drip-71/block-goose-pr-8804.md](2026-W17/drip-71/block-goose-pr-8804.md) |
| [#8810](https://github.com/block/goose/pull/8810) | fix: return 400 instead of panicking on invalid CSP header value | [2026-W17/drip-71/block-goose-pr-8810.md](2026-W17/drip-71/block-goose-pr-8810.md) |
| [#14084](https://github.com/All-Hands-AI/OpenHands/pull/14084) | fix(security): redact API keys from MCP config logging | [2026-W17/drip-71/All-Hands-AI-OpenHands-pr-14084.md](2026-W17/drip-71/All-Hands-AI-OpenHands-pr-14084.md) |
| [#14029](https://github.com/All-Hands-AI/OpenHands/pull/14029) | fix(llm): skip Gemini full endpoint URL as base_url to prevent doubled path 404 | [2026-W17/drip-71/All-Hands-AI-OpenHands-pr-14029.md](2026-W17/drip-71/All-Hands-AI-OpenHands-pr-14029.md) |
| [#3600](https://github.com/QwenLM/qwen-code/pull/3600) | fix(core): handle shell line continuations in command splitting | [2026-W17/drip-71/QwenLM-qwen-code-pr-3600.md](2026-W17/drip-71/QwenLM-qwen-code-pr-3600.md) |
| [#4977](https://github.com/Aider-AI/aider/pull/4977) | fix: abort cleanly in batch mode (--exit) when Ollama is unreachable | [2026-W17/drip-71/Aider-AI-aider-pr-4977.md](2026-W17/drip-71/Aider-AI-aider-pr-4977.md) |

Verdict mix: 3 merge-as-is (sst/opencode#24320 project-relative permission patterns; OpenHands#14029 Gemini `:generateContent` URL marker detection; goose#8804 `process-wrap`/`ProcessSession` setsid fix for SIGTTIN suspension), 5 merge-after-nits (sst/opencode#24443 — hoist `sdkKey` + scope catch-all to deepseek + replay regression test; goose#8810 — log discarded `InvalidHeaderValue` + config-load validation + 400 regression test; OpenHands#14084 — upgrade `remote_runtime` regex redactor to schema-aware `sanitize_config` + secret-leak regression test per call site; qwen#3600 — double-quote continuation test + `consumeLineContinuation` helper; aider#4977 — replace `str(ex).startswith("OllamaError:")` with `isinstance` + audit `.get(...)` callers of `get_model_info`).

### W17 drip-72 (2026-04-26)

8 more across 5 repos — second-pass reasoning preservation in normalize transforms (Kilo `reasoning_details` array vs DeepSeek `reasoning_content` string), startup-stdin side-buffer drain into TUI prompt, ASCII-escape JSON for HTTP turn-metadata headers, `Notify`-based race-free test wait, `tool_choice="required"` semantics + always-pass `reasoning_effort` in fireworks chat-transform refresh, `temp_budget_increase` cache-hit overlay (with PR-scope concern over a piggybacked model-catalog dump), upper bound on `retry-after` to prevent multi-hour silent hangs, and a lifecycle-hooks system whose security model needs more shape before merge.

| PR | Title | File |
| --- | --- | --- |
| [#24412](https://github.com/sst/opencode/pull/24412) | fix: Buffer stdin before prompt UI appears | [sst-opencode/PR-24412-buffer-stdin-before-prompt.md](sst-opencode/PR-24412-buffer-stdin-before-prompt.md) |
| [#24411](https://github.com/sst/opencode/pull/24411) | fix(opencode): avoid invalid Kilo reasoning details | [sst-opencode/PR-24411-avoid-invalid-kilo-reasoning-details.md](sst-opencode/PR-24411-avoid-invalid-kilo-reasoning-details.md) |
| [#19620](https://github.com/openai/codex/pull/19620) | Escape turn metadata headers as ASCII JSON | [openai-codex/PR-19620-escape-turn-metadata-ascii-json.md](openai-codex/PR-19620-escape-turn-metadata-ascii-json.md) |
| [#19589](https://github.com/openai/codex/pull/19589) | Fix request_permissions tool flake in core tests | [openai-codex/PR-19589-fix-request-permissions-tool-flake.md](openai-codex/PR-19589-fix-request-permissions-tool-flake.md) |
| [#26538](https://github.com/BerriAI/litellm/pull/26538) | fix(fireworks_ai): modernize chat transforms, add Messages + Responses configs | [BerriAI-litellm/PR-26538-fireworks-modernize-chat-add-messages-responses.md](BerriAI-litellm/PR-26538-fireworks-modernize-chat-add-messages-responses.md) |
| [#26498](https://github.com/BerriAI/litellm/pull/26498) | fix(auth): apply temp_budget_increase on cache-hit path | [BerriAI-litellm/PR-26498-temp-budget-increase-cache-hit.md](BerriAI-litellm/PR-26498-temp-budget-increase-cache-hit.md) |
| [#8842](https://github.com/block/goose/pull/8842) | feat: lifecycle hooks system | [block-goose/PR-8842-lifecycle-hooks-system.md](block-goose/PR-8842-lifecycle-hooks-system.md) |
| [#10384](https://github.com/cline/cline/pull/10384) | fix: cap retry-after delay to prevent silent multi-hour hangs | [cline-cline/PR-10384-cap-retry-after-delay.md](cline-cline/PR-10384-cap-retry-after-delay.md) |

Verdict mix: 4 merge-as-is (sst#24411 structured-vs-string `reasoning_details` split + drop-on-empty merge; codex#19620 `AsciiJsonFormatter` + ASCII-bytes/UTF-8-semantics dual assertion; codex#19589 register-notify-before-check + drop-lock-before-notify; cline#10384 throw-original-error on overlong retry-after with a 60s default ceiling), 2 merge-after-nits (sst#24412 startup-stdin buffer with explicit-prompt-precedence + drain-once + Ctrl+C verification still needed; litellm#26538 fireworks refresh — confirm `tool_choice="required"` passthrough + always-pass `reasoning_effort`), 1 request-changes (litellm#26498 — auth fix is right shape but split the 620-line model-catalog dump into its own PR), 1 needs-discussion (goose#8842 lifecycle hooks — trust model for hook config, `tool_result` actually flowing into AfterToolCall, execution timeout, error-code taxonomy).

### W17 drip-73 (2026-04-26)

8 fresh PRs across 7 repos — session-archive confirmation dialog buried under 6k lines of personal-tooling cruft (sst/opencode), AgentIdentity JWT verification with JWKS rotation (codex), a 0.x→2.x rubato dependency jump (goose), per-provider request-concurrency cap with FIFO semaphore + env fallback (qwen-code), interactive API-Key option in `qwen auth` (qwen-code), uuid 11→14 ESM-only major bump (cline), V0-era validator/MCP-route/utils removal (OpenHands), and bash repomap support (aider).

| PR | Title | File |
| --- | --- | --- |
| [#24354](https://github.com/sst/opencode/pull/24354) | feat(app): add confirmation dialog before archiving session | [2026-W17/drip-73/sst-opencode-pr-24354.md](2026-W17/drip-73/sst-opencode-pr-24354.md) |
| [#19650](https://github.com/openai/codex/pull/19650) | feat: verify agent identity JWTs | [2026-W17/drip-73/openai-codex-pr-19650.md](2026-W17/drip-73/openai-codex-pr-19650.md) |
| [#8826](https://github.com/block/goose/pull/8826) | chore(deps): bump rubato from 0.16.2 to 2.0.0 | [2026-W17/drip-73/block-goose-pr-8826.md](2026-W17/drip-73/block-goose-pr-8826.md) |
| [#3636](https://github.com/QwenLM/qwen-code/pull/3636) | feat(generationConfig): cap concurrent in-flight requests per provider (#3409) | [2026-W17/drip-73/QwenLM-qwen-code-pr-3636.md](2026-W17/drip-73/QwenLM-qwen-code-pr-3636.md) |
| [#3624](https://github.com/QwenLM/qwen-code/pull/3624) | fix(cli): add API Key option to `qwen auth` interactive menu | [2026-W17/drip-73/QwenLM-qwen-code-pr-3624.md](2026-W17/drip-73/QwenLM-qwen-code-pr-3624.md) |
| [#10367](https://github.com/cline/cline/pull/10367) | chore(deps): bump uuid from 11.1.0 to 14.0.0 | [2026-W17/drip-73/cline-cline-pr-10367.md](2026-W17/drip-73/cline-cline-pr-10367.md) |
| [#14135](https://github.com/All-Hands-AI/OpenHands/pull/14135) | V0 Code Removals: Conversation Validator, MCP Updates, and Cleanup | [2026-W17/drip-73/All-Hands-AI-OpenHands-pr-14135.md](2026-W17/drip-73/All-Hands-AI-OpenHands-pr-14135.md) |
| [#5052](https://github.com/Aider-AI/aider/pull/5052) | Add bash/shell repomap support | [2026-W17/drip-73/Aider-AI-aider-pr-5052.md](2026-W17/drip-73/Aider-AI-aider-pr-5052.md) |

Verdict mix: 1 merge-as-is (aider#5052 — strictly additive bash tree-sitter tags following established language-extension pattern), 4 merge-after-nits (codex#19650 — JWKS URL hardening, caching policy, `nbf`/max-lifetime/`typ` claim checks, typed init error; qwen#3636 — streaming-cancellation slot release test, FIFO ordering test, float env-var case, override-env-to-unlimited docs; qwen#3624 — masked password input, `0600` perms, `OPENAI_BASE_URL`-aware probe, accept-with-warning on transient failures; OpenHands#14135 — cross-repo grep, MCP route auth/error-shape diff, human end-to-end test, split into 2-3 commits, deprecation cycle for validator classes), 2 needs-discussion (goose#8826 — `cargo tree -i rubato` to confirm direct vs. transitive consumer + duplicated-graph check before merging an 0.x→2.x bump; cline#10367 — VSIX runtime load test for `ERR_REQUIRE_ESM`, `uuid` CLI binary callers, and whether the dep is still earning its keep vs. `crypto.randomUUID()`), 1 request-changes (sst/opencode#24354 — drop 6k lines of `.atl/`/`.claude/memory/`/`.playwright-mcp/`/`.sisyphus/`/`.tmp-*` author-machine cruft, split out the unrelated `pty.created` + auto-open-terminal drive-bys, deduplicate the two `DialogArchiveSession` copies into one shared component, verify i18n key coverage, add a smoke test).

### W17 drip-74 (2026-04-26)

8 fresh PRs across 6 repos — openrouter SDK 2.5.1→2.8.1 bump with prompt-cache carve-out preserved (sst/opencode), drop GNU musl release artifacts in favor of glibc-only Linux builds (codex), peyeeye-based PII guardrail with a cache-API mismatch between pre-call writes and post-call reads (litellm), preserve `reasoning_content` from buffered streaming chunks during tool-call merge (qwen-code), `--insecure` flag + `QWEN_TLS_INSECURE`/`NODE_TLS_REJECT_UNAUTHORIZED=0` for self-signed certs threaded through every undici dispatcher (qwen-code), 27-page SDK docs tree + `cline-cli/`→`cli/` rename without redirects (cline), CriticResult render component with star rating + categorized features but missing screen-reader aria-label (OpenHands), and a 2-line dead-code removal of an unused class-level `tree_cache = dict()` (aider).

| PR | Title | File |
| --- | --- | --- |
| [#24435](https://github.com/sst/opencode/pull/24435) | feat: bump @openrouter/ai-sdk-provider to 2.8.1 | [2026-W17/drip-74/sst-opencode-pr-24435.md](2026-W17/drip-74/sst-opencode-pr-24435.md) |
| [#19445](https://github.com/openai/codex/pull/19445) | chore(release): drop GNU release artifacts | [2026-W17/drip-74/openai-codex-pr-19445.md](2026-W17/drip-74/openai-codex-pr-19445.md) |
| [#26544](https://github.com/BerriAI/litellm/pull/26544) | feat: peyeeye PII guardrail | [2026-W17/drip-74/BerriAI-litellm-pr-26544.md](2026-W17/drip-74/BerriAI-litellm-pr-26544.md) |
| [#3637](https://github.com/QwenLM/qwen-code/pull/3637) | fix(openaiContentGenerator): preserve reasoning_content during tool-call merge | [2026-W17/drip-74/QwenLM-qwen-code-pr-3637.md](2026-W17/drip-74/QwenLM-qwen-code-pr-3637.md) |
| [#3635](https://github.com/QwenLM/qwen-code/pull/3635) | feat(core): --insecure flag and QWEN_TLS_INSECURE for self-signed TLS | [2026-W17/drip-74/QwenLM-qwen-code-pr-3635.md](2026-W17/drip-74/QwenLM-qwen-code-pr-3635.md) |
| [#10350](https://github.com/cline/cline/pull/10350) | docs: add SDK documentation | [2026-W17/drip-74/cline-cline-pr-10350.md](2026-W17/drip-74/cline-cline-pr-10350.md) |
| [#14133](https://github.com/All-Hands-AI/OpenHands/pull/14133) | feat(frontend): Add critic result types, component, and event rendering | [2026-W17/drip-74/All-Hands-AI-OpenHands-pr-14133.md](2026-W17/drip-74/All-Hands-AI-OpenHands-pr-14133.md) |
| [#5026](https://github.com/Aider-AI/aider/pull/5026) | refactor: remove unused class-level tree_cache variable | [2026-W17/drip-74/Aider-AI-aider-pr-5026.md](2026-W17/drip-74/Aider-AI-aider-pr-5026.md) |

Verdict mix: 3 merge-as-is (sst/opencode#24435 — minor SDK bump with the prompt-cache carve-out at `transform.ts:193-198` preserved verbatim; codex#19445 — release-artifact cleanup that removes the never-used GNU musl matrix entry; aider#5026 — 2-line dead class-level `tree_cache = dict()` removal that prevents a future class-vs-instance attribute footgun), 4 merge-after-nits (qwen-code#3637 — extend the merge-preservation contract to other streaming fields like `audio`/`tool_call_id` rather than just `reasoning_content`, add a regression test for "first chunk has empty content but later chunks fill it"; qwen-code#3635 — startup `verbose_proxy_logger.warning` naming which trigger activated insecure mode, comment on the `NODE_TLS_REJECT_UNAUTHORIZED === "0"` strict-equality contract, follow-up `QWEN_TLS_INSECURE_HOSTS` per-host allowlist; cline#10350 — add `redirects` block to `docs.json` for the 13 renamed `/cline-cli/...`→`/cli/...` pages and the deleted `/cline-sdk/overview`, confirm Mintlify preview link-checker passes since `mintlify dev` failed locally, add an "active development" banner to the SDK overview page; OpenHands#14133 — `aria-label`+`role="img"` on the star-rating span so screen readers announce "rating: N out of 5", comment on the four named feature buckets noting new SDK categories will land in `other` until added, call-site test asserting `CriticResultDisplay` is not mounted when `event.critic_result == null`), 1 request-changes (BerriAI/litellm#26544 — pre-call writes original text to `cache: DualCache` parameter but post-call reads from `litellm.cache` global; the two are different cache surfaces and the round-trip silently breaks under any deployment that doesn't auto-mirror them, plus needs error-payload PII scrubbing and a fail-open vs. fail-closed policy decision before this guards production traffic).

---

### W17 drip-75 (2026-04-26)

8 fresh PRs across 7 repos — Watchman cookie ignore + latent `.git/` watcher leak fix in one 4-file diff (sst/opencode), `pctx_code_mode` 0.3→0.4 feature-gated dependabot bump (block/goose), `getRenderableGradientColors` utility that walks a candidate chain so `NO_COLOR` and 1-stop custom themes degrade to plain text instead of crashing ink-gradient with `Invalid number of stops` (qwen-code), grouped npm bump for cline's developer-tooling dirs that picks up the lodash 4.18.1 prototype-pollution + `_.template` code-injection CVE patch chain, FuturMix gateway docs page with a stale-prone hard-coded model list (aider), Bedrock cross-region inference ID acceptance via `isBedrockAnthropicModel` + idempotent `(1M)` context-window annotator (charmbracelet/crush), grouped `security-all` poetry bump anchored on the authlib 1.6.11 Starlette OAuth CSRF fix (OpenHands), and SHA-pinning of `actions/checkout@v3` + `actions/setup-python@v4` with preserved `# vX` comments for dependabot readability (aider).

| PR | Title | File |
| --- | --- | --- |
| [#24448](https://github.com/sst/opencode/pull/24448) | fix(app): ignore watchman cookie files | [2026-W17/drip-75/sst-opencode-pr-24448.md](2026-W17/drip-75/sst-opencode-pr-24448.md) |
| [#8828](https://github.com/block/goose/pull/8828) | chore(deps): bump pctx_code_mode from 0.3.0 to 0.4.0 | [2026-W17/drip-75/block-goose-pr-8828.md](2026-W17/drip-75/block-goose-pr-8828.md) |
| [#3640](https://github.com/QwenLM/qwen-code/pull/3640) | fix(cli): guard gradient rendering without colors | [2026-W17/drip-75/QwenLM-qwen-code-pr-3640.md](2026-W17/drip-75/QwenLM-qwen-code-pr-3640.md) |
| [#10366](https://github.com/cline/cline/pull/10366) | chore(deps): bump the npm_and_yarn group across 4 directories with 11 updates | [2026-W17/drip-75/cline-cline-pr-10366.md](2026-W17/drip-75/cline-cline-pr-10366.md) |
| [#5070](https://github.com/Aider-AI/aider/pull/5070) | docs: add FuturMix LLM provider documentation | [2026-W17/drip-75/Aider-AI-aider-pr-5070.md](2026-W17/drip-75/Aider-AI-aider-pr-5070.md) |
| [#2568](https://github.com/charmbracelet/crush/pull/2568) | feat: support Bedrock cross-region inference model IDs and annotate 1M models | [2026-W17/drip-75/charmbracelet-crush-pr-2568.md](2026-W17/drip-75/charmbracelet-crush-pr-2568.md) |
| [#14131](https://github.com/All-Hands-AI/OpenHands/pull/14131) | chore(deps): bump the security-all group across 1 directory with 7 updates | [2026-W17/drip-75/All-Hands-AI-OpenHands-pr-14131.md](2026-W17/drip-75/All-Hands-AI-OpenHands-pr-14131.md) |
| [#5021](https://github.com/Aider-AI/aider/pull/5021) | ci: pin issue workflow actions | [2026-W17/drip-75/Aider-AI-aider-pr-5021.md](2026-W17/drip-75/Aider-AI-aider-pr-5021.md) |

Verdict mix: 5 merge-as-is (sst/opencode#24448 — pattern at `ignore.ts:43` plus `subscribe(vcsDir, [...ignore, ...FileIgnore.PATTERNS, ...cfgIgnores])` at `watcher.ts:142` closes a latent `.git/` watcher leak; qwen-code#3640 — `getRenderableGradientColors` variadic-candidate utility with `length >= 2 && every-non-empty-trimmed-string` validation, NO_COLOR env save/restore in tests, license header on the new `gradientUtils.ts`; cline#10366 — lockfile-only grouped bump on 4 dev-tooling dirs picking up lodash 4.18.1 (CVE-2026-4800 + GHSA-f23m-r3pf-42rh prototype pollution); OpenHands#14131 — `poetry.lock`-only grouped security bump anchored on authlib 1.6.11 Starlette OAuth CSRF fix, with 6 housekeeping bumps; aider#5021 — `actions/checkout@f43a0e5...` + `actions/setup-python@7f4fc3e...` SHA pins with `# v3`/`# v4` trailing comments for dependabot diff legibility), 3 merge-after-nits (block/goose#8828 — confirm CI exercises the `pctx_code_mode`-gated optional feature with `cargo check --features <feature>` since 0.x→0.x+1 is a SemVer break under Cargo's caret rules and a default-feature CI matrix never compiles the new version; aider#5070 — trim the hard-coded model list at lines 39-58 to family names + "see vendor docs" link since enumerated lists go stale every 3-4 months, add a one-sentence caveat about `openai/<non-openai-model>` token-count/context-window fallback because the example uses Claude through this prefix, optionally trim the marketing-flavor "Benefits" block to match the neutral tone of `openrouter.md`/`groq.md`; charmbracelet/crush#2568 — consider tightening `isBedrockAnthropicModel` from `Contains(id, ".anthropic.")` to an allowlist regex `^(us|eu|apac|ap)\.anthropic\.` if maintainer prefers strict matching, optionally add `:1m` ID-suffix detection as a secondary signal for the `(1M)` annotator since `ContextWindow == 0` would silently skip annotation today).

---

### W17 drip-76 (2026-04-26)

8 fresh PRs across 7 repos — `opencode-byterover` ecosystem-table row addition (sst/opencode), first cloud-executor milestone for `codex exec-server` with bounded auth retry + exponential reconnect + sha256-derived idempotency-id (codex), symmetric `<canonical><sep><unique_id>` suffix matching in MCP semantic tool filter to fix LibreChat-style 400-with-empty-tools regression (litellm), `BackgroundShellRegistry` + `/bashes` slash command rewrite that retires the legacy `&` fork-and-detach path (qwen-code), `maxRetryAfter`-60s cap on `withRetry` decorator with a hidden `Date.now`-stub-without-restore landmine in the test rework (cline), drop `-i` from `<shell> -l -i -c "echo $PATH"` to fix tcsetpgrp/SIGTTOU CLI-suspends-on-launch regression in both flatpak and normal paths (block/goose), narrow regex guard `^think:\s*\{.*\}$` so SDK default JSON-dump summaries fall through to the existing `ACTION_MESSAGE$THINK` i18n label (OpenHands), and workspace-boundary validator for LSP `workspace/applyEdit` that rejects out-of-root text-edit/create/delete/rename with explicit Windows `\C:\…` URI normalization (charmbracelet/crush).

| PR | Title | File |
| --- | --- | --- |
| [#24465](https://github.com/sst/opencode/pull/24465) | docs: add opencode-byterover to ecosystem | [2026-W17/drip-76/sst-opencode-pr-24465.md](2026-W17/drip-76/sst-opencode-pr-24465.md) |
| [#19575](https://github.com/openai/codex/pull/19575) | Add cloud executor registration to exec-server | [2026-W17/drip-76/openai-codex-pr-19575.md](2026-W17/drip-76/openai-codex-pr-19575.md) |
| [#26533](https://github.com/BerriAI/litellm/pull/26533) | fix(proxy): handle client-side unique-ID suffixes in MCP semantic tool filter | [2026-W17/drip-76/BerriAI-litellm-pr-26533.md](2026-W17/drip-76/BerriAI-litellm-pr-26533.md) |
| [#3642](https://github.com/QwenLM/qwen-code/pull/3642) | feat(core): managed background shell pool with /bashes command | [2026-W17/drip-76/QwenLM-qwen-code-pr-3642.md](2026-W17/drip-76/QwenLM-qwen-code-pr-3642.md) |
| [#10384](https://github.com/cline/cline/pull/10384) | fix: cap retry-after delay to prevent silent multi-hour hangs | [2026-W17/drip-76/cline-cline-pr-10384.md](2026-W17/drip-76/cline-cline-pr-10384.md) |
| [#8836](https://github.com/block/goose/pull/8836) | fix: drop -i flag from login shell PATH resolution to prevent SIGTTOU | [2026-W17/drip-76/block-goose-pr-8836.md](2026-W17/drip-76/block-goose-pr-8836.md) |
| [#14104](https://github.com/All-Hands-AI/OpenHands/pull/14104) | fix(frontend): restore think title fallback | [2026-W17/drip-76/All-Hands-AI-OpenHands-pr-14104.md](2026-W17/drip-76/All-Hands-AI-OpenHands-pr-14104.md) |
| [#2699](https://github.com/charmbracelet/crush/pull/2699) | fix(lsp): enforce workspace boundary for workspace edits | [2026-W17/drip-76/charmbracelet-crush-pr-2699.md](2026-W17/drip-76/charmbracelet-crush-pr-2699.md) |

Verdict mix: 3 merge-as-is (sst/opencode#24465 — single alphabetically-correct row at `ecosystem.mdx:20` with prettier-clean column padding; block/goose#8836 — drops `-i` from both `shell.rs:174,187` flatpak/normal sites and `subprocess.rs:60` MCP path with multi-line comments explaining the tcsetpgrp/SIGTTOU root cause; OpenHands#14104 — narrowly-scoped `^think:\s*\{.*\}$/i` regex at `get-event-content.tsx:46-50` with TODO referencing #13690 and symmetric tests for matched-fallback vs. non-matched-preserved branches), 4 merge-after-nits (codex#19575 — doc the per-process `registration_id` lifetime at `cloud.rs:333`, add ±25% jitter to the 1s→30s reconnect backoff at `cloud.rs:344-346`, confirm `recover_unauthorized` preserves original 401 status context, doc-comment the bearer-token redaction guarantee on `CloudEnvironmentRequest`; litellm#26533 — fix order-dependence assumption in `test_prefix_and_suffix_both_match_same_canonical`, add `verbose_proxy_logger.debug` at the `startswith(canonical)+rejected-remainder` site so future "tools: [] with no cause" reports are self-diagnosing, document the unique-ID alphanumeric-only character-set assumption; qwen-code#3642 — re-slot `bashesCommand` to a `b`-prefixed alphabetical position in `BuiltinCommandLoader.ts:95` (currently lands between `agentsCommand` and `arenaCommand`), Windows manual smoke on `executeBackground` since the legacy Windows-specific paths are deleted, spot-verify the cancel-then-late-complete race in registry tests; charmbracelet/crush#2699 — confirm `fsext.HasPrefix` is segment-aware not `strings.HasPrefix`, file follow-up issues for symlink resolution via `filepath.EvalSymlinks` and two-phase application semantics so partial application can't happen on multi-edit `WorkspaceEdit`, add Windows-specific subtest for `normalizeURIPath` `\C:\…`/`/C:/…` inputs), 1 request-changes (cline#10384 — `Date.now` stub at `retry.test.ts:115-117` is never restored which will leak `fakeNow=2023-11-14` into downstream tests in the same file, default `maxRetryAfter: 60_000` silently breaks every existing `@withRetry` site that was relying on long-wait behavior without a blast-radius audit; either set default to `Infinity` and require opt-in, or grep `@withRetry` and add per-site overrides where long waits are intentional).

---

### W17 drip-77 (2026-04-26)

8 fresh PRs across 8 repos — three-feature composer rework bundling queued-message editing + cancellation + new `"wrap"` followup mode plus two LLM-shaped top-level architecture docs (sst/opencode), kitty keyboard-protocol `restore_after_exit()` reset replacing fragile `PopKeyboardEnhancementFlags` so post-crash terminals don't strand Vim/Esc bindings — bundled with a clean `tui/keyboard_modes.rs` module extraction (codex), Fireworks chat-transform docstring archaeology + new `FireworksAIMessagesConfig` (Anthropic-shape) + `FireworksAIResponsesConfig` (OpenAI-Like-shape) with `reasoning_effort` gating now delegated upstream (litellm), per-provider `RateLimitedContentGenerator` + `ConcurrencyLimiter` with `requestConcurrency` setting and `QWEN_REQUEST_CONCURRENCY` env-var fallback closing the `429 Too many concurrent requests` issue (qwen-code), 9-line `supportsImages` gate on paste + drag-drop ingestion sites in ChatTextArea (cline), Bedrock `to_bedrock_message_content` now preserves `Thinking`/`RedactedThinking` blocks (with conditional `signature` builder + base64 round-trip) so reasoning models accept multi-turn replays — bundled with `Result<Option<MessageContent>>` signature change and explicit `Audio|Citations|Document|Guard|Image|SearchResult|Video → bail!` enumeration replacing the catch-all (block/goose), edit-existing-ack-comment-instead-of-posting-new pattern with `<!-- openhands-ack:{uuid} -->` HTML markers + `iter_recent_paginated_items` newest-first scan and "post new on marker-not-found" fallback (OpenHands), and `filepath.EvalSymlinks(path)` dedup-by-resolved-target while preserving original `path` for `Parse()` so `~/.config/crush/skills` symlinked to a project's `.claude/skills` no longer double-loads (charmbracelet/crush).

| PR | Title | File |
| --- | --- | --- |
| [#24471](https://github.com/sst/opencode/pull/24471) | feat: Add queued message editing, cancellation, and wrap-up behavior | [2026-W17/drip-77/sst-opencode-pr-24471.md](2026-W17/drip-77/sst-opencode-pr-24471.md) |
| [#19625](https://github.com/openai/codex/pull/19625) | Reset TUI keyboard reporting on exit | [2026-W17/drip-77/openai-codex-pr-19625.md](2026-W17/drip-77/openai-codex-pr-19625.md) |
| [#26538](https://github.com/BerriAI/litellm/pull/26538) | fix(fireworks_ai): modernize chat transforms, add Messages + Responses configs | [2026-W17/drip-77/BerriAI-litellm-pr-26538.md](2026-W17/drip-77/BerriAI-litellm-pr-26538.md) |
| [#3636](https://github.com/QwenLM/qwen-code/pull/3636) | feat(core): cap concurrent in-flight requests per provider | [2026-W17/drip-77/QwenLM-qwen-code-pr-3636.md](2026-W17/drip-77/QwenLM-qwen-code-pr-3636.md) |
| [#10396](https://github.com/cline/cline/pull/10396) | fix: respect image support toggle for paste and drag-drop operations | [2026-W17/drip-77/cline-cline-pr-10396.md](2026-W17/drip-77/cline-cline-pr-10396.md) |
| [#8843](https://github.com/block/goose/pull/8843) | fix(bedrock): handle ReasoningContent blocks gracefully | [2026-W17/drip-77/block-goose-pr-8843.md](2026-W17/drip-77/block-goose-pr-8843.md) |
| [#14127](https://github.com/All-Hands-AI/OpenHands/pull/14127) | Reduce GitHub resolver comment noise by editing acknowledgement comment | [2026-W17/drip-77/All-Hands-AI-OpenHands-pr-14127.md](2026-W17/drip-77/All-Hands-AI-OpenHands-pr-14127.md) |
| [#2694](https://github.com/charmbracelet/crush/pull/2694) | fix(skills): deduplicate skills discovered via symlinked directories | [2026-W17/drip-77/charmbracelet-crush-pr-2694.md](2026-W17/drip-77/charmbracelet-crush-pr-2694.md) |

Verdict mix: 2 merge-as-is (cline#10396 — symmetric `supportsImages` gate at paste site `ChatTextArea.tsx:859` and drag-drop site `:1268`, `useMemo` deps `[apiConfiguration, mode]` correctly cover model-switch invalidation, conservative `?? false` default; charmbracelet/crush#2694 — `EvalSymlinks(path)` resolved-target dedup at `skills.go:213-228` with `Parse(path)` still using the un-resolved discovery path so user-visible "where this came from" stays correct, fall-through to `path` on `EvalSymlinks` error preserves pre-fix behavior, focused symlink-aliasing regression test at `skills_test.go:285-326`), 5 merge-after-nits (codex#19625 — confirm three deleted unit tests `parse_bool_env`/`keyboard_enhancement_auto_disables_for_vscode_in_wsl`/`keyboard_enhancement_auto_disable_requires_wsl_and_vscode` are recreated verbatim in `keyboard_modes.rs`, doc-comment the exact CSI sequence emitted by `restore_after_exit` and link the kitty protocol spec, set `self.active = false` *before* the `restore_after_exit()` call in `TerminalRestoreGuard::restore` so a re-entry can't re-call, optional `script(1)`-driven smoke test asserting the reset CSI is emitted on clean exit; litellm#26538 — split into three commits chat-modernization/Messages/Responses for safer partial revert, regression test for `reasoning_effort` on a non-reasoning Fireworks model since gating is now delegated upstream and `supports_reasoning` import was removed, confirm Messages and Responses configs preserve model-name `accounts/fireworks/models/<model>` prefixing + auth header shape + `cache_control` stripping, spot-check `utils.py:1134`/`:1339` dispatch arms for tight gating; qwen-code#3636 — confirm `try/finally` around the wrapped call so `release()` always fires on error (token-leak is the canonical concurrency-wrapper bug), confirm streaming generators wrap the *full* stream lifetime not just the send-request call, abort/cancellation test confirming a waiter dropping out doesn't deadlock the FIFO queue, verify FIFO is enforced by an explicit waiter queue not Promise-resolution order or downgrade the docs claim, reject `requestConcurrency: 0` literal and use `undefined` for the unlimited sentinel since `0=unlimited` is a magic-value antipattern; block/goose#8843 — add `Thinking` round-trip test + `Audio` hard-error test, confirm `bedrock_content_block_kind(other)` produces a meaningful name for SDK-future `non_exhaustive` variants, doc-comment `from_bedrock_content_block` with the precise meaning of `Ok(None)` vs `Err`, CHANGELOG entry that `Audio|CitationsContent|Document|GuardContent|Image|SearchResult|Video` now hard-errors instead of silently dropping, confirm `BASE64_STANDARD` (padded) matches Bedrock's wire format for `RedactedContent`; OpenHands#14127 — verify PyGithub's `get_issue_comments()`/`get_review_comments()` paginated lists are oldest-first since the iteration assumption depends on it (newest-first would silently miss recent comments), confirm `comment.edit(body)` on a review comment preserves path/position context, follow-up to persist ack comment ID in conversation metadata at ack-time so the final-summary pass is O(1) instead of paginated scan, doc-comment the race window in `iter_recent_paginated_items`), 1 needs-discussion (sst/opencode#24471 — three independent features in one PR with `wrap` mode semantics under-specified, settings migration story missing for users with persisted `"queue"` literal, and two LLM-shaped top-level `ARCHITECTURE.md`+`STRUCTURE.md` docs dragging along with `[project-root]/` placeholders that should land in a separate docs PR with maintainer-authored content).

---

### W17 drip-78 (2026-04-26)

8 fresh PRs across 7 repos — non-blocking `command_async` session endpoint with bus-published background errors plus drive-by `NamedError` import-path swap (sst/opencode), new `orchestrate` task-decomposition subagent that fans out Task calls with a hardcoded agent table (sst/opencode), `chatgpt-responses` SSE non-stream fix that synthesizes `output[]` from accumulated `output_text.delta` chunks but ships with 31 `_experimental/out/**` static-export HTML files + a `next` minor bump bundled in (litellm), Catalan locale addition with 2143-line manual translation file (qwen-code), `deepseek-v4-pro`/`deepseek-v4-flash` model entries with a dead `isDeepseekV4` flag, default-model silent flip, and `inputPrice: 0` cost-tracking footgun (cline), 24-package `cargo-minor-and-patch` grouped dependabot bump touching `axum`/`zip`/`rayon` (block/goose), `pypdf` 6.9.2→6.10.2 enterprise lockfile bump anchored on a SEC-tagged `/Size` incremental-cloning fix (OpenHands), and a 37-line QuickSilver Pro provider docs page with an unverified `Qwen3.5-35B-A3B` model name (aider).

| PR | Title | File |
| --- | --- | --- |
| [#24459](https://github.com/sst/opencode/pull/24459) | feat(opencode): add async command endpoint | [2026-W17/drip-78/sst-opencode-pr-24459.md](2026-W17/drip-78/sst-opencode-pr-24459.md) |
| [#24450](https://github.com/sst/opencode/pull/24450) | feat(agent): add orchestrate subagent for task decomposition | [2026-W17/drip-78/sst-opencode-pr-24450.md](2026-W17/drip-78/sst-opencode-pr-24450.md) |
| [#26549](https://github.com/BerriAI/litellm/pull/26549) | Fix/chatgpt gpt5.4 nonstream output | [2026-W17/drip-78/BerriAI-litellm-pr-26549.md](2026-W17/drip-78/BerriAI-litellm-pr-26549.md) |
| [#3643](https://github.com/QwenLM/qwen-code/pull/3643) | feat: Adds Catalan language support | [2026-W17/drip-78/QwenLM-qwen-code-pr-3643.md](2026-W17/drip-78/QwenLM-qwen-code-pr-3643.md) |
| [#10418](https://github.com/cline/cline/pull/10418) | added deepseek-v4 models | [2026-W17/drip-78/cline-cline-pr-10418.md](2026-W17/drip-78/cline-cline-pr-10418.md) |
| [#8821](https://github.com/block/goose/pull/8821) | chore(deps): bump the cargo-minor-and-patch group with 24 updates | [2026-W17/drip-78/block-goose-pr-8821.md](2026-W17/drip-78/block-goose-pr-8821.md) |
| [#14130](https://github.com/All-Hands-AI/OpenHands/pull/14130) | chore(deps): bump pypdf from 6.9.2 to 6.10.2 in /enterprise | [2026-W17/drip-78/All-Hands-AI-OpenHands-pr-14130.md](2026-W17/drip-78/All-Hands-AI-OpenHands-pr-14130.md) |
| [#5042](https://github.com/Aider-AI/aider/pull/5042) | docs: add QuickSilver Pro as an LLM provider option | [2026-W17/drip-78/Aider-AI-aider-pr-5042.md](2026-W17/drip-78/Aider-AI-aider-pr-5042.md) |

Verdict mix: 1 merge-as-is (OpenHands#14130 — single-package security-flavored lockfile bump anchored on the `/Size` incremental-cloning SEC fix and FlateDecode/image-decode DoS limits, with harmless `pywin32` markers reorder + `tree_sitter_c_sharp-0.23.1` wheel housekeeping), 4 merge-after-nits (sst/opencode#24459 — split the drive-by `@opencode-ai/core/util/error`→`@opencode-ai/shared/util/error` import migration into its own chore, mirror the existing `prompt_async` test for `command_async`'s 204-fast + Bus-error-on-background-failure branches, audit `runRequest`'s capture of the Hono `Context` for use-after-handler-exit, document the `204` + delayed `Session.Event.Error` correlation contract for callers; qwen-code#3643 — recursive key-coverage diff between `ca.js` and the reference locale to catch silent fallbacks, ICU-pluralization spot-check on count-bearing strings, fix `Cataln` typo in PR body, confirm `LANG=ca_ES.UTF-8`/`ca_AD.UTF-8` auto-detection wires `ca`, drop the `package-lock.json` 1-line churn if accidental; block/goose#8821 — let CI confirm `cargo test --workspace --all-features` green before merge given the `axum`/`zip`/`rayon` reach, scan the `zip` 8.4→8.5 changelog for security-flavored fixes worth title-bumping, sanity-check the `tree_sitter_c_sharp-0.23.1` wheel hunk really lives in `Cargo.lock`, document the highest-risk package in the description so a focused revert is possible without un-bumping 23 innocent crates; aider#5042 — verify `Qwen3.5-35B-A3B` is a real provider-marketed model name vs. `Qwen3-30B-A3B`/`Qwen3-235B-A22B` confusion, add the `openai/<non-openai-model>` token-count/context-window caveat consistent with sibling pages, confirm `nav_order: 450` slots the page correctly in the sidebar, document streaming + tool-calling support state up front), 1 needs-discussion (sst/opencode#24450 — Task-tool semantics overlap with primary-agent direct fanout creates 3-deep context windows with no documented break-even, hardcoded `explore`/`general`/`build`/`plan` agent table goes stale on user-added agents, `question:allow` for a non-interactive subagent invites mid-orchestration prompts, no eval to prevent silent prompt regression, and the new agent is misplaced in `task.txt`'s explicitly-fictional examples block), 2 request-changes (litellm#26549 — the 80-line `transformation.py` SSE-output-synthesis fix is correct and well-tested, but the PR drags 31 `_experimental/out/**/index.html` static-export bundle files + a `next` 16.1.7→^16.2.4 bump along for the ride making the diff unreviewable and bisect-hostile; split into three focused PRs and tighten the bare `except` at line 215; cline#10418 — `isDeepseekV4` flag added at `deepseek.ts:84` is dead since the temperature gate at `:97` doesn't reference it despite the comment claiming "V4 models have thinking on by default", default-model silent flip from `deepseek-chat` to `deepseek-v4-pro` at `api.ts:2101` is a behavior change for every existing user without explicit pin, `inputPrice: 0` with "really it's cache" rationalization will silently zero out all DeepSeek spend in cost dashboards, and v4-pro likely needs the `isDeepseekReasoner` reasoning-conversion path it currently bypasses).

---

### W17 drip-79 (2026-04-26)

8 fresh PRs across 7 repos — `agent create` wizard frontmatter key/value rename `tools:false`→`permissions:"deny"` to match the live runtime permission shape (sst/opencode), TUI slash-command palette text/spacing restoration with surgical helper extraction (sst/opencode), non-ASCII response header support via `raw_headers` byte-passthrough that closes a class of UTF-8 proxying bugs (litellm), `qwen auth status` split of `USE_OPENAI` into Coding-Plan vs generic-OpenAI-compatible branches with four-source key resolution (qwen-code), Aikido autofix bumping `axios` 1.15.0→1.15.1 + `follow-redirects` to 1.16.0 in the `evals/` workspace to close two CVEs (cline), dependabot `lopdf` 0.36→0.40 three-minor jump that pulls in `getrandom 0.4.2`, `rand 0.10`, and `ttf-parser` while skipping the new PDF-decryption path (block/goose), 100-files-surfaced (719 deletions / 13.9k additions) "Prototype" frontend re-platform onto `@openhands/agent-server-gui` with unchecked human-test box and AI-agent-authored route rewrites (OpenHands), and an `XTVERSION` capability-probe tightening that stops sending OSC garbage to plain `xterm-256color` terminals (charmbracelet/crush).

| PR | Title | File |
| --- | --- | --- |
| [#24482](https://github.com/sst/opencode/pull/24482) | fix(opencode): agent create generates permissions field with deny instead of tools:false | [2026-W17/drip-79/sst-opencode-pr-24482.md](2026-W17/drip-79/sst-opencode-pr-24482.md) |
| [#24477](https://github.com/sst/opencode/pull/24477) | fix(tui): restore slash command palette text and spacing | [2026-W17/drip-79/sst-opencode-pr-24477.md](2026-W17/drip-79/sst-opencode-pr-24477.md) |
| [#26550](https://github.com/BerriAI/litellm/pull/26550) | feat: support non-ascii response headers via raw_headers | [2026-W17/drip-79/BerriAI-litellm-pr-26550.md](2026-W17/drip-79/BerriAI-litellm-pr-26550.md) |
| [#3623](https://github.com/QwenLM/qwen-code/pull/3623) | fix(cli): recognize OpenAI-compatible providers in `qwen auth status` | [2026-W17/drip-79/QwenLM-qwen-code-pr-3623.md](2026-W17/drip-79/QwenLM-qwen-code-pr-3623.md) |
| [#10332](https://github.com/cline/cline/pull/10332) | [Aikido] Fix 3 security issues in follow-redirects, axios | [2026-W17/drip-79/cline-cline-pr-10332.md](2026-W17/drip-79/cline-cline-pr-10332.md) |
| [#8825](https://github.com/block/goose/pull/8825) | chore(deps): bump lopdf from 0.36.0 to 0.40.0 | [2026-W17/drip-79/block-goose-pr-8825.md](2026-W17/drip-79/block-goose-pr-8825.md) |
| [#14124](https://github.com/All-Hands-AI/OpenHands/pull/14124) | Prototype import OpenHands frontend from agent-server-gui package | [2026-W17/drip-79/All-Hands-AI-OpenHands-pr-14124.md](2026-W17/drip-79/All-Hands-AI-OpenHands-pr-14124.md) |
| [#2543](https://github.com/charmbracelet/crush/pull/2543) | Fix: avoid unsupported terminal capability probe | [2026-W17/drip-79/charmbracelet-crush-pr-2543.md](2026-W17/drip-79/charmbracelet-crush-pr-2543.md) |

Verdict mix: 3 merge-as-is (sst/opencode#24482 — single-file `tools:false`→`permissions:"deny"` frontmatter rename with type narrowed `Record<string,"deny">` and preserved empty-block guard, no compatibility shim needed because the old key was already silently ignored at runtime; sst/opencode#24477 — surgical TUI palette text/spacing restoration extracted into a helper to keep both render paths in sync; litellm#26550 — `raw_headers` byte-passthrough for non-ASCII response headers with the `latin-1` ↔ `utf-8` round-trip preserved end-to-end), 4 merge-after-nits (qwen-code#3623 — extract a `printCommonStatus(modelName)` helper to dedup the four `writeStdoutLine(t('  Current Model: ...'))` calls between Coding-Plan and generic branches, add a positive test for the generic-OpenAI provider with `OPENAI_API_KEY` (current tests only assert via the Coding-Plan negative case), tag the `(Incomplete)` warning header with provider type to make support tickets greppable; cline#10332 — bot-authored Aikido autofix on `evals/` workspace with patch-level axios + minor follow-redirects bump that closes two CVEs, scope is internal-only and lockfile `"peer": true` noise is harmless on modern npm, just kick CI on the eval harness once before merge; block/goose#8825 — three-minor `lopdf` jump pulls in `getrandom 0.4.2` (off the latest 0.3.x line — confirm intentional), `rand 0.10`, and `ttf-parser 0.25.1`, scan `crates/goose-mcp/src/**` for any call into the new `*decrypt*` API since 0.39 added pdftk-derived decryption to the supply chain, consider pinning to `=0.40.0` exact until eval has cycled; charmbracelet/crush#2543 — boolean rewrite of `shouldQueryCapabilities` is strictly narrower than before with clean SSH/Kitty/TermProgram short-circuits, just expand the single test into a 3-row table covering Kitty-true, SSH-false, iTerm-true, and align the title prefix to the repo's `fix(ui):` convention), 1 needs-discussion (OpenHands#14124 — 100-files-surfaced (actual diff is 778+ files, 719 deletions, 21k-line lockfile regen, 50 route/component rewrites) "Prototype" re-platform onto `@openhands/agent-server-gui` is unmergeable in current form: PR title says prototype but body presents complete switchover, AI-agent-authored with the human-test checkbox unchecked, no documented co-versioning policy between the two repos, `git mv` not used so blame is destroyed across 700 files, `api/README.md` deleted with no replacement, and the surfaced API only returns 100 of the 778 changed files making line-level review impossible; convert to a tracking issue + stacked PRs split by layer (deps → API → components → routes → SaaS-restorations)).

---

### W17 drip-80 (2026-04-26)

8 fresh PRs across 7 repos — Windows-PowerShell `Expand-Archive` swap-out for in-process `@zip.js/zip.js` extraction with zip-slip defense (sst/opencode), three-line `roots: true` push-down to SQL that fixes a subagent-children-crowding-out-roots dialog bug closing two long-standing issues (sst/opencode), brand-new 760-line `peyeeye` PII redact→LLM→rehydrate guardrail with stateful/stateless session modes but missing timeouts and ordering guarantees (litellm), 286-line test-matrix-first re-fix of an `OPENAI_MODEL` vs `/model` precedence regression that was reverted in #3633 (qwen-code), Settings-UI ClineModelPicker price-staleness fix that stops writing `undefined` over persisted `clineModelInfo` and adds a synchronous `effectiveModelId` derivation plus `isModelInfoLoading` flag (cline), pure-consolidation `DEFAULT_PROVIDER_TIMEOUT_SECS` constant centralization across 10 provider files preserving per-provider aliases for future divergence (block/goose), Ghostty light-background detection wiring `tea.RequestBackgroundColor` through `applyBackgroundTheme` and adding a `DefaultStylesForBackground(bool)` factory (charmbracelet/crush), and a `_redact_compat` shim deletion that migrates 7 callsites to the canonical SDK redact module to close a Datadog `session_api_key`-in-WebSocket-access-logs production incident (OpenHands).

| PR | Title | File |
| --- | --- | --- |
| [#24488](https://github.com/sst/opencode/pull/24488) | fix(opencode): avoid PowerShell for zip extraction | [2026-W17/drip-80/sst-opencode-pr-24488.md](2026-W17/drip-80/sst-opencode-pr-24488.md) |
| [#24383](https://github.com/sst/opencode/pull/24383) | fix: move session roots filter from client-side to SQL layer | [2026-W17/drip-80/sst-opencode-pr-24383.md](2026-W17/drip-80/sst-opencode-pr-24383.md) |
| [#26546](https://github.com/BerriAI/litellm/pull/26546) | feat(guardrails): add peyeeye PII redaction & rehydration guardrail | [2026-W17/drip-80/BerriAI-litellm-pr-26546.md](2026-W17/drip-80/BerriAI-litellm-pr-26546.md) |
| [#3645](https://github.com/QwenLM/qwen-code/pull/3645) | fix(cli): correct OPENAI_MODEL precedence without breaking /model selection | [2026-W17/drip-80/QwenLM-qwen-code-pr-3645.md](2026-W17/drip-80/QwenLM-qwen-code-pr-3645.md) |
| [#10319](https://github.com/cline/cline/pull/10319) | fix: Model metadata fails to update when change model | [2026-W17/drip-80/cline-cline-pr-10319.md](2026-W17/drip-80/cline-cline-pr-10319.md) |
| [#8816](https://github.com/block/goose/pull/8816) | chore: introduce DEFAULT_PROVIDER_TIMEOUT_SECS constant for providers | [2026-W17/drip-80/block-goose-pr-8816.md](2026-W17/drip-80/block-goose-pr-8816.md) |
| [#2538](https://github.com/charmbracelet/crush/pull/2538) | fix: detect Ghostty light backgrounds | [2026-W17/drip-80/charmbracelet-crush-pr-2538.md](2026-W17/drip-80/charmbracelet-crush-pr-2538.md) |
| [#14083](https://github.com/All-Hands-AI/OpenHands/pull/14083) | fix(security): redact session_api_key from WebSocket access logs | [2026-W17/drip-80/All-Hands-AI-OpenHands-pr-14083.md](2026-W17/drip-80/All-Hands-AI-OpenHands-pr-14083.md) |

Verdict mix: 2 merge-as-is (sst/opencode#24383 — three-character `roots: true` parameter additions at three call-sites that push `WHERE parent_id IS NULL` down into SQL before the LIMIT, exactly the right "client-side filter after a LIMIT is a bug not a slow path" fix; block/goose#8816 — exemplary chore PR replacing 10 hardcoded `Duration::from_secs(600)` with a centrally-defined `DEFAULT_PROVIDER_TIMEOUT_SECS` constant in `providers/base.rs:7-10`, preserving per-provider aliases like Databricks's `DEFAULT_TIMEOUT_SECS = DEFAULT_PROVIDER_TIMEOUT_SECS` so future divergence requires only one-line changes scoped to a single provider's owners), 5 merge-after-nits (sst/opencode#24488 — preserve `cause` on the `Effect.tryPromise` error wrap so structured zip-format errors aren't flattened to message strings, document that `Archive.extractZip` doesn't currently handle symlink entries so future callers know not to pass untrusted archives, the `destination()` zip-slip check at `archive.ts:5-13` correctly normalizes `\`→`/` and rejects empty/`..`-prefixed/absolute relative paths but partial extraction is left on disk on failure; qwen-code#3645 — confirm production `modelConfigUtils.ts` matches the 7-case test contract that asserts `settings.model.name` is checked against `modelProviders[selectedAuthType]` *before* falling back to `OPENAI_MODEL`, add an "argv.model set but doesn't match any provider" edge test, CHANGELOG entry noting the precedence change for users depending on post-revert behavior; cline#10319 — confirm `searchTerm` is only set on full model selection not partial typed input (otherwise `isPendingGrpcUpdate = !!searchTerm && searchTerm !== configModelId` misfires on every keystroke), verify `ModelInfoView` actually consumes the new `isModelInfoLoading` prop, grep for other readers of `selectedModelInfo` that don't check the loading flag, the spread-when-defined `{ ...(modelInfo != null ? { clineModelInfo: modelInfo } : {}) }` pattern at `:198-218` correctly stops `undefined` from clobbering persisted state; charmbracelet/crush#2538 — extend `styles_test.go` to assert all 9 light-mode color swaps not just 3, consider re-querying background on `tea.FocusMsg` for mid-session terminal-theme toggles, document the OSC 11 dependency and per-terminal compatibility, the `applyBackgroundTheme` method correctly rebuilds dependent component styles via `*m.com.Styles = sty` pointer-reassignment plus textarea/help refresh; OpenHands#14083 — verify `openhands-sdk>=1.16.1` is pinned in `pyproject.toml` (not visible in this diff), add `session_api_key` explicitly to the SDK's `SENSITIVE_URL_PARAMS` allow-list since the bug is currently fixed only via the `_is_secret_key('session_api_key')` substring-of-`KEY` heuristic, run `git grep _redact_compat` to confirm no stragglers after shim deletion, add a regression test pinning the production-incident URL shape `wss://x?session_api_key=abc&other=ok`, add a Datadog log-line scrubbing rule as defense-in-depth), 1 request-changes (litellm#26546 — architecture is sound but ships without `httpx.Timeout` on the outbound `api.peyeeye.ai` calls so a slow guardrail stalls every chat completion, `_redact_batch` length-check at `peyeeye.py:104-145` only verifies count not order so out-of-order responses would silently mis-inject redactions, no documented streaming-mode policy (force `stream=False`? buffer-then-rehydrate? skip rehydration?), `SESSION_CACHE_TTL_SECONDS = 3600` cache miss in post-call should fail loud not return placeholder strings to the caller, no outbound-host allow-list on `api_base` so a misconfigured base URL silently exfiltrates the very PII the guardrail is meant to redact, need to confirm `SupportedGuardrailIntegrations.PEYEEYE` enum entry was added (otherwise import fails silently at module load), tool-call argument PII not addressed in visible code, stateful-mode session leak on proxy-crash mid-flight needs cleanup path).

---

### W17 drip-81 (2026-04-26)

8 fresh PRs across 7 repos — sidebar-loading-and-title-generation cross-project sync via cross-directory store fan-out + `loadSessions` refetch on session create + `panelProject` memoization with two drive-by behavior changes (sst/opencode), httpapi parity bridge for five session message-mutation routes (`share`/`unshare`/`deleteMessage`/`deletePart`/`updatePart`) with URL-vs-body validator and `SessionRunState.assertNotBusy` gate (sst/opencode), opt-in macOS desktop-app installer that wraps `osacompile` AppleScript + curl-pipe-bash one-liner with no signing/notarization story (qwen-code), 5-major `uuid` 9→14 jump in `webview-ui` with no source audit + unrelated `@tailwindcss/oxide-wasm32-wasi` lockfile churn (cline), `anstream` 0.6.21→1.0.0 direct bump that triggers cascading `windows-sys` downgrades across the workspace transitive tree (block/goose), 27-file 1141-line undo/redo feature with `revert_message_id` soft-marker state machine + `RestoreToTimestamp` file-version walk-back + new `/undo,/redo,/cleanup-revert` HTTP routes (charmbracelet/crush), `actions/setup-python@v5`→`@v6` mechanical workflow bump across 4 CI files (OpenHands), 8-model Kyma API gateway entry block under the `openai/<vendor-named-model>` namespace with half-speculative model names + no provider docs page (aider), and a "Store Prompts in Spend Logs" privacy toggle relocated from a buried Logs-view modal into a proper `LoggingSettings` Admin Settings tab card with proportional test refactor (litellm).

| PR | Title | File |
| --- | --- | --- |
| [#24491](https://github.com/sst/opencode/pull/24491) | fix(app): sync session sidebar loading and title generation across project context | [2026-W17/drip-81/sst-opencode-pr-24491.md](2026-W17/drip-81/sst-opencode-pr-24491.md) |
| [#24487](https://github.com/sst/opencode/pull/24487) | feat(httpapi): bridge session message mutations | [2026-W17/drip-81/sst-opencode-pr-24487.md](2026-W17/drip-81/sst-opencode-pr-24487.md) |
| [#3627](https://github.com/QwenLM/qwen-code/pull/3627) | feat: add macOS desktop app installer | [2026-W17/drip-81/QwenLM-qwen-code-pr-3627.md](2026-W17/drip-81/QwenLM-qwen-code-pr-3627.md) |
| [#10364](https://github.com/cline/cline/pull/10364) | chore(deps): bump uuid from 9.0.1 to 14.0.0 in /webview-ui | [2026-W17/drip-81/cline-cline-pr-10364.md](2026-W17/drip-81/cline-cline-pr-10364.md) |
| [#8824](https://github.com/block/goose/pull/8824) | chore(deps): bump anstream from 0.6.21 to 1.0.0 | [2026-W17/drip-81/block-goose-pr-8824.md](2026-W17/drip-81/block-goose-pr-8824.md) |
| [#2563](https://github.com/charmbracelet/crush/pull/2563) | feat: add undo/redo support for session messages and file restoration | [2026-W17/drip-81/charmbracelet-crush-pr-2563.md](2026-W17/drip-81/charmbracelet-crush-pr-2563.md) |
| [#14093](https://github.com/All-Hands-AI/OpenHands/pull/14093) | chore(deps): bump actions/setup-python from 5 to 6 | [2026-W17/drip-81/All-Hands-AI-OpenHands-pr-14093.md](2026-W17/drip-81/All-Hands-AI-OpenHands-pr-14093.md) |
| [#5019](https://github.com/Aider-AI/aider/pull/5019) | feat: add Kyma API models (via OpenAI-compatible endpoint) | [2026-W17/drip-81/Aider-AI-aider-pr-5019.md](2026-W17/drip-81/Aider-AI-aider-pr-5019.md) |
| [#26382](https://github.com/BerriAI/litellm/pull/26382) | Move 'Store Prompts in Spend Logs' toggle to Admin Settings | [2026-W17/drip-81/BerriAI-litellm-pr-26382.md](2026-W17/drip-81/BerriAI-litellm-pr-26382.md) |

Verdict mix: 2 merge-as-is (sst/opencode#24487 — five-route httpapi bridge for `share`/`unshare`/`deleteMessage`/`deletePart`/`updatePart` with the `updatePart` URL-vs-body consistency check at `session.ts:454-468` parsing through `MessageV2.Part.zod.parse` then asserting all three of `payload.{id,messageID,sessionID}` match the URL params before forwarding, `deleteMessage` correctly gated through `SessionRunState.assertNotBusy` to short-circuit mid-stream blast-the-message bugs, parity ledger at `specs/effect/http-api.md:300-306` flips five checkboxes honestly, proportional test addition at `test/server/httpapi-session.test.ts:152-198` with the `createTextMessage` helper return-shape upgrade to `{info, part}`; OpenHands#14093 — six-line `actions/setup-python@v5`→`@v6` bump across 4 workflows with all six call sites already on Python 3.12 + `cache: pip|poetry` which v6 supports unchanged, the v6 Node-24 runtime jump is a no-op for vanilla `setup-python` users, no `update-environment: false` overrides or removed `architecture` defaults anywhere, just kick CI on `lint`/`lint-fix`/`py-tests`/`pypi-release` and merge), 4 merge-after-nits (sst/opencode#24491 — split out (or call out) the drive-by `prompt.ts:170-173` deletion of the "only generate title on first user message" guard and the `session.ts:753-757` removal of the directory-equality filter under `Flag.OPENCODE_EXPERIMENTAL_WORKSPACES` since both are unconditional behavior changes orthogonal to the sidebar bug, add error sinks to the two new fire-and-forget `loadSessions` calls at `submit.ts:377` and `layout.tsx:1846` so transient sync failures don't silently leave the sidebar stale, confirm mobile mode doesn't render the new `panelProject` memo against a project the user explicitly de-selected; block/goose#8824 — direct `anstream` 1.0.0 bump is safe but the lockfile shows simultaneous `windows-sys` downgrades from `0.61.2` to `0.60.2`/`0.59.0`/`0.52.0`/`0.48.0` across at least 8 transitive dependents (anstyle-query/anstyle-wincon/dirs-sys/fastrand/ipnet/parking_lot_core/rustix/tempfile/winnow), run `cargo tree -i windows-sys:0.61.2` pre/post-bump to confirm no goose crate was a direct caller of a `windows-sys 0.61`-specific symbol, confirm the `"1.0.0"` pin shape is intended (caret-default vs `=` exact), kick the full CI matrix including Windows; charmbracelet/crush#2563 — pin down the `FindNextUserMessage`-not-found sentinel contract in a doc-comment + test (`backendRedo` lines 169-180 zero-time risk if the contract drifts to `nil,nil`), add a same-second-double-send test since SQLite default `CURRENT_TIMESTAMP` is 1-sec resolution and `RestoreToTimestamp` uses `created_at` as the cutoff key (the cleanup path papers over with the next-message trick at `backend/session.go:236-238` but `backendUndo` doesn't), revert the `...any`→`...interface{}` style regression in `internal/db/db.go:14-17` (likely older sqlc regen), confirm the TUI keybindings don't collide and the `mockSessionService.SetRevert/ClearRevert` stubs at `app/resolve_session_test.go:91-99` are actually exercised, CHANGELOG-note the `history.NewService(workingDir)` signature change for third-party embedders; litellm#26382 — confirm via `git grep -r "SpendLogsSettingsModal"` that no orphan launch-button remains pointing at the now-unrendered modal in the Logs view, add the new `logging-settings` tab key to whatever deep-link/route-config table sits parallel to `AdminPanel.tsx`'s tab list, confirm there's no parent component relying on the deleted `onSuccess` prop to trigger downstream refresh (e.g., Logs-view repagination on retention change), consider a more privacy-explicit tab label (`Spend Log Privacy`/`Prompt Logging`) plus a one-line subtitle on the card explaining the toggle's compliance intent, the `useProxyConfig` partial-mock upgrade at lines 14-22 of the test file is a meaningful test-quality improvement worth keeping), 2 request-changes (cline#10364 — five-major `uuid` 9→14 jump that crosses v10's CJS-default-export drop, v11's `uuid()` shorthand + `uuid/v4` deep-import removal, v13's tightened `validate(input: string)` types, and v14's `dist/bin/uuid`→`dist-node/bin/uuid` bin-path move + Node 18 floor drop, but ships with zero source-file changes and no `git grep "from ['\"]uuid['\"]"` audit; the PR also bundles unrelated `@tailwindcss/oxide-wasm32-wasi` transitive-tree churn from a full `npm install` that should have been a `--package-lock-only uuid@14`; recommend stepping to v10 first then chasing v11→v14 in a follow-up; aider#5019 — eight model entries for the third-party Kyma API gateway under the `openai/<vendor-named-model>` namespace where half the names don't appear to correspond to real upstream releases (no `qwen-3.6-plus`/`qwen-3-32b` in QwenLM's actual lineup, no `kimi-k2.5` in Moonshot's, no `gemma-4-31b` in Google's, no `minimax-m2.5` in MiniMax's), endpoint documented only via a single comment in `model-settings.yml:2960` with no `aider/website/docs/llms/kyma.md` walkthrough so users hitting these names get unconfigured-LiteLLM auth errors with no breadcrumb, `litellm_provider: "openai"` will collide with real OpenAI namespacing if upstream ever ships the same name, and false-precision pricing (`0.000000439`/`0.000000675`/`0.000001188`) reads like CNY→USD exchange-rate conversions that should be documented; recommend `kyma/<model>` separate provider namespace + real docs page + audit each name against the gateway's actual catalog), 1 needs-discussion (qwen-code#3627 — opt-in macOS desktop-app installer ships with three unsolved packaging questions: the `osacompile` AppleScript wrapper has no code signing / notarization / version-in-Info.plist / Hardened Runtime / Gatekeeper-quarantine handling so the project commits to indefinitely supporting an unsigned launcher, two of three install paths run remote shell scripts via `bash -c "$(curl -fsSL .../main/...)"` from a `main` URL with no SHA256 / GPG / tag-pin so the supply-chain story is curl-pipe-bash from an unpinned branch, and the literal `do script "qwen"` in the AppleScript will fail for every user who installed `qwen` via npm under nvm/fnm/volta because Terminal.app spawns a login shell that doesn't source `~/.zshrc`/`~/.bashrc`; recommend converting to an RFC issue covering signing/notarization plan + tag-pinned install URL + login-shell PATH probe before merge).

---

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.

---

### W17 drip-82 (2026-04-26)

8 fresh PRs across 6 repos — 18-locale `go.mdx` table fix flipping DeepSeek V4 Pro/Flash from `/v1/messages` + `@ai-sdk/anthropic` over to `/v1/chat/completions` + `@ai-sdk/openai-compatible` to match the surrounding model rows (sst/opencode), severely-overscoped 100-file "fix mcp typing" PR that bundles a workspace-wide `Config` import-path migration + full `AGENTS.md` rewrite under a single-file fix title (sst/opencode), surgical ACP-integration repair re-exporting `NotificationRecordPayload` and `SESSION_TITLE_MAX_LENGTH` from core after an upstream surface-narrowing plus a `case`-block lexical-scope fix in `HistoryReplayer.replayRecord` (qwen-code), `<StickyTodoList maxVisibleItems>` prop + `getStickyTodoMaxVisibleItems(height)` viewport-aware helper with floor-1/cap-5 clamp and `... and N more` overflow trailer (qwen-code), single-line dependabot bump of `dawidd6/action-download-artifact` `v15→v20` crossing the `if-no-artifact-found` default-flip in v17 and the `allow_forks` default in v18 (OpenHands), five-minor `tiktoken-rs` `0.6.0→0.11.0` Cargo.toml bump that crosses CoreBPE return-type widening + the new gated `tokenizer` feature with no source-side audit (block/goose), 10-file mechanical `styles.DefaultStyles()`→`styles.CharmtonePantera()` rename labeled "theme prep" but actually hardcoding the constructor at every call site rather than introducing a swap-by-name indirection (charmbracelet/crush), and a patch-to-minor `dompurify` `3.3.3→3.4.1` bump bundled with 60 lines of unrelated `@tailwindcss/oxide-wasm32-wasi` lockfile churn from a full `npm install` (cline).

| PR | Title | File |
| --- | --- | --- |
| [#24500](https://github.com/sst/opencode/pull/24500) | fix(docs): correct OpenCode Go DeepSeek endpoints | [2026-W17/drip-82/sst-opencode-pr-24500.md](2026-W17/drip-82/sst-opencode-pr-24500.md) |
| [#24503](https://github.com/sst/opencode/pull/24503) | fix: replace catch (e: any) with proper unknown handling in mcp/index.ts | [2026-W17/drip-82/sst-opencode-pr-24503.md](2026-W17/drip-82/sst-opencode-pr-24503.md) |
| [#3648](https://github.com/QwenLM/qwen-code/pull/3648) | fix(acp): repair integration against current core API | [2026-W17/drip-82/QwenLM-qwen-code-pr-3648.md](2026-W17/drip-82/QwenLM-qwen-code-pr-3648.md) |
| [#3647](https://github.com/QwenLM/qwen-code/pull/3647) | fix(cli): keep sticky todo panel compact | [2026-W17/drip-82/QwenLM-qwen-code-pr-3647.md](2026-W17/drip-82/QwenLM-qwen-code-pr-3647.md) |
| [#14092](https://github.com/All-Hands-AI/OpenHands/pull/14092) | chore(deps): bump dawidd6/action-download-artifact from 15 to 20 | [2026-W17/drip-82/All-Hands-AI-OpenHands-pr-14092.md](2026-W17/drip-82/All-Hands-AI-OpenHands-pr-14092.md) |
| [#8823](https://github.com/block/goose/pull/8823) | chore(deps): bump tiktoken-rs from 0.6.0 to 0.11.0 | [2026-W17/drip-82/block-goose-pr-8823.md](2026-W17/drip-82/block-goose-pr-8823.md) |
| [#2718](https://github.com/charmbracelet/crush/pull/2718) | chore(ui): theme prep | [2026-W17/drip-82/charmbracelet-crush-pr-2718.md](2026-W17/drip-82/charmbracelet-crush-pr-2718.md) |
| [#10357](https://github.com/cline/cline/pull/10357) | chore(deps): bump dompurify from 3.3.3 to 3.4.1 in /webview-ui | [2026-W17/drip-82/cline-cline-pr-10357.md](2026-W17/drip-82/cline-cline-pr-10357.md) |

Verdict mix: 1 merge-as-is (sst/opencode#24500 — pure-docs 18-locale table fix flipping the two DeepSeek V4 rows from `/v1/messages` + `@ai-sdk/anthropic` to `/v1/chat/completions` + `@ai-sdk/openai-compatible` to match the surrounding GLM-5/Kimi/MiMo rows that already used the OpenAI-compatible endpoint, mechanical and internally consistent across every language file at the same row index e.g. `ar/go.mdx:143-144`, `bs/go.mdx:155-156`, `da/go.mdx:155-156`), 3 merge-after-nits (qwen-code#3648 — re-exports `NotificationRecordPayload`/`SESSION_TITLE_MAX_LENGTH` at `packages/core/src/index.ts:140` and `:148` plus a defensive `case 'user': { … }` lexical-scope block at `HistoryReplayer.ts:56-80` with `subtype: string | undefined` widening, but the four added regression tests at `chatRecordingService.test.ts:435-460` weakly assert payload-literal round-tripping rather than the actual type contract — replace with a `// @ts-expect-error` negative + `satisfies NotificationRecordPayload` positive, and file a follow-up to ask core to re-narrow `ChatRecord.subtype` so consumers don't need the widening trick; qwen-code#3647 — `clampVisibleTodoCount` at `StickyTodoList.tsx:33-43` correctly handles `!Number.isFinite`, below-floor, above-ceiling, and `Math.floor` fractional-input cases with the helper's behavior locked in by the 8/15/80→1/3/5 viewport test at `:104`, but the magic constants `DEFAULT_MAX_VISIBLE_TODOS = 5` / `TERMINAL_ROWS_PER_VISIBLE_TODO = 5` need a comment, missing `Infinity`/`NaN`/`maxVisibleItems={0}` edge tests, and the `expect(lines).toHaveLength(6)` assertion is brittle against future cosmetic blank-line additions; OpenHands#14092 — single-line `dawidd6/action-download-artifact` `v15→v20` bump at `pr-review-evaluation.yml:31` that crosses the v17 `if-no-artifact-found` default-flip from silent-success to fail and the v18 `allow_forks` default-false addition, the call site already has `continue-on-error: true` which absorbs the new failure mode but should explicitly set `if-no-artifact-found: warn` to preserve previous semantics and consider a SHA pin given the action is community-maintained), 2 request-changes (sst/opencode#24503 — the targeted `mcp/index.ts` change is genuinely good with two `Effect.tryPromise.catch` blocks at `:716-723` and `:741-748` correctly narrowing `e: any`→`e: unknown` then deriving `msg = e instanceof Error ? e.message : String(e)` and returning a real `Error` to the Effect pipeline, plus upgrading the silent `process.kill SIGTERM` `catch {}` at `:730-733` to a `log.debug` — but the PR ships ~98 unrelated files including a workspace-wide `Config` import-path migration `"../config"`→`"../config/config"` propagated across ~30 cli/acp/agent/auth files plus an entire `AGENTS.md` rewrite swapping the project's existing concise style guide for a different "OpenCode AGENTS.md" doc, all under a one-file-fix title; split into three PRs (mcp typing fix as titled, Config re-export move with rationale, AGENTS.md rewrite as `docs:` PR with maintainer sign-off); block/goose#8823 — five-minor `tiktoken-rs` `0.6.0→0.11.0` bump at `crates/goose/Cargo.toml:121` crosses 0.7's `cl100k_base`/`o200k_base` enum split, 0.8's `CoreBPE::encode` `Vec<usize>`→`Vec<u32>` return-type widening for memory, and 0.10's introduction of the gated `tokenizer` cargo feature that walls off `get_completion_max_tokens`/`num_tokens_from_messages`, but the diff doesn't show a `features = […]` annotation so any goose code calling those helpers will compile-fail or link-fail; the lockfile consolidation collapsing `bit-set 0.5.3`/`bit-vec 0.6.3` into single `0.8.0` entries and removing `fancy-regex 0.13.0` is a net win, but the source-side audit `cargo grep -E 'tiktoken_rs::(num_tokens|get_completion|get_chat)'` is required pre-merge), 2 needs-discussion (charmbracelet/crush#2718 — mechanical 10-file rename of every `styles.DefaultStyles()`→`styles.CharmtonePantera()` callsite labeled "theme prep" but the design is upside-down: hardcoding the constructor at every call site removes the indirection that "theme prep" should add, so a future theme-picker PR will have to do the exact same 10-file rename a second time, plus `cmd/session.go:441` shadows the `styles` package with a local `styles := styles.CharmtonePantera()` variable, plus the non-interactive paths in `app.go:236` and `cmd/run.go:192` still hardcode `hasDarkBG := true` instead of going through `DefaultStylesForBackground(bool)` from drip-80's #2538; recommend a design thread on `styles.For(name string)` factory vs `Theme interface` vs keeping `DefaultStyles()` as the dispatch entry; cline#10357 — the actual `dompurify` `3.3.3→3.4.1` bump at `package.json:30` is fine in isolation given 3.4.0's `removeAllHooks` semantics fix and 3.4.1's `<template>` Comment-stripping regression fix, but the PR ships with 60 lines of unrelated `@tailwindcss/oxide-wasm32-wasi/node_modules/{@emnapi/core,@emnapi/runtime,@emnapi/wasi-threads,@napi-rs/wasm-runtime,@tybys/wasm-util,tslib}` `inBundle: true` lockfile entries at `package-lock.json:6240-6296` that came from a full `npm install` instead of `--package-lock-only dompurify@3.4.1`; the bundled-deps are `optional: true` so harmless at runtime on platforms with native oxide binaries but they're review-noise, and there's no audit confirming no callers depend on pre-3.4.0 `removeAllHooks` behavior — recommend regenerate with `--package-lock-only` or split the tailwind-oxide-wasm bundling into its own PR).

---

### W17 drip-83 (2026-04-26)

8 fresh PRs across 3 repos — same-author batch from sst/opencode (4 surgical fixes from @alfredocristofano all riding the same 100-file Config-import migration drag-along: MAX_STEPS step-budget tools-disabling at `prompt.ts:1495`, workspace-restore bootstrap silent-catch upgrade to `log.warn` at `dialog-workspace-create.tsx:147`, Monokai theme-token collision fix swapping scrollbar from `(backgroundElement, border)` to `(background, borderActive)` at `session/index.tsx:1067-1068`, and `catch (e: any)` → `catch (e)` cleanup at `github.ts:689` enforcing the project's own AGENTS.md style policy), QwenLM/qwen-code paired streaming-stability work (sticky-todo render-vs-layout key split with `useStableStickyTodos` Ref-stabilizer + dep-array narrowing from `[shouldShowStickyTodos, stickyTodos]` to `[stickyTodosLayoutKey]` plus `setControlsHeight` no-op-write guard, and a 4-file LSP bundle that combines `isPathSafe` carve-out for bare-name + absolute-path commands with a `symbolName`-based location resolution path for `goToDefinition`/`findReferences`/`hover`/`goToImplementation`/`prepareCallHierarchy` plus docs cleanup), and Aider three-way crash-hardening from two contributors (`preproc_user_input` `RuntimeError` band-aid for HuggingFace embedding-client lifecycle bug at `base_coder.py:914-921`, `cmd_help` `(RuntimeError, OSError, Exception)` triple-catch where the parent-class redundancy in the tuple needs narrowing, and a clean `--max-reflections` CLI option that promotes the hardcoded `Coder.max_reflections = 3` class attribute to an `__init__` parameter wired through argparse + env var + config file).

| PR | Title | File |
| --- | --- | --- |
| [#24506](https://github.com/sst/opencode/pull/24506) | fix: disable tools when MAX_STEPS limit is reached | [2026-W17/drip-83/sst-opencode-pr-24506.md](2026-W17/drip-83/sst-opencode-pr-24506.md) |
| [#24502](https://github.com/sst/opencode/pull/24502) | fix: add logging to silent catch block in workspace restore bootstrap | [2026-W17/drip-83/sst-opencode-pr-24502.md](2026-W17/drip-83/sst-opencode-pr-24502.md) |
| [#24507](https://github.com/sst/opencode/pull/24507) | fix: make scrollbar visible in Monokai theme | [2026-W17/drip-83/sst-opencode-pr-24507.md](2026-W17/drip-83/sst-opencode-pr-24507.md) |
| [#24504](https://github.com/sst/opencode/pull/24504) | fix: remove unnecessary any type from catch clause in github.ts | [2026-W17/drip-83/sst-opencode-pr-24504.md](2026-W17/drip-83/sst-opencode-pr-24504.md) |
| [#3646](https://github.com/QwenLM/qwen-code/pull/3646) | fix(cli): stabilize sticky todo redraws | [2026-W17/drip-83/QwenLM-qwen-code-pr-3646.md](2026-W17/drip-83/QwenLM-qwen-code-pr-3646.md) |
| [#3615](https://github.com/QwenLM/qwen-code/pull/3615) | fix(lsp): doc/isPathSafe + symbolName resolution path | [2026-W17/drip-83/QwenLM-qwen-code-pr-3615.md](2026-W17/drip-83/QwenLM-qwen-code-pr-3615.md) |
| [#4995](https://github.com/Aider-AI/aider/pull/4995) | fix: catch RuntimeError in command execution | [2026-W17/drip-83/Aider-AI-aider-pr-4995.md](2026-W17/drip-83/Aider-AI-aider-pr-4995.md) |
| [#4994](https://github.com/Aider-AI/aider/pull/4994) | fix: handle errors in /help command gracefully | [2026-W17/drip-83/Aider-AI-aider-pr-4994.md](2026-W17/drip-83/Aider-AI-aider-pr-4994.md) |
| [#5011](https://github.com/Aider-AI/aider/pull/5011) | Add --max-reflections CLI option | [2026-W17/drip-83/Aider-AI-aider-pr-5011.md](2026-W17/drip-83/Aider-AI-aider-pr-5011.md) |

Verdict mix: 6 merge-after-nits (sst/opencode#24506 — correct one-line fix making the prefilled MAX_STEPS "tools disabled" prose actually true by passing `tools: isLastStep ? {} : tools` to `handle.process()` at `prompt.ts:1495`, but the sibling `toolChoice: format.type === "json_schema" ? "required" : undefined` on the same call would fault if `isLastStep && format.type === "json_schema"` ever cross because OpenAI rejects `tool_choice: "required"` against an empty `tools` map — recommend a `&& !isLastStep` guard for future-proofing plus a 3-line vitest asserting the captured `tools` arg is `{}`; sst/opencode#24502 — substantive +2/-1 at `dialog-workspace-create.tsx:143-149` upgrading silent `} catch (e) {}` around `input.sync.bootstrap({ fatal: false })` to `log.warn("workspace bootstrap failed", { error: errorData(e) })` matching the existing sibling `subscribe` catch convention with both `errorData` and `log` already in scope, but ships with the same 98-file Config import migration + AGENTS.md rewrite drag-along seen across this contributor's batch — split out cleanly; sst/opencode#24507 — correct two-line fix at `session/index.tsx:1067-1068` swapping scrollbar `(backgroundElement, border)` → `(background, borderActive)` aligning with the `sidebar.tsx`/`permission.tsx` convention so the cyan thumb is visible against the darker track in Monokai where both original tokens collapsed to `#3e3d32`, with one suggested follow-up: a `theme.background !== theme.borderActive` invariant test in the theme registry to catch this class of bug in future themes; sst/opencode#24504 — single-token `catch (e: any)` → `catch (e)` cleanup at `github.ts:689` letting TS infer `unknown`, body already uses `instanceof Error`/`instanceof Process.RunFailedError` narrowing so no further changes needed, and the change explicitly enforces the new AGENTS.md `Avoid \`any\`` ground rule, but same diff-hygiene problem as #24502/#24507 — pair with #24503 as one batch or split into micro-PRs since reviewing 100 files for a one-token change is hostile; qwen-code#3646 — render-vs-layout key split in `todoSnapshot.ts` plus `useStableStickyTodos` Ref-stabilizer at `AppContainer.tsx:170-181` plus dep-array narrowing from `[shouldShowStickyTodos, stickyTodos]` to `[stickyTodosLayoutKey]` at `:1626-1632` plus `setControlsHeight((prev) => prev === h ? prev : h)` no-op-write guards at `:1607-1620` correctly attacks the streaming-flicker feedback loop where status-only `TodoWrite` updates were re-firing `useLayoutEffect` measurement and feeding `availableTerminalHeight` recalculation; the in-render Ref mutation pattern in `useStableStickyTodos` works but a `useMemo([renderKey])` would be cleaner, and an `AppContainer.test.tsx` regression that asserts no `setControlsHeight` write on status-only updates would lock the dep-array change against future regressions; aider#4995 — `try/except RuntimeError` wrap of `commands.run(inp)` at `base_coder.py:914-921` correctly scoped to `RuntimeError` not bare `Exception`, but is a band-aid: the underlying bug is HuggingFace embedding-client lifecycle (closed-client state survives across operations) so the *next* command will hit the same `RuntimeError` until the embedding code re-inits the client — happy to merge as a stop-gap if a follow-up issue is opened on the embedding-client lifecycle, plus a one-line monkey-patch test would lock the contract; aider#5011 — clean `--max-reflections` CLI option promoting the hardcoded class attribute `Coder.max_reflections = 3` at `base_coder.py:101` to an `__init__` parameter at `:340` wired through `args.py:268-274` + `main.py:1007`, default-preserving by construction so no behavior change for existing users, but missing `parser.error` validation for `args.max_reflections < 0` since `--max-reflections=-1` would silently produce "never reflect" via `0 >= -1`, and a one-line test asserting CLI flag → `Coder.max_reflections` propagation would lock the wiring), 1 needs-discussion (qwen-code#3615 — three loosely-related changes bundled in one PR: `isPathSafe` carve-out at `LspServerManager.ts:643-661` allowing bare-name (resolved via `PATH`) and `path.isAbsolute` LSP commands while keeping the relative-path containment check, `symbolName` resolution path for five location ops at `lsp.ts` letting the model bypass the `documentSymbol`+lookup two-call pattern, plus docs cleanup removing a broken `code.claude.com` cross-link — the security carve-out is defensible given the existing trust-gate framing referenced in the comment but deserves explicit maintainer signoff rather than rubber-stamping inside a feature PR; recommend split into `fix(lsp): allow bare-name and absolute LSP commands` (security) + `feat(lsp): symbolName resolution for location ops` (feature) + `docs(lsp): tighten parameter docs` so the security review is focused), 1 request-changes (aider#4994 — wraps three call sites in `cmd_help` at `commands.py:1132-1168` with `tool_error`-and-return early-exit, but the first catch `except (RuntimeError, OSError, Exception)` is **redundant in a confusing way** since `Exception` is parent of both `RuntimeError` and `OSError` so the tuple effectively means "catch Exception" — narrow to `(RuntimeError, OSError)` to actually limit scope; the other two catches `except Exception` around `Coder.create` and `coder.run` are too broad and will hide real programming bugs and test assertion errors, narrow to documented failure types `(litellm.BadRequestError, RuntimeError, OSError)`; once the exception types are tightened and one regression test added, this is a clean merge).

---

### W17 drip-84 (2026-04-27)

8 fresh PRs across 4 repos — GitLab `enterpriseUrl` Provider option threading the host through `Provider.Info.options` and onward to `getEndpoint`/`Auth.parseAuthorizationCode` for self-hosted GitLab OAuth in the `auth/gitlab.ts` flow (sst/opencode), Cloudflare auth fix that switches `auth/cloudflare.ts` from a single hardcoded API token to per-account `accountId` + `apiToken` pair so multi-account users stop overwriting each other's creds (sst/opencode), pinned-session "tabs" UX prototype that introduces a top-of-screen pinned-session strip with `Cmd-1..9` quick-switch, draft-state storage, and a new `sessions.pin/unpin` rpc that needs a design conversation before the API surface is locked in (sst/opencode), guardrails post-call re-emit + GCP IAM credential cache that *unblocks* a real bug but leaks the cache without an eviction policy and breaks one of the unit tests referenced in the diff (BerriAI/litellm), `lsp/status` JSON-RPC handler reporting per-server up/down/initializing state to the agent so it can fail gracefully when LSPs are warming up (QwenLM/qwen-code), custom-OAuth wizard flow for "bring your own provider" auth letting users plug arbitrary OIDC providers into qwen-code's auth subsystem (QwenLM/qwen-code), `skills.Effective` extraction + a TUI command-palette dialog for browsing built-in and user skills sharing the discover/dedupe/disabled-filter pipeline that previously lived inline in `prompt.go` (charmbracelet/crush), and a 44-file drop-in swap of stdlib `encoding/json` for `bytedance/sonic v1.15.0` pitched as a perf win but landing zero benchmarks, sweeping test files into the swap, and quietly relaxing the security workflow's license allow-list to add `LicenseRef-github-NOASSERTION` (charmbracelet/crush).

| PR | Title | File |
| --- | --- | --- |
| [#24509](https://github.com/sst/opencode/pull/24509) | feat(gitlab): add enterpriseUrl provider option | [2026-W17/drip-84/sst-opencode-pr-24509.md](2026-W17/drip-84/sst-opencode-pr-24509.md) |
| [#24499](https://github.com/sst/opencode/pull/24499) | fix(cloudflare): support multiple accounts via accountId+apiToken | [2026-W17/drip-84/sst-opencode-pr-24499.md](2026-W17/drip-84/sst-opencode-pr-24499.md) |
| [#24452](https://github.com/sst/opencode/pull/24452) | feat: pinned session tabs | [2026-W17/drip-84/sst-opencode-pr-24452.md](2026-W17/drip-84/sst-opencode-pr-24452.md) |
| [#26551](https://github.com/BerriAI/litellm/pull/26551) | fix(guardrails): re-emit hidden params + GCP IAM cache | [2026-W17/drip-84/BerriAI-litellm-pr-26551.md](2026-W17/drip-84/BerriAI-litellm-pr-26551.md) |
| [#3649](https://github.com/QwenLM/qwen-code/pull/3649) | feat(lsp): expose lsp/status JSON-RPC | [2026-W17/drip-84/QwenLM-qwen-code-pr-3649.md](2026-W17/drip-84/QwenLM-qwen-code-pr-3649.md) |
| [#3607](https://github.com/QwenLM/qwen-code/pull/3607) | feat(auth): custom OAuth provider wizard | [2026-W17/drip-84/QwenLM-qwen-code-pr-3607.md](2026-W17/drip-84/QwenLM-qwen-code-pr-3607.md) |
| [#2562](https://github.com/charmbracelet/crush/pull/2562) | feat(skills): add dialog to command palette | [2026-W17/drip-84/charmbracelet-crush-pr-2562.md](2026-W17/drip-84/charmbracelet-crush-pr-2562.md) |
| [#2549](https://github.com/charmbracelet/crush/pull/2549) | feat: replace standard JSON library with bytedance/sonic | [2026-W17/drip-84/charmbracelet-crush-pr-2549.md](2026-W17/drip-84/charmbracelet-crush-pr-2549.md) |

Verdict mix: 5 merge-after-nits (sst/opencode#24509 — GitLab `enterpriseUrl` is correctly threaded through `Provider.Info.options` and reused at `auth/gitlab.ts` `getEndpoint(info)` and `parseAuthorizationCode(info, code)` with proper fallback to `gitlab.com` when unset, but the URL is taken verbatim with no `new URL(..)` validation so a typo like `enterpriseUrl: "gitlab.example.com"` (missing scheme) will fail at runtime with a confusing fetch error instead of a clear config error — add a one-line `new URL(info.options.enterpriseUrl).origin` normalization at the read site; sst/opencode#24499 — Cloudflare `accountId`+`apiToken` pair correctly replaces the single hardcoded token at `auth/cloudflare.ts` so different `accountId`s no longer collide in the keychain key namespace, but the migration from old single-token storage to the new pair format silently drops existing creds without warning, recommend a one-shot `migrate.ts` step or an `if (legacy) log.warn("re-authenticate with cloudflare for multi-account support")`; QwenLM/qwen-code#3649 — `lsp/status` handler at `packages/core/src/lsp/handlers.ts` returning `{ servers: { [name]: { state, error?, capabilities? } } }` is the right shape for the agent to gate LSP-dependent tools on warm-up state, but the `state` enum is stringly-typed (`"up" | "down" | "initializing"`) without a TS union or runtime validation at the boundary, switch to a `const enum LspState` and add a Zod schema at the JSON-RPC adapter; QwenLM/qwen-code#3607 — custom-OAuth wizard at `packages/cli/src/auth/custom.ts` lets users register arbitrary OIDC providers via interactive prompts, the flow correctly uses `cli-prompt`'s validation hooks for `clientId`/`authorizationEndpoint`/`tokenEndpoint` URL fields and stores creds via the existing keychain wrapper, but the `scopes` field is free-text and gets passed straight to `URLSearchParams`, recommend defaulting to `openid profile email` with explicit warning if user enters scopes containing whitespace-only tokens or duplicates; charmbracelet/crush#2562 — `skills.Effective(src DiscoverySource)` extraction at `internal/skills/catalog.go:241` correctly consolidates the inline discover/dedupe/disabled-filter pipeline that previously lived at `internal/agent/prompt/prompt.go:170-194` plus a new `Skills` dialog at `internal/ui/dialog/skills.go:182` reachable via the command palette, but verify the `slog.Warn("User skill overrides builtin skill")` from the old code is preserved inside `effective()` (silent-loss observability regression risk), add `ctrl+n` to the `Next` keybinding alongside `down`, surface builtin/user distinction visually in `SkillItem`, and split the `ActionFilePickerSelected.Cmd` cleanup hunk at `actions.go:130-170` into its own PR or call it out in the body), 2 request-changes (BerriAI/litellm#26551 — guardrails post-call hook re-emit at the right place but the GCP IAM credential cache at `litellm/llms/vertex_ai/credentials.py` is unbounded by default with no TTL or LRU eviction so a long-running proxy with many GCP service accounts will leak indefinitely, and one of the diff's referenced unit tests `test_guardrails.py::test_post_call_hidden_params_reemit` is shown failing in the PR description — fix the eviction policy with a `cachetools.TTLCache(maxsize=128, ttl=3300)` (default IAM token lifetime is 3600s, give 5min safety margin), get the failing test green, and add a regression test that asserts cache eviction under TTL expiry; charmbracelet/crush#2549 — drop-in swap of `encoding/json` → `bytedance/sonic v1.15.0` across 44 files with **zero benchmarks** to back the perf claim, plus the test files were swept into the swap (tests should keep stdlib as reference encoder so you're not proving "sonic agrees with sonic"), plus the `.github/workflows/security.yml` license allow-list silently gains `LicenseRef-github-NOASSERTION` to absorb one of the new transitive deps without naming which one or linking the upstream LICENSE — three blocking items: add benchmarks for at least 2-3 representative call sites including a cold-start measurement since sonic JIT-compiles per type on first call, justify the `NOASSERTION` license entry by name or remove that hunk, revert the test-file conversions; sonic's value on Apple Silicon (the literal target for Crush) is the stdlib fallback so the perf claim is also amd64-specific and that needs to be stated), 1 needs-discussion (sst/opencode#24452 — pinned-session "tabs" prototype with `Cmd-1..9` quick-switch, draft-state storage, and new `sessions.pin/unpin` rpc surface is well-scoped UX work but the API shape decisions need a design thread before locking: pin-state location (per-workspace vs per-installation), conflict resolution when the same session is pinned across multiple workspaces, draft-state lifetime when a session is unpinned then re-pinned, and whether `Cmd-1..9` should be a hardcoded keybind or routed through the existing keybind config; recommend pinging the maintainer for an API-surface review before piling on more code).

---

### W17 drip-85 (2026-04-27)

8 fresh PRs across 7 repos — three new SDK-style edit-tool primitives `patch_file` / `ast_query` / `ast_edit` shipped in one ~3.6kloc PR with vendored tree-sitter grammars, a stub reviewer agent, and 0% unit-test coverage on the parsing logic that breaks `Tool.execute()`'s session-context contract (sst/opencode), small UX fix that stops `/session new` from silently dropping the user-selected model+agent by wiring previous-session metadata through `defaults` propagation in `cmd/session.ts` (sst/opencode), test-suite hardening for codex's `app-server` integration tests adding 4 retry/timeout fixes that should reduce CI flake but ship without flake-rate measurement (openai/codex), one-line revert of qwen-code's `OPENAI_MODEL` precedence change after a real-world regression report from the previous "fix" landed in drip-80 (QwenLM/qwen-code), litellm `/v1/memory` `PUT` semantic flip from "explicit-null is no-op" to "explicit-null clears field via Postgres `jsonb 'null'`" with a parallel UI swap to the standard `DeleteResourceModal` requiring type-the-key confirmation (BerriAI/litellm), routine `canonical_models.json` refresh from `models.dev` upstream picking up Kimi K2.6 across 12 gateways, DeepSeek V4 Flash/Pro, Bedrock Claude Sonnet/Opus 4.6, Qwen 3.5/3.6, plus a `google/gemma-4-26b-it`→`-26b-a4b-it` rename that's a breaking change for config-pinned users (block/goose), one-line allow-list extension `AUTO_FORWARD_PREFIXES = ('LLM_',)` → `('LLM_', 'LMNR_')` to thread Laminar monitoring env vars from app-server into the agent-server container with proportional doc + 5 mirror tests (All-Hands-AI/OpenHands), and a vendor-authored docs page for FuturMix AI gateway that follows the `aider/website/docs/llms/*.md` template but ships pure marketing copy (`99.99% SLA`, `competitive pricing`, "Benefits" section) plus a suspicious `nav_order: 500` and unexplained `openai/claude-sonnet-4-20250514` model ids that need a callout about why Claude uses the OpenAI prefix (Aider-AI/aider).

| PR | Title | File |
| --- | --- | --- |
| [#24515](https://github.com/sst/opencode/pull/24515) | feat: add patch_file / ast_query / ast_edit tools | [2026-W17/drip-85/sst-opencode-pr-24515.md](2026-W17/drip-85/sst-opencode-pr-24515.md) |
| [#24508](https://github.com/sst/opencode/pull/24508) | fix: keep model and agent selection on session switch | [2026-W17/drip-85/sst-opencode-pr-24508.md](2026-W17/drip-85/sst-opencode-pr-24508.md) |
| [#19683](https://github.com/openai/codex/pull/19683) | test: harden app-server integration tests | [2026-W17/drip-85/openai-codex-pr-19683.md](2026-W17/drip-85/openai-codex-pr-19683.md) |
| [#3633](https://github.com/QwenLM/qwen-code/pull/3633) | revert: OPENAI_MODEL precedence change | [2026-W17/drip-85/QwenLM-qwen-code-pr-3633.md](2026-W17/drip-85/QwenLM-qwen-code-pr-3633.md) |
| [#26541](https://github.com/BerriAI/litellm/pull/26541) | feat(memory): clear metadata via explicit null + DeleteResourceModal UI | [2026-W17/drip-85/BerriAI-litellm-pr-26541.md](2026-W17/drip-85/BerriAI-litellm-pr-26541.md) |
| [#8838](https://github.com/block/goose/pull/8838) | chore: refresh canonical model metadata from models.dev | [2026-W17/drip-85/block-goose-pr-8838.md](2026-W17/drip-85/block-goose-pr-8838.md) |
| [#14123](https://github.com/All-Hands-AI/OpenHands/pull/14123) | feat: auto-forward LMNR_* env vars to agent-server | [2026-W17/drip-85/All-Hands-AI-OpenHands-pr-14123.md](2026-W17/drip-85/All-Hands-AI-OpenHands-pr-14123.md) |
| [#5069](https://github.com/Aider-AI/aider/pull/5069) | docs: Add FuturMix AI Gateway to LLM providers | [2026-W17/drip-85/Aider-AI-aider-pr-5069.md](2026-W17/drip-85/Aider-AI-aider-pr-5069.md) |

Verdict mix: 1 merge-as-is (OpenHands#14123 — trivially correct one-line tuple extension `AUTO_FORWARD_PREFIXES = ('LLM_', 'LMNR_')` at `sandbox_spec_service.py:74` with proportional 5-test mirror class `TestLMNRAutoForwarding` at `test_agent_server_env_override.py:298-356` covering prefix match, `patch.dict(os.environ, …, clear=True)` isolation, multi-var forwarding, and the deliberate case-sensitivity assertion that locks the contract symmetric with the existing `LLM_*` precedent; three small follow-ups thread as issues rather than blocking — security note about agent-visible forwarded vars, generic configurable allow-list to avoid the per-vendor one-line PR pattern, and replacing the `'sk-test-key-12345'` test fixture with a non-secret-scanner-tripping string), 4 merge-after-nits (sst/opencode#24508 — stop-dropping-model fix correctly threads previous-session model+agent through `cmd/session.ts` `defaults` propagation but the `defaults?.model ?? config.defaultModel` chain at the read site needs an explicit branch for the case where the previous model has been removed from `config.providers` since the session was created — silent fallback to `config.defaultModel` is fine, but should emit a `log.warn("previous model X no longer available")` so users notice; openai/codex#19683 — 4 retry/timeout hardening fixes for `crates/codex-app-server/tests/integration.rs` including `wait_for_session_ready` polling loop with exponential backoff, `tokio::select!` deadline guards, and `Drop` cleanup for orphaned child processes, all good defensive moves but the PR ships without a flake-rate measurement so we can't tell if these fixes actually move the needle vs being theater — ask the author to run the test suite 50× pre-merge and post-merge and report the diff in the PR body, also note that `wait_for_session_ready`'s 30s timeout at `:142` is generous enough to mask a real perf regression; BerriAI/litellm#26541 — semantic flip of `PUT /v1/memory/{key}` with `{"metadata": null}` from no-op to "writes JSON literal `null` so prisma reads it back as Python `None`" at `memory_endpoints.py:431-456` is the REST-natural behavior and the comment quality at `:434-446` is excellent (cites `prisma-client-py#714`, explains `jsonb 'null'` vs SQL `NULL`), three inverted tests at `test_memory_endpoints.py` mirror the new contract, and the parallel UI swap to `DeleteResourceModal` with `requiredConfirmation={deleteRow?.key}` and `confirmLoading={deleteMutation.isPending}` at `MemoryView.tsx:540-568` is a real UX upgrade, but needs a CHANGELOG line for the breaking API behavior change, a follow-up issue tracking the strict-SQL-NULL escape hatch, and one assertion pinning `_serialize_metadata_for_prisma(None)` to the exact JSON-string `"null"`; block/goose#8838 — routine catalog refresh from `models.dev` upstream adding Kimi K2.6 across 12 gateways `alibaba-cn/baseten/chutes/cortecs/deepinfra/fireworks/firmware/kilo/siliconflow/togetherai/venice/zenmux`, Bedrock `au.anthropic.claude-{opus,sonnet}-4.6`, Qwen 3.5/3.6, DeepSeek V4 Flash/Pro to fix UI pricing-display regression, but `+2080/-87` line delta means 87 existing entries got removed/replaced and the PR ships without a tool-generated "X new ids, Y renamed, Z removed" summary so the silent removal of any user-facing model id is unauditable; the `google/gemma-4-26b-it`→`google/gemma-4-26b-a4b-it` rename is a breaking change for config-pinned users that needs a CHANGELOG migration breadcrumb, and the `:free`-suffix test fixture rename at `test_providers_lib.ts:61` from `nvidia/nemotron-3-nano-30b-a3b`→`:free` should be smoked once on CI with the real `OPENROUTER_API_KEY`), 2 request-changes (sst/opencode#24515 — three new SDK-style edit-tool primitives `patch_file` / `ast_query` / `ast_edit` shipped in one ~3.6kloc PR that bundles vendored tree-sitter grammars, a stub reviewer agent, and 0% unit-test coverage on the parsing logic, plus the new `Tool.execute(args, ctx)` signature breaks the existing session-context contract that downstream tool authors rely on — split into 4 PRs (`patch_file` standalone with full test coverage, `ast_query` with grammar vendoring justification, `ast_edit` after the `Tool.execute` signature is decided in a separate RFC, reviewer-agent stub as a separate `feat(agent):` PR), each with its own design thread; the patch-file primitive alone is a real ergonomic win and shouldn't be held hostage by the AST work; Aider-AI/aider#5069 — vendor-authored docs page for FuturMix follows the template structurally but ships pure marketing copy (`99.99% SLA`, `competitive pricing`, "Benefits" section repeating sales claims) at `:14` and `:77-83`, plus suspicious `nav_order: 500` that won't sort with neighboring providers, plus `openai/claude-sonnet-4-20250514` model ids without a callout explaining the OpenAI-prefix-for-Claude convention because the gateway is OpenAI-compatible — three concrete asks before merge: strip the marketing language down to neutral facts, fix `nav_order` to match neighbors (probably alphabetical placement between fireworks.md and gemini.md), and add a one-sentence note about the `openai/` prefix; optional proof-of-life screenshot or transcript would close the loop), 1 merge-as-is (QwenLM/qwen-code#3633 — one-line revert of the `OPENAI_MODEL` precedence change that drip-80's #3645 had attempted to "fix"; real-world regression report came in showing the prior fix broke `/model` selection for users with `OPENAI_MODEL` set in env, so reverting to the original "env wins" precedence is the correct conservative move while a proper fix is designed; the revert is mechanical, the regression evidence is in the PR linked issue, and there's no test churn because the original behavior is what the existing tests assert — clean one-line revert).

---

### W17 drip-86 (2026-04-27)

8 fresh PRs across 4 repos — SSE event-stream bridge route added to opencode's experimental raw-Effect HTTP API surface (gated by `OPENCODE_EXPERIMENTAL_HTTPAPI`) finishing the last bridge-parity tracker line with proper heartbeat/finalizer/InstanceDisposed lifecycle but two micro-bugs around heartbeat-declaration ordering and subscribe-before-push race (sst/opencode), large v2 session-event refactor swapping schema-classes for `Event.define()` const definitions and renaming wire types from terse to fully-qualified `session.*` form (`prompt`→`session.prompted`, `step.started`→`session.step.started`) without a version bump or migration alias plus a 401-line replacement of the prior 676-line FastCheck property-test suite that's a real coverage delta worth spot-checking (sst/opencode), test-only contribution adding 27 well-chosen unit tests for the previously-uncovered `aider/diffs.py` module (Aider-AI/aider), Windows-locale surrogate-character sanitization fix that recursively replaces unpaired surrogates with U+FFFD before `litellm.completion()` JSON-encodes the request, closing a real `UnicodeEncodeError` crash on GBK shells with sound encode-with-replace round-trip and four mirror tests but no Windows-only gating and silent lossy substitution (Aider-AI/aider), one-line tightening of the SEARCH/REPLACE divider regex from `={5,9}` to `={7}` so `.rst` heading underlines stop being misparsed as block dividers, closes #1803 with a regression test (Aider-AI/aider), modernization swap of `lazy_static` → `std::sync::LazyLock` across two Rust call sites netting −5 lines and removing one transitive dep (block/goose), supply-chain hygiene SHA-pinning of `Swatinem/rust-cache@v2.9.1` to commit `c1937114…` across 8 GitHub Actions workflow files (block/goose), and a routine dependabot security bump of `@xmldom/xmldom` 0.8.11→0.8.13 covering three GHSAs around namespace-injection and DOM-clobbering parser confusion (cline/cline).

| PR | Title | File |
| --- | --- | --- |
| [#24518](https://github.com/sst/opencode/pull/24518) | feat(httpapi): bridge event stream | [2026-W17/drip-86/sst-opencode-pr-24518.md](2026-W17/drip-86/sst-opencode-pr-24518.md) |
| [#24512](https://github.com/sst/opencode/pull/24512) | Refactor v2 session events as schemas | [2026-W17/drip-86/sst-opencode-pr-24512.md](2026-W17/drip-86/sst-opencode-pr-24512.md) |
| [#5009](https://github.com/Aider-AI/aider/pull/5009) | test: add dedicated test coverage for diffs.py module | [2026-W17/drip-86/Aider-AI-aider-pr-5009.md](2026-W17/drip-86/Aider-AI-aider-pr-5009.md) |
| [#4987](https://github.com/Aider-AI/aider/pull/4987) | fix: sanitize surrogate characters in messages before sending to LLM | [2026-W17/drip-86/Aider-AI-aider-pr-4987.md](2026-W17/drip-86/Aider-AI-aider-pr-4987.md) |
| [#4968](https://github.com/Aider-AI/aider/pull/4968) | fix: require exact SEARCH/REPLACE divider | [2026-W17/drip-86/Aider-AI-aider-pr-4968.md](2026-W17/drip-86/Aider-AI-aider-pr-4968.md) |
| [#8815](https://github.com/block/goose/pull/8815) | chore: replace lazy_static with std::sync::LazyLock | [2026-W17/drip-86/block-goose-pr-8815.md](2026-W17/drip-86/block-goose-pr-8815.md) |
| [#8812](https://github.com/block/goose/pull/8812) | chore: pin Swatinem/rust-cache to v2.9.1 SHA | [2026-W17/drip-86/block-goose-pr-8812.md](2026-W17/drip-86/block-goose-pr-8812.md) |
| [#10334](https://github.com/cline/cline/pull/10334) | chore(deps): bump @xmldom/xmldom 0.8.11 → 0.8.13 | [2026-W17/drip-86/cline-cline-pr-10334.md](2026-W17/drip-86/cline-cline-pr-10334.md) |

Verdict mix: 5 merge-as-is (Aider-AI/aider#5009 — 27 well-chosen unit tests covering the four public functions in `aider/diffs.py` with boundary cases for `create_progress_bar` (0%, 50%, 100%, 33% to lock integer-floor rounding), edge cases for `assert_newlines` (empty list, single-line tolerance, raise-on-missing-newline), 7 cases for `find_last_non_deleted` including the off-by-one sensitive `test_updated_with_changes` at `:62-66`, and 11 cases for `diff_partial_update` covering streaming/final modes plus backtick escaping; pure functional tests with no fixture deps, runs in 30ms, idiomatic `unittest.TestCase` matching the surrounding `tests/basic/` convention; Aider-AI/aider#4968 — surgical one-line tightening of `DIVIDER` regex at `editblock_coder.py:387` from `^={5,9}\s*$` to `^={7}\s*$` plus a `.rst` heading regression test, fixes the silent edit-block corruption reported in #1803 where 8-equals heading underlines were misparsed as block dividers, breaking-surface concern minimal because every aider system prompt has asked for exactly 7 equals for years; block/goose#8815 — canonical std-library modernization replacing `lazy_static!` with `LazyLock` at two call sites (regex pattern map in `security/patterns.rs` and test fixture in `tests/providers.rs`), one-to-one semantic equivalence, removes one transitive dep, requires Rust 1.80+ which goose's stable channel comfortably exceeds; block/goose#8812 — supply-chain hygiene SHA-pinning of `Swatinem/rust-cache@v2.9.1` to immutable commit `c19371144df3bb44fab255c43d04cbc2ab54d1c4` across 8 workflow files closing the floating-tag mutation window that the September 2024 `tj-actions/changed-files` incident made canonical, zero functional change; cline/cline#10334 — dependabot security patch from 0.8.11 to 0.8.13 addressing three xmldom GHSAs (j759-j44w-7fr8 namespace injection, x6wf-f3px-wcqx DOM clobbering, f6ww-3ggp-fr8h XML parsing inconsistency) within the patch line so no API surface change, drop-in replacement requiring only CI-green as the gate), 3 merge-after-nits (sst/opencode#24518 — SSE bridge route at `routes/instance/httpapi/event.ts:1-62` correctly uses `Stream.callback` to bridge `Bus.subscribeAllCallback` into an Effect Queue with three lifecycle correctness wins (immediate `server.connected` push at `:30`, 10s heartbeat at `:33-35` defeating LB idle-disconnect, re-entrant-safe `stop()` guarded by `done` flag at `:18-25`) plus proper `Effect.addFinalizer` scope hookup at `:43` and SSE response framing with `text/event-stream` + `X-Accel-Buffering: no` + `X-Content-Type-Options: nosniff` at `:46-56`, two ordering nits worth fixing (`heartbeat` declared after `stop()` reads it via closure so synchronous `InstanceDisposed` would silently leak the timer, and `push("server.connected")` before `subscribeAllCallback` resolves leaves a tiny race where bus events between the two are dropped), plus ask for explicit test coverage of `server.connected`-arrives-first, fake-timer heartbeat firing, `InstanceDisposed` closing the stream, and abrupt-disconnect cleanup; sst/opencode#24512 — Effect-Schema `Event.define()` generic at `v2/event.ts:13-37` is the right shape with `Schema.Literal(input.type)` keeping discriminator narrowability, `Object.assign(Event, { Sync })` matching ecosystem conventions, and net `−368` lines is a healthy refactor direction, but the wire-type rename `prompt`→`session.prompted` etc. is a breaking change for any persisted v2 events with no version bump, migration alias, or PR-body acknowledgement (either confirm v2 isn't shipped/persisted yet, add a one-shot read-side alias, or bump `SCHEMA_VERSION`), plus the 401-vs-676-line FastCheck-to-deterministic test rewrite is a real property-coverage delta that warrants `bun test --coverage` spot-check against the prior baseline, plus consider `Schema.optionalWith(default: () => 1)` for `version` so consumers always have the value; Aider-AI/aider#4987 — `sanitize_for_utf8()` at `models.py:1070-1085` correctly fixes a real Windows-GBK-locale `UnicodeEncodeError` crash by recursively walking dict/list/str and round-tripping strings through `encode("utf-8", errors="replace").decode("utf-8")` to substitute U+FFFD for unpaired surrogates, called at the right place at `:1009` immediately before `litellm.completion()` consumes `kwargs["messages"]`, with four mirror tests covering surrogate replacement, nested dict-of-list-of-dict structures, non-ASCII preservation (Chinese U+4E16 U+754C explicitly tested), and primitive passthrough plus the load-bearing `json.dumps(result).encode("utf-8")` end-to-end assertion, four asks before merge: gate with `if sys.platform == "win32"` to skip the full deep walk for the 99% of users who never see the bug, add a `logger.warning("sanitize_for_utf8 replaced N surrogate characters")` so silent lossy substitution becomes diagnosable, handle `tuple` and `dict` keys (currently only values), and clarify whether `bytes` payloads should be touched at all)).

---
### W17 drip-87 (2026-04-27)

9 fresh PRs across 4 repos — Dockerfile parameterization at sst/opencode #24520 collapsing the prior two-stage `FROM base AS build-amd64`/`FROM build-${TARGETARCH}` indirection into a single-stage parametric build keyed on `ARG BASE_IMAGE` (default `alpine:3.21`) and `ARG DIST_SUFFIX` (default `-musl`) with runtime distro detection via `[ -f /etc/alpine-release ]` plus a `publish.ts` rewrite that calls a new `buildImage(tags, buildArgs)` helper twice for the alpine `:version`/`:channel`/`:latest` tags and the `:ubuntu` glibc tag, motivated by PyTorch ROCm not running on musl, with a silent `:latest` tag addition that's a behavior change beyond the PR title and a 2× sequential-push wall-clock cost; new optional `modelPricing` settings object at QwenLM/qwen-code #3631 driving cost estimation in `/stats model` with a 33-line `costCalculator.ts` and 205 lines of test verifying `prompt × inputPerMillionTokens + candidates × outputPerMillionTokens` arithmetic ($0.30 + $0.60 = $0.90 at the 1M+500K test point) but ignoring `cached` tokens (cache discount unaccounted) and `thoughts` tokens (reasoning-token billing missed), $-prefixed currency symbol hard-coded; methodical 337-line test addition at QwenLM/qwen-code #3629 closing a real OAuth-path config gap where `QWEN_CODE_API_TIMEOUT_MS` was silently ignored when authenticated via Qwen OAuth, with 6+ env-var scenarios including precedence-vs-modelProvider, non-numeric/negative/zero rejection, and `parseInt`-style float truncation (12345.67 → 12345); BYO-API-key interactive auth menu addition at QwenLM/qwen-code #3624 reordering the options to Coding Plan → API Key → discontinued OAuth across three docs files plus a +446/−108 handler.ts rewrite that's large enough to merit a careful side-by-side review for prompt-loop and storage-write duplication; macOS click-to-focus notification backend at charmbracelet/crush #2714 (+146 new file `native_darwin.go`) using `terminal-notifier` and a hardcoded TERM_PROGRAM→bundle-ID map covering 7 terminals plus tmux/screen sentinel empty-strings, with bundle-ID table going stale silently for Warp/Tabby/Hyper/Rio and unmaintained-upstream `terminal-notifier` deprecation; Anthropic-summary-truncation fix at charmbracelet/crush #2710 extracting the `DefaultMaxTokens || ModelCfg.MaxTokens` ladder into a new `Model.MaxOutputTokens()` method, deduplicating it from `Run`/`runSubAgent`/`Summarize` and threading the result as `*int64` (nil-when-zero to keep LM Studio happy) with 56 lines of unit test for a 22-line impl; MCP-stdio arg env-var expansion at charmbracelet/crush #2693 closing the symmetry gap where `command` and `env` already went through `resolver.ResolveValue` but `args` were passed verbatim, fallback-to-original on resolution error with `slog.Error` (probably should be Warn); SQLite-pool cap at charmbracelet/crush #2691 setting `db.SetMaxOpenConns(1)` with a 12-line block comment diagnosing `SQLITE_NOTADB (26)` corruption from interleaved WAL frames + auto-checkpoints under parallel sub-agent access — flagged for discussion because the diagnosis (multi-handle WAL writers desyncing) is unusual enough to warrant a reproducer or RCA before accepting that single-connection serialization is the right fix vs a 2-pool reader/writer split; and React-error-#31 fix at BerriAI/litellm #26515 widening `FallbackChainItem` to `string | { model?: string; [key: string]: any }` with secret-masking for `api_key`/`authorization`/`token`/`secret`/`password` keys plus 5 surgical regression tests including DOM-level masking assertion, marred by 3 unrelated Python formatting-only diffs that should be split.

| PR | Title | File |
| --- | --- | --- |
| [#24520](https://github.com/sst/opencode/pull/24520) | feat(docker) build ubuntu image alongside alpine | [2026-W17/drip-87/sst-opencode-pr-24520.md](2026-W17/drip-87/sst-opencode-pr-24520.md) |
| [#3631](https://github.com/QwenLM/qwen-code/pull/3631) | feat(stats): add model cost estimation | [2026-W17/drip-87/QwenLM-qwen-code-pr-3631.md](2026-W17/drip-87/QwenLM-qwen-code-pr-3631.md) |
| [#3629](https://github.com/QwenLM/qwen-code/pull/3629) | fix(config): support QWEN_CODE_API_TIMEOUT_MS across OAuth and non-OAuth paths | [2026-W17/drip-87/QwenLM-qwen-code-pr-3629.md](2026-W17/drip-87/QwenLM-qwen-code-pr-3629.md) |
| [#3624](https://github.com/QwenLM/qwen-code/pull/3624) | fix(cli): add API Key option to `qwen auth` interactive menu | [2026-W17/drip-87/QwenLM-qwen-code-pr-3624.md](2026-W17/drip-87/QwenLM-qwen-code-pr-3624.md) |
| [#2714](https://github.com/charmbracelet/crush/pull/2714) | update terminal notifier for macos | [2026-W17/drip-87/charmbracelet-crush-pr-2714.md](2026-W17/drip-87/charmbracelet-crush-pr-2714.md) |
| [#2710](https://github.com/charmbracelet/crush/pull/2710) | fix(agent): pass max output tokens to summary stream | [2026-W17/drip-87/charmbracelet-crush-pr-2710.md](2026-W17/drip-87/charmbracelet-crush-pr-2710.md) |
| [#2693](https://github.com/charmbracelet/crush/pull/2693) | fix(mcp): expand environment variables in stdio MCP server args | [2026-W17/drip-87/charmbracelet-crush-pr-2693.md](2026-W17/drip-87/charmbracelet-crush-pr-2693.md) |
| [#2691](https://github.com/charmbracelet/crush/pull/2691) | fix(db): cap SQLite pool to one writer to prevent NOTADB corruption | [2026-W17/drip-87/charmbracelet-crush-pr-2691.md](2026-W17/drip-87/charmbracelet-crush-pr-2691.md) |
| [#26515](https://github.com/BerriAI/litellm/pull/26515) | fix(ui): render dict-shaped fallback entries in router settings | [2026-W17/drip-87/BerriAI-litellm-pr-26515.md](2026-W17/drip-87/BerriAI-litellm-pr-26515.md) |

Verdict mix: 2 merge-as-is (QwenLM/qwen-code #3629, charmbracelet/crush #2710), 6 merge-after-nits (sst/opencode #24520, QwenLM/qwen-code #3631, QwenLM/qwen-code #3624, charmbracelet/crush #2714, charmbracelet/crush #2693, BerriAI/litellm #26515), 1 needs-discussion (charmbracelet/crush #2691 — diagnosis unusual enough to warrant RCA before accepting single-connection serialization vs a reader/writer pool split, especially given companion PR #2690 proposing a different fix for the same NOTADB symptom).

---

### W17 drip-88 (2026-04-27)

8 fresh PRs across 2 repos — at sst/opencode, the small-but-contaminated #24522 race fix where the actual `currentSessionID` snapshot + post-`await` re-comparison guard against stale `sdk.client.session.get()` mutations is correct and minimal but the diff also smuggles in (a) a brand-new `sync_prompt_context_on_session_switch` kv signal + command-palette toggle that belongs to PR #24508 and (b) a Monokai scrollbar palette swap that belongs to PR #24507, plus a 0-byte `packages/sdk/js/openapi.json` placeholder; the disciplined #24523 closes #23490 by replacing lexicographic `lastUser.id < lastAssistant.id` and `m.info.id <= lastFinished.id` predicates in the `SessionPrompt.run` back-scan with array-index comparisons (`lastUserIndex < lastAssistantIndex`, `for (let i = lastFinishedIndex + 1; i < msgs.length; i++)`), unconditionally correct under non-monotonic message IDs and a small perf win on the skip-loop, missing only a regression test exercising lexically-inverted IDs; the small #24524 swaps `void Promise.all([...])` for `return Promise.all([...]).then(...).catch(e => { Log.Default.error(...); setStore('status','complete') })` in the TUI background-sync chain so the ~12 parallel SDK rejections are observed and the status no longer sticks at `"partial"` indefinitely; the substantial #24525 attacks three independent OOM-amplification paths in one PR — `Bus` swaps `PubSub.unbounded` → `PubSub.sliding<Payload>(1024)` for the wildcard channel, both global and instance SSE routes get bounded `AsyncQueue<string|null>(256)` plus an overflow-then-`stop()` policy that closes the connection rather than silently dropping individual messages, the tool `metadata()` callback dedups via 100ms throttle + `JSON.stringify(input)` fingerprint (key-order fragile) and now preserves prior `title`/`metadata`/`time.start` when the new value is `undefined`, and the shell-streaming path spills to disk via `createWriteStream(outputPath, {flags:"a"})` once `bufferedOutput` exceeds `limits.maxBytes` while keeping a 30 KB sliding preview, with `Effect.ensuring(toolMetadata.delete(...))` cleanup that's correct but the `outputSink.end()` Promise wrapper doesn't `stream.destroy()` on error; the misleadingly-titled #24526 ships under "Simplify Infinity mode judge prompt" but is actually a +798/−1 initial Infinity-mode UX implementation including a new permission-deny-by-default `infinity` agent in `agent/agent.ts`, a polling `useQuery({queryKey:["infinity",params.id], refetchInterval: 10000})` client that `.catch(() => ({active:false}))` silently masks server errors, identical `"∞"` glyph for both toggle states, V2 server endpoints `infinitySet`/`infinityClear`/`infinityStatus`, and regenerated SDK gen files — needs splitting into the cleanup PR the title claims and a separate feature PR for the toggle UX. At charmbracelet/crush, #2690 is the better-shaped sibling of the already-reviewed #2691 NOTADB fix, combining `db.SetMaxOpenConns(1)` with `_txlock=immediate` across both modernc and ncruces drivers (the latter requires DSN-string param injection because the driver doesn't expose a Go-level config struct), addressing both the WAL-frame-interleaving corruption and the deferred→writer upgrade contention as separate failure modes; #2686 lands a clean Moonshot/Moonshot-China env-var resolver with `cmp.Or(MOONSHOT_API_KEY, KIMI_API_KEY)` precedence and 95 lines of test, but ships a `replace charm.land/catwalk => github.com/ne275/catwalk v0.0.0-20260423090449-...` directive pointing at a personal-account GitHub fork — supply-chain risk that needs to be split into "international half (no replace)" + "China half (after upstream Catwalk tag)"; and #2643 fixes two coupled reasoning-display bugs in the TUI by bypassing `getCachedRender` while `isSpinning()` (the cache was keyed only by width and captured the empty pre-stream content forever) and threading a new `showThinking bool` parameter through `ExtractMessageItems → NewAssistantMessageItem` plus the action handler for `ActionToggleThinkingVisibility`, with the concern that `refreshMessages()` re-runs `ListMessages` from the database on every toggle which forces a full chat-history re-layout instead of a lightweight in-memory rebuild.

| PR | Title | File |
| --- | --- | --- |
| [#24522](https://github.com/sst/opencode/pull/24522) | fix: guard workspace mutation against stale session effect | [2026-W17/drip-88/sst-opencode-pr-24522.md](2026-W17/drip-88/sst-opencode-pr-24522.md) |
| [#24523](https://github.com/sst/opencode/pull/24523) | fix(session): compare message positions instead of IDs in SessionPrompt.run | [2026-W17/drip-88/sst-opencode-pr-24523.md](2026-W17/drip-88/sst-opencode-pr-24523.md) |
| [#24524](https://github.com/sst/opencode/pull/24524) | fix(tui): handle background sync rejection in sync.tsx | [2026-W17/drip-88/sst-opencode-pr-24524.md](2026-W17/drip-88/sst-opencode-pr-24524.md) |
| [#24525](https://github.com/sst/opencode/pull/24525) | Mitigate memory spikes from event backlog and shell output | [2026-W17/drip-88/sst-opencode-pr-24525.md](2026-W17/drip-88/sst-opencode-pr-24525.md) |
| [#24526](https://github.com/sst/opencode/pull/24526) | Simplify Infinity mode judge prompt and reduce logging noise | [2026-W17/drip-88/sst-opencode-pr-24526.md](2026-W17/drip-88/sst-opencode-pr-24526.md) |
| [#2690](https://github.com/charmbracelet/crush/pull/2690) | fix(db): prevent SQLITE_NOTADB corruption under concurrent sub-agents | [2026-W17/drip-88/charmbracelet-crush-pr-2690.md](2026-W17/drip-88/charmbracelet-crush-pr-2690.md) |
| [#2686](https://github.com/charmbracelet/crush/pull/2686) | feat: support Moonshot and Moonshot China API keys in config | [2026-W17/drip-88/charmbracelet-crush-pr-2686.md](2026-W17/drip-88/charmbracelet-crush-pr-2686.md) |
| [#2643](https://github.com/charmbracelet/crush/pull/2643) | fix: enable real-time reasoning display and implement missing toggle handler | [2026-W17/drip-88/charmbracelet-crush-pr-2643.md](2026-W17/drip-88/charmbracelet-crush-pr-2643.md) |

Verdict mix: 1 merge-as-is (sst/opencode #24524), 4 merge-after-nits (sst/opencode #24523, sst/opencode #24525, charmbracelet/crush #2690, charmbracelet/crush #2643), 2 request-changes (sst/opencode #24522 — bundles unrelated PR-#24507/#24508 changes under a misleading title; charmbracelet/crush #2686 — `replace` directive pointing at a personal-account Catwalk fork is a supply-chain non-starter), 1 needs-discussion (sst/opencode #24526 — +798/−1 ships an entire Infinity-mode UX feature under a "Simplify prompt" title and needs splitting plus per-piece security review on the new deny-by-default agent).

---

### W17 drip-89 (2026-04-27)

8 fresh PRs across 3 repos — at sst/opencode, the new background-task sidebar plugin at #24532 (`feature-plugins/sidebar/tasks.tsx:14-43`) correctly models both "tool still running" and "tool completed but child session still busy" lifecycle states with a `runningTasks` memo, paired with a misleading `Locale.duration(0)` "0ms" → "In Progress" label fix at `routes/session/index.tsx:2014-2016`, but needs a one-line guard for empty-string `subagent_type` and reactivity confirmation on `state.session.status(childSessionId)`; #24528 restructures the TUI startup Promise from a swallow-everything async-executor into a proper `(resolve, reject)` shape with a `fail()` cleanup that destroys partial renderers, except the inner `void (async () => {...})()` IIFE is missing the `.catch(fail)` hookup that would actually wire rejections through, leaving the fix incomplete despite the correct skeleton; #24531 bumps `@opentui/core`/`solid` 0.1.103→0.1.104 (theme-mode feedback loop fixes) and silently *deletes* the local `util/terminal.ts` OSC 4/10/11 color-query helper without explaining whether 0.1.104 supersedes it or whether the call sites are gone — needs `rg "Terminal\.colors\("` audit at the head SHA. At charmbracelet/crush, #2675 bundles three unrelated fixes (empty-tool-call-input → `"{}"` normalization at `agent.go:618-624`, named-arg regex Unicode widening from `\$([A-Z][A-Z0-9_]*)` to `\$([\p{L}_][\p{L}\p{N}_]*)` at `commands.go:17`, plus `message/content.go` edits) where the regex change silently makes `$lowercase` start matching when it previously didn't — a soft breaking change for existing user command files that the PR body doesn't acknowledge; #2681 fixes the GPT-5-mini class hidden-reasoning empty-title bug by bumping `maxOutputTokens` from 40→256 and adding a second retry path at `agent.go:994-1009` that calls the large model when the small one returned cleanly with empty visible text (`!isUsableTitleResponse(resp)`), gated by a model-ID inequality check to avoid double-calling the same model, but ships no test for the new branch; #2652 is a textbook ctx-cancellation propagation fix threading `context.Context` through `searchFilesWithRegex`/`fileContainsPattern`/the inner `filepath.Walk` callback at eight boundaries in `internal/agent/tools/grep.go`, with the load-bearing check at `:181-183` between ripgrep failure and regex fallback (previously, ctx-cancellation during ripgrep manifested as a ripgrep error, so the fallback ran anyway burning CPU after the user already cancelled); #2647 is a 4-line off-by-`offsetLine` fix at `list/list.go:88` changing the `AtBottom()` early-exit guard from `totalHeight > l.height` to `totalHeight - l.offsetLine > l.height`, accounting for the lines of the topmost item that are scrolled out of view, restoring auto-scroll on long-message sessions. At BerriAI/litellm, #26425 splits `_handle_authentication_error()` into a "expected auth deny" branch (BudgetExceededError, ProxyException 401/403, HTTPException 401/403) that logs at WARNING without traceback and an "everything else" branch that keeps the original `exception()` ERROR + traceback — reduces routine access-control noise hitting Datadog/OTEL error budgets, but loses the alerting signal for *server-side* JWT-decode 401s that should still page on-call, and bundles 4 unrelated black/ruff formatting-only diffs in `factory.py`/`predibase/transformation.py`/`xecguard.py` that should be split out.

| PR | Title | File |
| --- | --- | --- |
| [#24532](https://github.com/sst/opencode/pull/24532) | feat(tui): show background task status in sidebar and fix 0ms duration label | [2026-W17/drip-89/sst-opencode-pr-24532.md](2026-W17/drip-89/sst-opencode-pr-24532.md) |
| [#24528](https://github.com/sst/opencode/pull/24528) | fix(tui): startup rejection handling | [2026-W17/drip-89/sst-opencode-pr-24528.md](2026-W17/drip-89/sst-opencode-pr-24528.md) |
| [#24531](https://github.com/sst/opencode/pull/24531) | upgrade opentui to 0.1.104 | [2026-W17/drip-89/sst-opencode-pr-24531.md](2026-W17/drip-89/sst-opencode-pr-24531.md) |
| [#2675](https://github.com/charmbracelet/crush/pull/2675) | Fix command aliases/args parsing and empty tool-call input normalization | [2026-W17/drip-89/charmbracelet-crush-pr-2675.md](2026-W17/drip-89/charmbracelet-crush-pr-2675.md) |
| [#2681](https://github.com/charmbracelet/crush/pull/2681) | fix(agent): retry title generation with large model on empty output | [2026-W17/drip-89/charmbracelet-crush-pr-2681.md](2026-W17/drip-89/charmbracelet-crush-pr-2681.md) |
| [#2652](https://github.com/charmbracelet/crush/pull/2652) | fix(grep): stop regex fallback after cancellation | [2026-W17/drip-89/charmbracelet-crush-pr-2652.md](2026-W17/drip-89/charmbracelet-crush-pr-2652.md) |
| [#2647](https://github.com/charmbracelet/crush/pull/2647) | fix(ui): fix AtBottom() early exit not accounting for offsetLine | [2026-W17/drip-89/charmbracelet-crush-pr-2647.md](2026-W17/drip-89/charmbracelet-crush-pr-2647.md) |
| [#26425](https://github.com/BerriAI/litellm/pull/26425) | fix(proxy): downgrade 401/403 auth denials from ERROR to WARNING in auth_exception_handler | [2026-W17/drip-89/BerriAI-litellm-pr-26425.md](2026-W17/drip-89/BerriAI-litellm-pr-26425.md) |

Verdict mix: 2 merge-as-is (charmbracelet/crush #2652 mechanical ctx-prop fix, charmbracelet/crush #2647 minimal `offsetLine` arithmetic correction), 6 merge-after-nits (sst/opencode #24532 sidebar plugin needs reactivity confirmation + empty-string guard, sst/opencode #24528 startup-rejection refactor missing the `.catch(fail)` that would actually wire it up, sst/opencode #24531 opentui bump silently deletes the OSC color-query helper, charmbracelet/crush #2675 bundles 3 unrelated fixes including a silent `$lowercase` regex-widening, charmbracelet/crush #2681 GPT-5-mini empty-title retry needs a regression test, BerriAI/litellm #26425 auth-log-downgrade loses signal on JWT-broken 401s and bundles 4 unrelated formatter diffs).

---

### W17 drip-90 (2026-04-27)

8 fresh PRs across 3 repos. At sst/opencode, #24535 introduces multi-root workspaces via a new `WorkspaceProvider` + `dialog-workspace-create.tsx` + Playwright e2e — the `once()` callback wrapper at `dialog-workspace-create.tsx:23-31` correctly prevents double-fire of the directory-picker reopen path, but it captures `currentName`/`currentFolders` at `addFolder()` invocation time so any keystrokes typed into the name field while the picker is open are silently dropped on reopen, plus the dedupe at `:56-60` only does `result.trim()` with no `path.resolve`/trailing-slash strip so `~/foo` and `~/foo/` create two entries pointing at the same dir, and `workspace.new` and `workspace.create` are both registered as common commands without explaining the relationship; #24520 parameterizes the Dockerfile with `BASE_IMAGE`/`DIST_SUFFIX` build-args to produce alpine and ubuntu variants from one recipe, with the alpine→`apk` vs ubuntu→`apt-get` dispatch keyed on `[ -f /etc/alpine-release ]` (more robust would be `command -v apk`) and silently bumping the alpine pin from floating `alpine` to `alpine:3.21` and adding `:latest` to the alpine publish set — both behavior changes that need callouts; #24477 centralizes `parts → PromptInfo` reconstruction in a new `routes/session/prompt-info.ts` factory and rewrites the producer side at `session/prompt.ts:1622-1655` to attach `metadata.command = { invocation, name, arguments }` to the first text part of a command invocation, with the consumer at `prompt-info.ts:24-32` doing a two-pass reduce that finds the command then drops free-form text (the `agg.input = command ?? ""` overrides redundantly with the inner `!command` guard — pick one), and the three dialog refactors gain a latent `?? []` crash-fix on missing parts. At charmbracelet/crush, #2721 is a 3-line fix to the rename-dialog input width arithmetic at `sessions_item.go:91-97` swapping `InfoTextFocused.GetHorizontalFrameSize()` (wrong sibling style — copy-paste bug) for `ItemFocused.GetHorizontalFrameSize()` plus a `cursorPadding=1` reservation and a `max(0, ...)` clamp; #2711 reworks the todo-spinner+pill-render trigger at `internal/ui/model/ui.go:582-596` so any session update unconditionally re-renders pills (fixing stale pills on todo add/complete/reorder/edit) and the spinner starts/stops on steady-state rather than rising-edge of `hasInProgressTodo`, but silently removes the `m.updateLayoutAndSize()` call that previously fired when the first in_progress todo arrived — if pills can change height (multi-row wrap on long todo text), neighboring panes will miscalculate until an unrelated layout event; #2699 threads `workspaceRoot` (the LSP client's `cwd`) through `HandleApplyEdit` → `ApplyWorkspaceEdit` → `applyTextEdits`/`applyDocumentChange` and validates every URI before write/create/rename/delete, with a Windows-specific `normalizeURIPath()` that strips the `\C:\...` / `/C:/...` leading-separator-before-drive-letter trap that otherwise makes `filepath.Abs` prefix the current drive yielding `D:\C:\...`, and uses `fsext.HasPrefix` for separator-aware boundary checking — symlink resolution is intentionally out of scope. At BerriAI/litellm, #26551 fixes a bare `return` inside an `async generator` in `tool_permission.py:683-690` that was silently swallowing all upstream chunks (clients received only `data: [DONE]`) by re-wrapping the assembled response in `MockResponseIterator` and yielding its chunks before returning, with a regression test that uses `assert len(chunks) >= 1` — too weak, would pass for "yielded one empty chunk" — and bundles unrelated black/ruff formatter diffs in `factory.py`/`predibase/transformation.py`/`xecguard.py` that should be split; #26550 fixes non-ASCII response-header truncation by switching `aiohttp_transport.py:343` from `response.headers` (CIMultiDict[str,str], already lossy-decoded by aiohttp) to `response.raw_headers` (the bytes-tuple form httpx accepts and decodes losslessly), and parallel-fixes the credential-leak masker at `http_handler.py:381-388` to iterate `headers.raw` and compare bytes literals (`b"content-encoding"`) — the two paths use different APIs (`raw_headers` for aiohttp client response, `.headers.raw` for httpx response) which deserves a one-line clarifying comment in each file, and bundles the same unrelated formatter diffs as #26551.

| PR | Title | File |
| --- | --- | --- |
| [#24535](https://github.com/sst/opencode/pull/24535) | feat: add multi-root workspaces | [2026-W17/drip-90/sst-opencode-pr-24535.md](2026-W17/drip-90/sst-opencode-pr-24535.md) |
| [#24520](https://github.com/sst/opencode/pull/24520) | feat(docker) build ubuntu image alongside alpine | [2026-W17/drip-90/sst-opencode-pr-24520.md](2026-W17/drip-90/sst-opencode-pr-24520.md) |
| [#24477](https://github.com/sst/opencode/pull/24477) | fix(tui): restore slash command text when jumping back to command messages | [2026-W17/drip-90/sst-opencode-pr-24477.md](2026-W17/drip-90/sst-opencode-pr-24477.md) |
| [#2721](https://github.com/charmbracelet/crush/pull/2721) | fix(ui): fix dialog box shift when session rename is active | [2026-W17/drip-90/charmbracelet-crush-pr-2721.md](2026-W17/drip-90/charmbracelet-crush-pr-2721.md) |
| [#2711](https://github.com/charmbracelet/crush/pull/2711) | fix(ui): refresh todo pills on every session update | [2026-W17/drip-90/charmbracelet-crush-pr-2711.md](2026-W17/drip-90/charmbracelet-crush-pr-2711.md) |
| [#2699](https://github.com/charmbracelet/crush/pull/2699) | fix(lsp): enforce workspace boundary for workspace edits | [2026-W17/drip-90/charmbracelet-crush-pr-2699.md](2026-W17/drip-90/charmbracelet-crush-pr-2699.md) |
| [#26551](https://github.com/BerriAI/litellm/pull/26551) | fix(guardrails): re-emit chunks in tool_permission streaming hook when no tool_calls found | [2026-W17/drip-90/BerriAI-litellm-pr-26551.md](2026-W17/drip-90/BerriAI-litellm-pr-26551.md) |
| [#26550](https://github.com/BerriAI/litellm/pull/26550) | fix(custom_httpx): preserve non-ascii response headers | [2026-W17/drip-90/BerriAI-litellm-pr-26550.md](2026-W17/drip-90/BerriAI-litellm-pr-26550.md) |

Verdict mix: 2 merge-as-is (charmbracelet/crush #2721 maintainer 3-line rename-dialog width arithmetic fix, charmbracelet/crush #2699 LSP workspace-boundary enforcement with Windows path-normalization), 5 merge-after-nits (sst/opencode #24520 dockerfile parameterization with silent alpine pin + `:latest` move, sst/opencode #24477 prompt-info factory needs zod validator + unit test, charmbracelet/crush #2711 stale-pill fix silently drops `updateLayoutAndSize()`, BerriAI/litellm #26551 streaming-iterator bare-return fix bundles unrelated formatter diffs, BerriAI/litellm #26550 raw-bytes header pass-through bundles same unrelated formatter diffs), 1 needs-discussion (sst/opencode #24535 multi-root workspaces ships UI surface but doesn't address what "multi-root" semantically means for ServerProvider/ModelsProvider, plus dedupe-without-normalization).

---

