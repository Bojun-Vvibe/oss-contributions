# block/goose#8990 — chore(deps): bump cargo-minor-and-patch group across 1 directory with 10 updates

- PR ref: `block/goose#8990`
- Head SHA: `cb30b83cbaf178a6dd1583f74cc40a0a97f85eb2`
- Title: chore(deps): bump the cargo-minor-and-patch group across 1 directory with 10 updates
- Verdict: **needs-discussion**

## Review

Mostly routine Dependabot-style minor/patch bumps in `Cargo.lock` (~252 changed
lines) plus the matching `Cargo.toml`, `crates/goose/Cargo.toml`, and
`crates/goose-cli/Cargo.toml` constraint updates. The patch-level bumps to
`anstyle-query 1.1.5`, `aws-sigv4 1.3.7→1.3.8`, `aws-runtime 1.5.17→1.6.0` etc. are
the kind of thing that should land without ceremony.

However, this is **not** a clean minor/patch group — there are at least three
things in this diff that need a closer look before approving as a no-thought bump:

1. The AWS SDK runtime jumps from `1.5.17` to `1.6.0` (`Cargo.lock:432-434`), which
   pulls in a new `http 1.4.0` dependency *alongside* the existing `http 0.2.12`
   in `aws-sdk-bedrockruntime` (`Cargo.lock:476-477`), `aws-sdk-sagemakerruntime`,
   `aws-sdk-sso`, `aws-sdk-ssooidc`, and `aws-sdk-sts`. Carrying two major versions
   of `http` in the same binary is a known footgun (incompatible `HeaderName` /
   `HeaderValue` types across versions) — confirm the AWS SDK changelog says this is
   intentional during the 0.x→1.x http transition and that nothing in goose's own
   code passes `http::Header*` types between SDK calls.
2. `windows-sys` bumps from `0.60.2` to `0.61.2` (`Cargo.lock:143`, `:154`). This
   transitively forces every dep that depends on `windows-sys` onto the new version
   on the next resolve; worth running a Windows CI smoke check (which I can't see
   the result of from the diff alone) before merging.
3. `aws-smithy-observability` is newly pulled in across all the SDK crates
   (`Cargo.lock:469`, `:495`, etc.). That's a new dependency surface — check it
   doesn't ship with default features that emit telemetry to anywhere unexpected.

Recommend a maintainer comment confirming (a) the dual `http` versions are the
expected transient state, (b) Windows CI passed, and (c) the new
`aws-smithy-observability` defaults are acceptable. Otherwise the bumps themselves
are fine.
