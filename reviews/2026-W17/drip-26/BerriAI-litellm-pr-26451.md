# BerriAI/litellm PR #26451 — Azure Sentinel truncation + gzip + batch splitting

- **Repo:** BerriAI/litellm
- **PR:** [#26451](https://github.com/BerriAI/litellm/pull/26451)
- **Head SHA:** `cb177053b70496635b1b7ad4fe07fe75f1194fdd`
- **Author:** alihacks
- **Size:** +504 / −77 across 4 files
- **Reviewer:** Bojun (drip-26)

## Summary

Three related improvements to the Azure Sentinel integration in
`litellm/integrations/azure_sentinel/azure_sentinel.py`:

1. **Always-on gzip compression** for the Logs Ingestion API request
   body (`Content-Encoding: gzip`), reducing payload by 5–10× and
   keeping more requests under Azure's 1 MB per-request limit.
2. **Always-on batch splitting** so a single flush that would
   exceed `MAX_BATCH_SIZE_BYTES = 950_000` is divided into multiple
   gzip-compressed POSTs instead of being dropped or truncated by
   the gateway.
3. **Opt-in column-level truncation** (gated by
   `AZURE_SENTINEL_TRUNCATE_CONTENT`) that pre-truncates `messages`
   and `response` to the Azure Log Analytics 256 KB column limit,
   keeping the *tail* (most recent content) and recording
   truncation metadata under
   `metadata.litellm_content_truncated`.

PR claims to close #26450. Default behaviour is preserved for the
truncation flag; gzip + batch splitting are unconditional changes.

## Key changes

### `litellm/integrations/azure_sentinel/azure_sentinel.py:124–129` — opt-in flag

```python
truncate_env = os.getenv("AZURE_SENTINEL_TRUNCATE_CONTENT", "false")
self.truncate_content = truncate_env.lower() in ("true", "1", "yes")
```

Default is `false`. The truthy set covers the common reasonable
strings; reasonable.

### `litellm/integrations/azure_sentinel/azure_sentinel.py:252–337` — `_enforce_column_limits`

```python
limit = self.MAX_COLUMN_CHARS  # 262_144
messages = payload.get("messages")
response = payload.get("response")
msg_str = str(messages) if messages is not None else ""
resp_str = str(response) if response is not None else ""

needs_truncation = len(msg_str) > limit or len(resp_str) > limit
if not needs_truncation:
    return payload  # cheap path

entry = deepcopy(payload)
...
prefix = "[truncated by litellm]..."
tail_limit = limit - len(prefix)

if len(msg_str) > limit:
    entry["messages"] = prefix + msg_str[-tail_limit:]
    truncated_fields.append("messages")
...
truncation_info = {
    "truncated": True,
    "truncate_reason": "azure_column_limit",
    "truncated_fields": truncated_fields,
    "original_messages_chars": len(msg_str),
    "original_response_chars": len(resp_str),
    "max_column_chars": limit,
}
```

A few things to flag here (see Concerns).

### `litellm/integrations/azure_sentinel/azure_sentinel.py:339–end` — `_split_into_batches`

```python
batches: List[bytes] = []
current_batch: list = []
current_size = 2  # account for JSON array brackets '[]'

for payload in payloads:
    if self.truncate_content:
        payload = self._enforce_column_limits(payload)

    entry_json = safe_dumps(payload)
    entry_size = len(entry_json.encode("utf-8"))

    if entry_size + 2 > self.MAX_BATCH_SIZE_BYTES:
        # Flush accumulated batch first, then send the oversize entry alone
        if current_batch:
            batches.append(gzip.compress(safe_dumps(current_batch).encode("utf-8")))
            ...
        single_body = safe_dumps([payload])
        ...
```

Estimates uncompressed JSON size, packs into ≤ 950 KB groups,
gzips each group. The "single oversize entry sent alone" branch
is the right behaviour — better to attempt a 1.2 MB gzipped POST
that *might* fit post-compression than to drop the entry.

### Doc updates

- `docs/my-website/docs/observability/azure_sentinel.md` adds an
  `AZURE_SENTINEL_TRUNCATE_CONTENT` row to the env table and a
  full "Handling Large Payloads" section covering gzip, batch
  splitting, and truncation behaviour. Includes the
  `litellm_content_truncated` metadata schema.
- `docs/my-website/docs/proxy/config_settings.md` adds the same
  env var to the global proxy config-settings reference table.

## What's good

- **Layered fix matching layered Azure limits.** Gzip + batch split
  address the request-size limit (1 MB); column truncation
  addresses the per-column limit (256 KB). These are genuinely
  different limits and need distinct handling — collapsing them
  into one would be wrong.
- **Truncation is opt-in.** Default behaviour is unchanged for
  existing deployments. Users who already rely on Azure's silent
  truncation get the same behaviour they had; users who want
  metadata + tail-preservation flip one env var.
- **Tail preservation is the right choice for chat logs.** The
  most diagnostically valuable bytes in a long chat transcript are
  the most recent ones (latest turns, model output). Head
  truncation would lose those; tail truncation preserves them at
  the cost of dropping system prompt / early context — a fair
  trade for security telemetry.
- **`needs_truncation` early-return** on `_enforce_column_limits`
  avoids the `deepcopy` cost in the common case where neither
  field exceeds the limit. Important — `deepcopy` of a fat
  StandardLoggingPayload is not free at proxy QPS.
- **Truncation metadata schema is rich** — `truncated_fields`
  list, original char counts, the limit value itself. That makes
  downstream KQL queries like "find requests where messages was
  truncated to >500 KB original" straightforward.
- The two debug `verbose_logger.debug(...)` calls in
  `async_log_success_event` / `async_log_failure_event` that were
  spamming kwargs at debug level are removed — minor but a real
  perf/log-volume win for high-throughput Sentinel users.
- Docs are updated in the same PR with the new env var, the
  metadata schema, and the rationale.

## Concerns

1. **`str(messages)` is `repr(list)`, not JSON serialization.** The
   length check uses `len(str(messages))`, but the *actual* payload
   sent to Azure is whatever `safe_dumps` produces (JSON). For a
   list-of-dicts `messages` field, `str(...)` produces Python repr
   form (`"[{'role': 'user', ...}]"`) and JSON produces
   `'[{"role": "user", ...}]'`. The character counts can diverge
   substantially (different quote styles, escape sequences,
   `True/False/None` vs `true/false/null`). The truncation
   threshold could trigger on a payload that JSON-serializes well
   under 256 KB, or — worse — *not* trigger on a payload that
   JSON-serializes over 256 KB. Should compare
   `len(safe_dumps(messages))` against the limit, then truncate
   the *string field that will actually be transmitted*. The
   current implementation also stores the truncated value as
   `prefix + msg_str[-tail_limit:]`, which is a Python repr — that
   string will then be JSON-encoded (re-escaped) and sent, which
   is not what the user expects to see in Sentinel.

2. **Truncation runs *before* JSON serialization, but column limit
   applies to the *serialized* column value.** Even ignoring
   concern #1, the 256 KB limit is the byte length of the
   *serialized JSON string column* in Log Analytics, not the
   length of the original Python value. Strings with non-ASCII
   characters (Chinese, Japanese, emoji) JSON-encode to `\uXXXX`
   sequences that are 6 chars per code-point — a 200 KB Python
   string of CJK can serialise to >1.2 MB JSON and still bust the
   column limit after this fix.

3. **`gzip.compress(...)` runs on the event loop thread.** The
   compression call is synchronous CPU work and the integration is
   `CustomBatchLogger` (async). For a 950 KB batch this is on the
   order of milliseconds, but for back-to-back flushes during a
   spike it can stall other coroutines. Consider
   `await asyncio.to_thread(gzip.compress, body)` for batches
   above some size threshold.

4. **`MAX_BATCH_SIZE_BYTES = 950_000` is uncompressed.** That's
   correct for the 1 MB-uncompressed-API-side limit, but Azure's
   actual limit is on the *received* compressed body. With
   typical JSON-of-LLM-payload compressing 5–10×, this is wildly
   conservative — most batches will compress to 100–200 KB and
   could safely include 5–10× more entries. Worth considering a
   compressed-size estimator (compress, check size, repack if
   over target) for high-throughput tenants. Not a blocker;
   correctness > throughput here.

5. **Single-oversize-entry branch can still bust the request
   limit.** If `entry_size > MAX_BATCH_SIZE_BYTES`, the entry is
   sent alone in a single POST. Post-gzip it usually fits, but
   for a 5 MB uncompressed entry of dense binary-ish data it
   may not. There's no fallback (drop? log warning?
   double-truncate? split the messages list across requests?).
   At minimum, a `verbose_logger.warning` when an entry exceeds
   `MAX_BATCH_SIZE_BYTES` would help operators see the problem.

