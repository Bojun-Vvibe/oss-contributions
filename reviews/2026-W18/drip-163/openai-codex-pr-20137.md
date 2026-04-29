# openai/codex#20137 — Route tools through selected environments

- URL: https://github.com/openai/codex/pull/20137
- Head SHA: `d14c3ed67681f5c580023990ece0e8818d361869`
- Size: +2367 / −486, 78 files
- Verdict: **needs-discussion**

## Summary

Threads an optional `environmentId` through the v2 app-server protocol surface — `ListMcpServerStatusParams`, `McpResourceReadParams`, `SkillsListParams`/`SkillsListEntry`, plus the runtime carriers (`fs_api.rs`, `core/src/session/{turn,mcp,handlers}.rs`, `core-skills/src/{injection,loader,manager,render}.rs`, `core/src/apply_patch.rs`, `core/src/context/environment_context.rs`). Filesystem APIs gain directory-entry metadata. Skills now emit URIs under a new `oai_env://<environment_id>/<absolute_path>` scheme so the model can disambiguate same-named files across environments.

## Specific issues flagged

### 1. `oai_env_uri` / `parse_oai_env_uri` duplicated across two crates

Two near-identical definitions ship side-by-side:

- `core-skills/src/injection.rs:828-840` defines `parse_oai_env_uri`, `oai_env_uri`.
- `core-skills/src/render.rs:1199-1202` (per the diff hunks at lines 1153/1193/1199 in the unified diff) defines another `oai_env_uri` with the same body (`format!("oai_env://{environment_id}/{path}")`).

These need to live in one place — `codex-protocol` or a new `oai_env_uri.rs` module — otherwise a future scheme change (path-encoding, query-param suffix for revision pin, etc.) drifts in one site and not the other. The bug class is silent: both produce valid URIs that the parser at `:828` accepts, so divergence won't fail tests, it'll just produce different URIs from different code paths.

### 2. New `environmentId` field is `Option<String>` everywhere — no validation domain

The schema additions (e.g. `v2/ListMcpServerStatusParams.json:28-35`, `v2/McpResourceReadParams.json:1-8`, `v2/SkillsListParams.json` per diff lines 96-103, 142-149) all type it as `["string", "null"]` with no pattern/length constraint. Server-side validation at `core/src/session/handlers.rs` (per `invalid_request(format!("unknown environmentId `{environment_id}`"))` at diff lines 458/506/544) is the only gate, meaning a 1MiB attacker-supplied string flows through deserialization, gets logged via `tracing` if any debug is on, and only then gets rejected. Add `maxLength: 256` (or whatever the legitimate ID length cap is) to the schemas, and consider a `pattern: "^[A-Za-z0-9_-]+$"` so `oai_env://<environment_id>/...` round-trips safely without URL-encoding surprises in the parser at `:828-836`.

### 3. `parse_oai_env_uri` does not appear to URL-decode the path

`fn parse_oai_env_uri(uri: &str) -> Option<(&str, &str)>` at `:828` strips `oai_env://` and splits on `/`. But `oai_env_uri(environment_id, path)` at `:837` builds `format!("oai_env://{environment_id}/{path}")` where `path` is `AbsolutePathBuf`'s `Display` impl. If `path` ever contains a literal `?`, `#`, or `%`, the round-trip through a URI-aware consumer (an LLM that treats this as a real URI) breaks. Either commit to "this is a URI shaped like a URI but not RFC-3986" (and document that loudly at the helper sites) or actually use `percent_encoding`. The current half-measure is a footgun.

### 4. `forceReload` semantics with `environmentId`

`SkillsListParams` now accepts both `forceReload: bool` and `environmentId: Option<String>`. The PR description says "When omitted, uses the default environment and preserves legacy local fallback behavior." But what does `forceReload=true` mean when `environmentId="remote"`? Does it force re-scan of just the remote environment's skill cache, or does it invalidate local + remote? The schema description at diff lines 3214-3221 only says "scan" not "scan & invalidate cache" — needs a doc-comment in `core-skills/src/manager.rs` pinning this.

### 5. 78 files is huge for an `Option<String>` plumbing PR — split for review?

This is a foundational protocol change touching every tool surface. Even with no bugs, reviewer fatigue at 78 files means real bugs hide. Suggest splitting into: (1) protocol+schema + `oai_env_uri` helper crate, (2) skills surface adoption, (3) MCP surface adoption, (4) FS+apply_patch surface adoption, each independently mergeable. The current shape forces all-or-nothing review.

## Why needs-discussion not merge-after-nits

Items 1 (duplicated helper across crates) and 5 (PR shape) are architectural concerns the maintainer should weigh in on before nits get resolved. If split happens, items 2/3/4 fold into the protocol-only PR.
