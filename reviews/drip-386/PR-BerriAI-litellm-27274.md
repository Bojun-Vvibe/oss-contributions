# BerriAI/litellm#27274 — proxy: hot-reload config YAML when --reload is set

- **Head SHA**: `d9cc076d97683251e6c4eb068e259ace418e562e`
- **Stats**: +66 / -2, 2 files

## Summary

When the proxy is started with `--reload --config some.yaml`, uvicorn's autoreload only watches `*.py` files in the CWD by default — so editing the config YAML does *not* trigger a restart, contrary to dev-time intuition. This PR teaches `_get_default_unvicorn_init_args`'s reload path to extend uvicorn's `reload_dirs` and `reload_includes` to cover the config file's directory and basename, so saving the YAML now restarts the worker. Help text is updated to document the new behavior.

## Specific citations

- `litellm/proxy/proxy_cli.py:170-201`: new static helper `ProxyInitializationHelpers._get_reload_options(config_path: Optional[str]) -> dict`. The `if not config_path` early-return at line 178 returns `{"reload": True}` — preserving today's behavior when no config is passed (e.g., model_list-via-env scenarios). When a config path is set, the helper resolves it absolute, splits dir/basename, builds `reload_dirs = [cwd]` plus `[config_dir]` if it differs from cwd, and `reload_includes = ["*.py", basename_only]`. The `*.py` retention at line 195 is necessary because uvicorn's reload-pattern resolution *replaces* the default include list when you pass any `reload_includes`, it doesn't append. Forgetting that would silently break Python autoreload — the test at `tests/test_litellm/proxy/test_proxy_cli.py:147` and `:163` pins both elements.
- `litellm/proxy/proxy_cli.py:191-194` doc comment correctly calls out the absolute-vs-relative-pattern uvicorn footgun (linked to "uvicorn discussion #2156"): `pathlib.Path.glob()` raises `NotImplementedError` on absolute patterns starting with `/`. Using just `os.path.basename(config_abs)` as the include pattern, with the directory added to `reload_dirs`, is the right composition. This is one of those "the obvious thing crashes" cases worth the four-line comment.
- `litellm/proxy/proxy_cli.py:651`: `--reload` help string adds "Also reloads when the --config YAML file changes." Keeps the discoverability promise.
- `litellm/proxy/proxy_cli.py:1059-1062`: the call site flips from a one-line `uvicorn_args["reload"] = True` to `uvicorn_args.update(ProxyInitializationHelpers._get_reload_options(config))`. `update()` (vs. assignment) is correct — preserves any `reload_*` keys earlier code may have set.
- `tests/test_litellm/proxy/test_proxy_cli.py:136-167`: three new pytest cases. (a) `_no_config` returns `{"reload": True}` only — pinned to ensure no `reload_dirs`/`reload_includes` keys leak through and accidentally narrow uvicorn's default watch set. (b) `_with_config_in_cwd` fakes `tmp_path/config.yaml` and asserts `reload_dirs == [str(tmp_path)]` (no duplicate config_dir entry, since equal to cwd). (c) `_with_config_outside_cwd` separates `cwd_dir` and `elsewhere`, asserts both are in `reload_dirs` and that `reload_includes` is `["*.py", "proxy.yaml"]` — basename only. The third test's docstring at line 161 redundantly re-explains the absolute-vs-relative pattern footgun; one comment in source is enough.

## Verdict

**merge-as-is**

## Rationale

Tightly scoped, well-tested, defensible-by-default (no behavior change when no `--config` is set), and the obvious dev-loop quality-of-life win. The author noticed the load-bearing uvicorn footgun (absolute-pattern `NotImplementedError`) and documented it both in a code comment and a test docstring, which is exactly the right level of paranoia for a reload-hooks change. Three optional follow-ups, none worth blocking on: (1) adding `*.yaml` and `*.yml` as include patterns would let users edit *any* yaml — not just the explicit `--config` one — and reload, but that's a behavior expansion, not a correctness fix; (2) on Windows, `os.path.dirname` of a relative `config.yaml` returns `''`; the `if config_dir and config_dir != cwd` guard at line 184 short-circuits that case correctly, but a Windows-relative-config test would pin it; (3) `os.path.abspath` resolves symlinks differently than `os.path.realpath` — if users keep their config behind a symlink, watchfiles may miss the underlying-file write; not a regression vs. status-quo. Ship it.