6. **Doc duplication / typo at
   `azure_sentinel.md:130–183`.** The diff shows the new "Handling
   Large Payloads" section ending with a duplicated `Sends logs in
   the StandardLoggingPayload format / Automatically handles both
   success and failure events / Caches OAuth2 tokens and refreshes
   them automatically` block — these three lines appear both at
   `:131–134` (in the original "How It Works" bullet list) and at
   `:177–179` (orphaned at the tail of the new section). Pure
   markdown cleanup needed before merge.

7. **The four-rung config table change at `config_settings.md`** is
   missing column alignment in the diff (no trailing `|`), which
   may render odd in the published docs depending on the markdown
   processor. Cosmetic.

8. **No test added.** This is a 504-line change with non-trivial
   string slicing, batching arithmetic, and Azure-specific limits.
   The PR body checks the "added tests" box but the diff only
   shows source + docs changes. Actual tests under
   `tests/test_litellm/integrations/azure_sentinel/` would catch
   concerns #1, #2, #5 trivially. **Block on tests.**

## Risk

Medium-high. The default-off truncation is safe, but the
unconditional gzip + batch-split changes alter the wire format
and request count for every Sentinel user immediately on upgrade.
A bug in `_split_into_batches` can drop entries silently.
Concerns #1 and #2 are real correctness bugs in the truncation
math even when the feature is opted-in.

## Verdict

**request-changes**

The direction is right and the user-visible improvements are
genuinely valuable — Azure Sentinel's silent column truncation is
an annoyance for security teams and gzip is overdue. But the PR
ships three changes at once with no automated tests, has a real
correctness bug in the truncation length check (`str()` vs
serialization, concerns #1+#2), and has minor doc breakage that
should be cleaned up before publication.

Concrete asks before merge:

- Replace `len(str(messages))` with a serialization-aware length
  measurement, and truncate the actual string that will go on the
  wire (concerns #1, #2).
- Add at least three tests:
  - small payload → no truncation, no metadata field;
  - oversize ASCII payload → truncated, tail preserved, metadata
    populated with correct char counts;
  - batch of N entries totalling > 950 KB → split into ≥ 2
    POSTs, each within the size budget after gzip.
- Fix the duplicated 3-line block in the
  `azure_sentinel.md` "How It Works" section.
- Add the `verbose_logger.warning` when a single entry exceeds
  `MAX_BATCH_SIZE_BYTES` (concern #5).

Once those land this becomes a clean merge.

## What I learned

When you're enforcing limits on a *serialized representation* of
data (JSON column, gzip body, network MTU), the length check has
to use the same serialization the wire will use. `len(str(...))`
on a Python collection is one of the most seductively-wrong proxies
for "how many bytes will this be after JSON encoding" — close
enough to look right in tests with simple ASCII, off by an order
of magnitude for non-ASCII content. The right pattern is
`len(json.dumps(value, ensure_ascii=False).encode("utf-8"))` (or
the project's `safe_dumps` equivalent), measured *once*, both for
the gating decision and for the truncation slice.

The second lesson is operational: when a single PR ships three
related-but-independent improvements (compression, batching,
truncation), each with its own correctness profile, the sane
review move is to ask for the unconditional changes (gzip, batch
split) to land first as their own PR with their own tests, and
then the opt-in feature (truncation) to land on top. That keeps
blast radius bounded if any one of the three has a bug.
