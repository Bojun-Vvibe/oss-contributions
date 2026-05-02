# block/goose #8960 — chore(deps): bump the cargo-minor-and-patch group with 9 updates

- **Head SHA:** `9aa6b43229826db1a643a21d168f448974667b0e`
- **Files:** `Cargo.lock`, `Cargo.toml`, `crates/goose-cli/Cargo.toml`, `crates/goose/Cargo.toml` (+120/-119)
- **Verdict:** `merge-after-nits`

## Rationale

Dependabot group bump that pulls 9 minor/patch updates. The non-trivial movers visible in `Cargo.lock` are AWS SDK family upgrades — `aws-config 1.8.12 → 1.8.13`, `aws-runtime 1.5.17 → 1.6.0`, `aws-sdk-bedrockruntime 1.120.0 → 1.124.0`, `aws-sdk-sagemakerruntime 1.93.0 → 1.95.0` — and these are the ones to scrutinize because the diff shows `aws-runtime` flipping its dependency from `http 0.2.12` / `http-body 0.4.6` to `http 1.4.0` / `http-body 1.0.1`, and `aws-sdk-bedrockruntime` now pulls in *both* `http 0.2.12` and `http 1.4.0` plus `http-body-util` while dropping `hyper 0.14.32`. Mixed `http` major versions in a single binary is a known source of hard-to-diagnose trait-mismatch errors when a crate exposes `http::Request` in its public API and a dependent crate expects the other version.

Nits: (1) confirm `cargo tree -d` is still empty for the `http` and `hyper` crates after the bump, or document the duplication is intentional (the AWS SDK is in transition to `http 1.x`); (2) run the bedrock and sagemaker integration tests against a live endpoint at least once before merge — minor bumps inside `aws-sdk-bedrockruntime` 1.120→1.124 historically include subtle changes to streaming response handling on Converse; (3) the PR title says "9 updates" but the diff snippet only shows AWS-family movement — list the other 5 in the PR body so reviewers don't have to diff `Cargo.lock` themselves; (4) `aws-smithy-observability` is a *new* transitive dep on `aws-sdk-bedrockruntime` — verify it doesn't introduce a default tracing subscriber that conflicts with goose's own `tracing` setup.

Risk is medium because the AWS SDK churn touches the wire protocol layer for Bedrock — goose's Bedrock provider is in active use. Merge after a green Bedrock smoke test and a duplicate-deps audit.
