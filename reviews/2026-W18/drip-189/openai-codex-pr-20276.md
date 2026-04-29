# openai/codex#20276 — feat(app-server): add remote control enrollment read

- PR: https://github.com/openai/codex/pull/20276
- HEAD: `191dc00a71a7902ea81806bfb781aa9998d7e23a`
- Author: apanasenko-oai
- Files changed: 14 (+218 / -5) — `app-server-protocol/src/protocol/{common,v2}.rs`,
  `app-server/src/{lib,in_process,message_processor,codex_message_processor}.rs`,
  `app-server/src/transport/{mod,remote_control/mod}.rs`, README, snapshot.

## Summary

Adds a stable `remoteControl/enrollment/read` v2 request to the app-server
JSON-RPC surface so external clients can discover the persisted
remote-control enrollment identity for the initialized client. The existing
`remoteControl/status/changed` notification only carried connection state +
an `environmentId` snapshot; this exposes the durable `serverId` that
clients need for enrollment matching/deduping. The response is
`{enrollment: null}` when no current-account/client enrollment exists, or
`{enrollment: {serverId, environmentId}}` when one does — explicitly **not**
returning the bearer token or any other enrollment metadata. The PR also
threads a new `Option<Arc<StateRuntime>>` through `MessageProcessorArgs`
construction so the read can resolve against the local state DB.

## Cited hunks

- `codex-rs/app-server-protocol/src/protocol/common.rs:754-758` — new
  `RemoteControlEnrollmentRead` arm in `client_request_definitions!` with
  `serialization: None` (so each client connection gets its own queue
  rather than serializing globally — correct for a read, would be wrong
  for a write).
- `codex-rs/app-server-protocol/src/protocol/v2.rs:2949-2968` — three new
  types: empty `RemoteControlEnrollmentReadParams`,
  `RemoteControlEnrollmentReadResponse { enrollment: Option<...> }`, and
  the `RemoteControlEnrollment { server_id, environment_id }` value
  object. The response wrapper-with-Option shape matches the README's
  contract: `null` for "no enrollment" rather than an HTTP-error-like
  signal.
- `codex-rs/app-server-protocol/src/protocol/common.rs:2396-2412` — wire
  serialization test asserts the on-the-wire shape is exactly
  `{"method":"remoteControl/enrollment/read","id":7,"params":{}}`. Locks
  the contract against accidental method-name drift and against future
  additions to `Params` that would break empty-object serialization.
- `codex-rs/app-server/README.md:208` — README entry explicitly names the
  redaction policy: "the enrollment contains only `serverId` and
  `environmentId`". Good — the next reviewer of a "let's also return
  the token" follow-up has a documented contract to push back against.
- `codex-rs/app-server/src/codex_message_processor.rs:1313-1317` — adds a
  `warn!` arm for `ClientRequest::RemoteControlEnrollmentRead { .. }`
  parallel to the existing `ModelProviderCapabilitiesRead` warning.
  Correctly classifies this as "this request shouldn't reach here" —
  enrollment-read is dispatched by the outer `MessageProcessor`, not the
  per-Codex inner one.
- `codex-rs/app-server/src/in_process.rs:402-406` — `StateRuntime::init`
  is `.ok()`'d so a state-DB init failure doesn't kill the in-process
  client. Defensible — read-the-enrollment with `state_db: None` falls
  through to `enrollment: null` which is the same as "no enrollment", but
  this conflates "I have no enrollment" with "I can't tell you because my
  DB is broken" and operators can't distinguish.

## Risks

- The conflation noted at `:402-406` between "no enrollment" and "DB
  failed to init" is the main correctness concern. A misconfigured
  `sqlite_home` would silently look like a missing enrollment to every
  client, and the only diagnostic is the swallowed init error. Consider
  surfacing a `state_db_unavailable` flag or an explicit error variant
  in the response when state init failed; clients that care about the
  distinction can branch on it.
- The new `state_db: Option<Arc<StateRuntime>>` field threads through
  every `MessageProcessorArgs` construction site. Diff shows
  `in_process.rs` and `lib.rs:752` updated; need to confirm there are no
  third construction sites (e.g. integration-test fixtures) that would
  now fail to compile or default the field to `None` and silently break
  enrollment-read in tests.
- `Option<RemoteControlEnrollment>` at the response level is the right
  shape, but the spec doesn't say what happens if a client calls this
  before `initialize` completes. Likely the request just returns
  `enrollment: null` because state hasn't loaded yet, but the README
  should name this contract or the client could race the init flow.

## Verdict

**merge-after-nits**

## Recommendation

Land after (1) adding a smoke test that asserts the
"DB-init-failed → enrollment: null" distinction, (2) adding a one-line
README clarification that calling before `initialize` returns null, and
(3) verifying no other `MessageProcessorArgs` construction sites silently
default `state_db: None`. The redaction discipline (only `serverId` +
`environmentId` exposed) and the wire-shape lock test are the right
choices for a remote-control surface.
