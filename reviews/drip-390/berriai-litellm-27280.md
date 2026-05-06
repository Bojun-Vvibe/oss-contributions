# Review: BerriAI/litellm#27280 — Add Azure Sentinel audit log support

- Head SHA: `7e3cdd9352779d9694d2ea834deeb8d2aef5c3cf`
- Files: 4 (+152 / -18)
- Verdict: **merge-after-nits**

## Summary

Adds native Azure Sentinel audit-log support so `audit_log_callbacks: ["azure_sentinel"]` actually queues `StandardAuditLogPayload` records into the existing Azure Monitor Logs Ingestion batching path. Previously the Sentinel logger inherited the no-op `async_log_audit_log_event` from `CustomLogger`, silently dropping audit events.

## Specific evidence

- **Bug repro shown explicitly in PR body**: `pytest tests/test_litellm/integrations/test_azure_sentinel.py::test_azure_sentinel_queues_audit_log_event -q` fails on the starting ref because `logger.log_queue == []` instead of holding the audit payload. PR includes the post-fix `3 passed` transcript.
- **New `async_log_audit_log_event`** at `litellm/integrations/azure_sentinel/azure_sentinel.py:247-273`:
  ```python
  async def async_log_audit_log_event(
      self, audit_log: StandardAuditLogPayload
  ) -> None:
      ...
      self.log_queue.append(audit_log)
      if len(self.log_queue) >= self.batch_size:
          await self.async_send_batch()
  ```
  Reuses the existing `log_queue` + `async_send_batch` ingestion path — same OAuth, same batching, same Azure Monitor Logs Ingestion endpoint. Correct: audit logs and proxy success/failure logs both land in the configured DCR stream.
- **Queue type widening** at `:122-124`:
  ```python
  self.log_queue: List[Union[StandardLoggingPayload, StandardAuditLogPayload]] = []
  ```
  Correct typed widening — `async_send_batch` `safe_dumps` will serialize either shape.
- **Registry wiring** at `litellm/litellm_core_utils/custom_logger_registry.py:17,77`: imports `AzureSentinelLogger` and adds `"azure_sentinel": AzureSentinelLogger` to the dispatch dict. This is what allows `audit_log_callbacks: ["azure_sentinel"]` in proxy config to resolve.
- **`litellm/proxy/_types.py`** (+12 lines) — adds `azure_sentinel` to the env-var introspection enum so the existing `/get_supported_callbacks` style endpoints surface it.
- **Hygiene cleanup**: deletes two duplicate `import time` statements at `:129,173` and one inline `from litellm.litellm_core_utils.safe_json_dumps import safe_dumps` at `:289` — promotes them to module-level. Correct: `time` is used in two methods, so module-level is the right scope. The `safe_dumps` promotion is a small import-cycle audit risk but `safe_json_dumps` should have no transitive dependency back into integrations.
- **Test additions** at `tests/test_litellm/integrations/test_azure_sentinel.py` (+103 / -7):
  - `test_azure_sentinel_queues_audit_log_event` pins `log_queue` length after `async_log_audit_log_event(...)`.
  - `test_azure_sentinel_sends_audit_log_payload_to_ingestion_api` pins the actual HTTP body sent for an audit payload.
  - `test_azure_sentinel_oauth_and_send_batch` retained — no regression on the OAuth/batch path.

## Nits

1. **PR mentions `5074 passed, 40 skipped` from the broader suite then "unrelated failures"** (`test_dst_fall_back` missing tz data, `test_acreate_simple` invalid OpenAI key). These should be confirmed unrelated by ref-name (i.e. these tests fail on `main` without this PR) — easy to verify, would raise this to merge-as-is.
2. **No documentation page added** in `docs/my-website/docs/proxy/audit_logs.md` (or similar) listing `azure_sentinel` as a supported callback. The wire-shape is now functional but discoverability is zero without a doc bump.
3. **Demo video reference** to `/opt/cursor/artifacts/azure_sentinel_audit_log_tests_demo.mp4` is local-only — gist upload failed due to missing scope per PR body. The transcript is in the PR body, which is sufficient. No action needed but the unreachable artifact path could be scrubbed.
4. **`async_log_audit_log_event` exception handler** at `:269-273` swallows the exception with `pass` after a `verbose_logger.exception(...)`. Consistent with the sibling `async_log_failure_event` at `:240-247`, so this matches house style — but a queue overflow during a 4xx OAuth refresh storm could silently drop audit records. Audit logs are typically append-only-must-not-lose; worth a maintainer note on whether to add a bounded-retry path here as a follow-up.
5. The `Union[StandardLoggingPayload, StandardAuditLogPayload]` queue type means `async_send_batch` (not shown in diff but called at `:265`) needs to handle both shapes when constructing the ingestion-API row. Should be confirmed by inspecting the body-construction path that it doesn't index `kwargs["standard_logging_object"]` or similar success-payload-only fields.

Wire-correct fix for a real silent-drop bug, with focused regression tests. Merge after CI confirms the unrelated test failures are unrelated.
