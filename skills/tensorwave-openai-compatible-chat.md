---
name: Call ScalarLM as an OpenAI-compatible endpoint
description: Point an existing OpenAI client at a ScalarLM deployment for chat and text
  completions, and handle its two different response transports.
api: openapi/tensorwave-scalarlm-openapi.yml
provider: tensorwave
operations:
  - listModels
  - createChatCompletion
  - createCompletion
generated: '2026-08-02'
method: generated
source: openapi/tensorwave-scalarlm-openapi.yml + https://www.scalarlm.com/inference/
---

# Call ScalarLM as an OpenAI-compatible endpoint

ScalarLM's `/v1` surface is a drop-in OpenAI-compatible endpoint backed by vLLM. Any OpenAI client
works — set the base URL to the deployment and leave the API key empty or set to any placeholder.

## Before you start

- Base URL is the deployment root; the OpenAI paths hang off `/v1`.
- **No authentication.** The documented example on scalarlm.com sends only
  `Content-Type: application/json`. See `authentication/tensorwave-authentication.yml`.
- The request schemas are vLLM's own `ChatCompletionRequest` and `CompletionRequest`, so the
  OpenAI field set applies. Only fields on the router's allow-list are forwarded upstream.

## Steps

1. **Discover the model.** `listModels` (`GET /v1/models`) proxies vLLM's model list. A ScalarLM
   deployment usually serves exactly one model, set in `values.yaml` at deploy time — you do not
   pick a model per request the way you would against a multi-model vendor.

2. **Send the chat completion.** `createChatCompletion` (`POST /v1/chat/completions`) with a
   `messages` array of `{role, content}` objects. Roles are `system`, `user`, `assistant`, `tool`.

   ```
   curl <base>/v1/chat/completions \
       -H "Content-Type: application/json" \
       -d '{"messages":[{"role":"system","content":"You are a helpful assistant."},
                        {"role":"user","content":"Who won the world series in 2020?"}]}'
   ```

3. **Know which transport you are on.** This one operation has two completely different response
   behaviours, selected by `stream`:

   | `stream` | Path | Response |
   |---|---|---|
   | `true` | Proxied straight to vLLM | `text/event-stream` SSE, standard OpenAI chunks |
   | `false` | Admission control → coalescer → SQLite work queue → worker | Chunked JSON preceded by **whitespace heartbeats** that hold the connection open while the request waits in the queue |

   The non-streaming path is the one that surprises clients. A strict JSON parser that reads the
   body incrementally may choke on the leading whitespace; buffer the full body before parsing, and
   set a generous read timeout because the heartbeats mean the connection stays open by design
   rather than because the response is imminent.

4. **Text completions.** `createCompletion` (`POST /v1/completions`) takes `prompt` (string or
   array of strings) and always answers as an SSE stream.

## Error handling

Errors do **not** arrive as HTTP error statuses on the streaming path. Once headers are sent the
status is already 200, so an upstream vLLM failure arrives as a single SSE event:

```
data: {"error": "Failed to create chat completions: <upstream body>"}
```

Parse every SSE `data:` frame for an `error` key, not just for content deltas. A malformed request
that fails Pydantic validation before dispatch does return a real `422` with FastAPI's
`{"detail": [{"loc": ..., "msg": ..., "type": ...}]}` envelope. See
`errors/tensorwave-problem-types.yml`.

## Rules

- **From a browser, this will fail.** CORS is configured for `http://localhost:5173` only. Call it
  server-side or put your own proxy in front.
- No idempotency key exists; a retried completion is a second billed generation.
- For many prompts at once, do not loop this operation — use `generate` and the batch queue
  instead (see `tensorwave-batch-inference.md`).
