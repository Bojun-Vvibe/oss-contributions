# block/goose #8844 — fix: convert quoted numeric config values to numbers if needed

- **Repo**: block/goose
- **PR**: #8844
- **Author**: philipphenkel (Philipp Henkel)
- **Head SHA**: 1a18189ff8cfd1651a8062e67b29a836ff65042b
- **Link**: https://github.com/block/goose/pull/8844
- **Size**: 13 production lines + 48 test lines in
  `crates/goose/src/config/base.rs`.

## What it changes

`Config::get_param` (`base.rs:732-748`) gains a fall-back path
when `serde_yaml::from_value(...)` fails: if the failed value is
a YAML string, retry by routing it through
`Self::parse_env_value(...)` (the existing env-var coercer that
turns `"300"` into a number, `"true"` into a bool, etc.) and
then `serde_json::from_value` to land in the requested type
`T`. If *that* fails, return the *original* yaml error (so
callers see "expected u64, got string" not the secondary JSON
error).

Concrete bug: a `config.yaml` written by the goose CLI/UI with
an entry like `OLLAMA_TIMEOUT: '1200'` (quoted on disk) was
ignored by provider code that called
`get_param::<u64>("OLLAMA_TIMEOUT")` because YAML deserialises
`'1200'` as `String`, not `u64`. After this fix, the second
branch parses the string into a JSON number and then into `u64`.

Tests at `:1167-1216`:
- `test_get_param_reads_numeric_yaml_as_u64` — unquoted numeric
  reads as `u64` (regression baseline).
- `test_get_param_reads_quoted_numeric_yaml_as_u64` — the
  motivating bug fix.
- `test_get_param_reads_quoted_numeric_yaml_as_string` — quoted
  numeric still reads as `String` if asked for one (no
  regression on string consumers).
- `test_get_param_rejects_invalid_string_as_u64` — `"invalid"`
  asked-as-u64 returns `DeserializeError` and not a panic.

## Strengths

- **The four-test matrix is exactly right.** It covers (a) the
  baseline that didn't break, (b) the bug fix, (c) the
  no-regression-on-other-shape, and (d) the failure path. This
  is the rare PR where the test set matches the spec set.
- **Returns the *original* yaml error on the failure path**
  (`:746` — `.map_err(|_| yaml_err.into())`). Without this, a
  user with `OLLAMA_TIMEOUT: 'foo'` would see something
  confusing like "JSON deserialization failed: invalid digit"
  instead of the canonical "expected u64, got string". Small
  detail, real UX win.
- **Routes through the *existing* `parse_env_value` helper** so
  the coercion semantics (which env-var consumers already
  depend on) and the YAML-to-typed coercion semantics stay in
  sync. If `parse_env_value` is ever extended to handle
  `"1.5h"` → `5400`, both paths get it for free.
- **`let-else` pattern at `:739-741` cleanly handles the
  "value is not a string" case** by returning the original yaml
  error, which is the right policy: if it's not a string, the
  fallback can't help, so don't pretend.
- **Issue trail referenced** (`#8437`, the follow-up reporter's
  comment) gives reviewers context.

## Concerns / asks

- **The string→JSON→T coercion is potentially over-broad.**
  `parse_env_value("true")` returns a JSON bool, so
  `get_param::<bool>("FOO")` where `FOO: 'true'` now succeeds
  even if the original config-write convention was strings-only.
  That's almost certainly the right call (it's symmetric with
  env-var loading), but it does subtly widen the contract:
  before this PR, only env-var-supplied bools were coerced;
  after, on-disk quoted bools coerce too. Worth a sentence in
  the docstring of `get_param` saying "string-shaped YAML
  values are coerced via `parse_env_value` if direct
  deserialisation to T fails".
- **No `let-else` style note about the discarded yaml error**:
  `Self::parse_env_value(string_value)?` at `:743` propagates
  with `?`, so a `parse_env_value` failure on a non-numeric
  string currently surfaces as the env-parser's error, not the
  yaml error. The test at `:1208-1216` confirms this returns
  `DeserializeError` — which is the right enum variant — but a
  reader of the source code could reasonably expect symmetry
  with the `serde_json::from_value(parsed).map_err(|_|
  yaml_err.into())` line just below. Either coerce both via
  `.map_err(|_| yaml_err.into())` or add a comment explaining
  the asymmetry.
- **The fix only handles strings**. YAML floats deserialised
  into integers (e.g. `OLLAMA_TIMEOUT: 1200.0` for `u64`) still
  fail. Probably out of scope, but worth a line in the PR body
  saying so.
- **`OPENAI_TIMEOUT`, `LITELLM_TIMEOUT`, `OLLAMA_TIMEOUT` are
  cited in the PR body** but no integration test ships that
  exercises a real provider read of one. The unit tests above
  use `XXX_TIMEOUT` — fine for the unit, but a smoke test that
  starts a provider with a quoted timeout would close the loop.
- **`env_lock::lock_env` usage** — each test acquires the env
  lock for `XXX_TIMEOUT`. If two tests with the same key ran
  in parallel without the lock, this would flake. The lock is
  the right pattern; just confirming the tests use a
  unique-enough key (`XXX_TIMEOUT` is fine as long as no other
  file in the crate uses it).

## Verdict

**merge-as-is** — small, well-justified, well-tested fix for a
real footgun (quoted YAML values silently ignored by typed
consumers). The asks are docstring-level and shouldn't gate
the merge.

## What I learned

The "config-on-disk shape disagreement with config-in-memory
type" bug is one of those quiet correctness regressions that
no schema validator catches because the file *is* valid YAML —
it just happens to be a string where the consumer wanted a
number. The right fix is exactly what this PR does: a typed
coercion fallback that routes through the same env-var
coercion path so the two ingestion surfaces stay
behaviorally aligned. The wrong fix would have been to make
the writer always emit unquoted numbers, because that breaks
roundtripping of any pre-existing `config.yaml` on disk.
