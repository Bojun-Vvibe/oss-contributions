# openai/codex PR #19763 — refactor: load agent identity runtime eagerly

- URL: https://github.com/openai/codex/pull/19763
- Head SHA: `c21a9665093faf4820d88a677f98733b9c97213e`
- Diff: +47 / -188 across 6 files
- Author: openai member (efrazer-oai)
- Stack: PR 2 of 3 (depends on #19762 "make auth loading async";
  followed by #19764 "verify AgentIdentity JWTs with JWKS")

## Context / Problem

`AgentIdentityAuth` previously deferred its `register_agent_task` call
behind an `Arc<OnceCell<String>>`. The auth value could exist before its
task binding was known, so every model-provider call site that wanted
the task id had to deal with `Option<&str>` and call `ensure_runtime()`
defensively. That makes the call-path harder to reason about and makes a
"cloned auth before runtime registered" race observable to downstream
code rather than rejected at construction.

## Design

`codex-rs/login/src/auth/agent_identity.rs` is rewritten to make
runtime allocation part of construction:

```rust
// before
#[derive(Debug)]
pub struct AgentIdentityAuth {
    record: AgentIdentityAuthRecord,
    process_task_id: Arc<OnceCell<String>>,
}
// custom Clone impl (Arc-shared OnceCell)
impl AgentIdentityAuth {
    pub fn new(record: ...) -> Self { ... }
    pub fn process_task_id(&self) -> Option<&str> { ... }
    pub async fn ensure_runtime(&self) -> std::io::Result<()> { ... }
}

// after
#[derive(Clone, Debug)]
pub struct AgentIdentityAuth {
    record: AgentIdentityAuthRecord,
    process_task_id: String,
}
impl AgentIdentityAuth {
    pub async fn load(record: ...) -> std::io::Result<Self> {
        let process_task_id = register_agent_task(
            &build_reqwest_client(),
            &agent_identity_authapi_base_url(),
            key(&record),
        ).await.map_err(std::io::Error::other)?;
        Ok(Self { record, process_task_id })
    }
    pub fn process_task_id(&self) -> &str { &self.process_task_id }
}
```

Net effect:

- `Clone` is now `derive(Clone)` because the custom `Arc`-sharing impl
  is gone — the field is plain `String`.
- `process_task_id()` returns `&str` not `Option<&str>`, so call sites
  no longer have to handle the lazy-not-yet-registered case. (See the
  knock-on simplification in the surrounding files; the diff is
  +47/-188 because the `OnceCell` plumbing was non-trivial.)
- `ensure_runtime()` is gone. The contract is now "if you have an
  `AgentIdentityAuth`, runtime is registered."
- `from_agent_identity_jwt(jwt)` flips `fn → async fn` at
  `codex-rs/login/src/auth/manager.rs:249-252` and is awaited at the
  one previously-sync call site (`manager.rs:215`). This is the only
  reason the prior "make auth loading async" PR (#19762) was needed.
- Helper extraction: the per-instance `key()` method becomes a free
  `fn key(record: &Record) -> AgentIdentityKey<'_>` and the base URL
  becomes a `fn agent_identity_authapi_base_url() -> String` so it
  can later be mocked at one site (probably what #19764 will need
  for JWKS testing).

Drive-by in `codex-rs/core/src/connectors.rs:148-155` and `:220-227`:
the connector-list gate flips from
`is_some_and(CodexAuth::is_chatgpt_auth)` →
`is_some_and(CodexAuth::uses_codex_backend)`. That widens the gate to
include AgentIdentity-backed auth (which is the whole point of the
stack — agent-identity should see the same connector surface that
ChatGPT auth sees). Worth confirming in review that `uses_codex_backend`
is true for both ChatGPT and AgentIdentity and false for Codex-API-key,
which is the implicit contract.

Test deletion at `codex-rs/login/src/auth/auth_tests.rs:625-651` removes
`load_auth_reads_agent_identity_from_env`. That test relied on
`AgentIdentityAuth::new` being sync and constructible from a fake JWT
without a working network — the new `load()` requires a successful
`register_agent_task` call which can't work against a fake URL. The
deletion is *correct in motivation* (the test was a sync-only
construction smoke that doesn't exist anymore) but leaves a behavioral
gap: nothing now pins "loading auth from `CODEX_AGENT_IDENTITY_ENV_VAR`
produces an `AgentIdentity` variant when the env var is well-formed."
A replacement test using a wiremock-stubbed
`agent_identity_authapi_base_url()` would close the gap and is the
obvious thing the new free function was extracted for. The
`agent_identity_record` fixture was updated at `auth_tests.rs:781-786`
to use real `generate_agent_key_material()` PKCS#8, which is the right
preparation for #19764's JWKS verification.

## Risks / nits

1. **Network call moved into the auth-loading hot path.** Anywhere
   `from_agent_identity_jwt` is invoked now does a network round-trip
   to `auth.openai.com/api/accounts` before returning. Previously this
   was deferred to first-use via `ensure_runtime()`. Behavioral
   consequences: (a) cold-start auth-load latency increases by one
   round-trip; (b) startup paths that construct auth speculatively
   will now fail loudly on transient registration failures instead of
   waiting for a real model call. Both are arguably correct ("if you
   can't register, you can't use this auth"), but the PR body doesn't
   mention either. Worth a one-line note in the changelog.
2. **No replacement for the deleted env-var test.** The wiremock-able
   `agent_identity_authapi_base_url()` free function is the obvious
   seam. Add it now or it will quietly stay missing.
3. **`uses_codex_backend` widening is a behavior change** in the
   connector-list gate, not a refactor. The PR title is "refactor"
   and the PR body doesn't mention `connectors.rs` at all. Minor
   accuracy nit — split this into its own PR or call it out
   explicitly in the body so reviewers don't miss it.
4. **`build_reqwest_client()` is constructed fresh on every `load()`
   call.** That's fine for a one-shot construction but if `load()`
   ends up called per-session, the connection-pool cost could matter.
   Worth confirming this is a once-per-process call.

## Verdict

`merge-after-nits` — the simplification is real (Option<&str> → &str,
custom-Clone → derive-Clone, deleted ensure_runtime callers all over),
the stack-of-three sequencing is correct (async first, then eager
runtime, then JWKS), and the wiremock seam is in the right place. The
deleted env-var test, the cold-start-latency note, and the
"connectors gate widened" call-out should land before merge.

## What I learned

When a struct's invariant is "this field is always populated after
construction," `Arc<OnceCell<T>>` is a code smell — the type is
saying "maybe populated, maybe not, and clones share state." Either
the invariant is real (then make construction async and store `T`
directly, as this PR does) or it isn't (then the field probably
belongs in a separate registration-state object, not the auth struct).
The eager-load shape forces every potential failure mode to surface
at one well-defined call site.
