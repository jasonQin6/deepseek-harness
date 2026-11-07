---
kind: upgrade-guide
description: "A malformed tool call from a pi-ai route now fails with the retryable MALFORMED_TOOL_CALL code, and stored empty-name calls are dropped from requests."
---

# Malformed model tool calls use their own error code

English | [中文](guide.zh.md)

## Change

A pi-ai route reports completed tool arguments that are not valid JSON, and a completed call whose name is empty, as the new `MALFORMED_TOOL_CALL` failure code instead of `MALFORMED_RESPONSE`. The default normal retry policy retries `MALFORMED_TOOL_CALL`, so one re-roll of the step usually clears a model sampling defect; `MALFORMED_RESPONSE` now covers wire-protocol violations only and stays outside the default retryable set. Both adapters also drop a stored empty-name tool call together with the tool result that answered it, so a Session written before this release replays without that call while its durable content keeps both.

Scripts, telemetry, and dashboards that match `failure.code` in Session records or `assistant/attempt` events are affected, as are provider `retryPolicy.retryableCodes` settings. Stored Session records keep their old codes; only new failures use the new one.

## Migration

1. Where code treated `MALFORMED_RESPONSE` as "the model returned broken tool arguments", match `MALFORMED_TOOL_CALL` instead. Keep `MALFORMED_RESPONSE` for malformed SSE framing, event-type mismatches, and unreadable replay state, which repeat deterministically.
2. In a provider `retryPolicy.retryableCodes` list, add `MALFORMED_TOOL_CALL` to retry malformed model tool calls like the default policy does, or leave it out to fail such attempts immediately.
3. To confirm, send a tool-calling prompt through a pi-ai route on a provider that returns malformed arguments and check that the attempt reports `MALFORMED_TOOL_CALL` and that the step is retried. Then open a Session written before the upgrade that contains an empty-name call and check that its next request succeeds.
