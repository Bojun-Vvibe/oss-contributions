# Review Index

181 + W17 drips (through drip-49) PR reviews across 10 OSS AI-coding-agent projects. Each review
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


See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.

## Aider-AI/aider

| PR | Title | File |
|---|---|---|
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

## BerriAI/litellm

| PR | Title | File |
|---|---|---|
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

## browser-use/browser-use

| PR | Title | File |
|---|---|---|
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

---

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.
