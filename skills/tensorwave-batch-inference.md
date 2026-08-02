---
name: Run batch inference on a ScalarLM deployment
description: Submit a batch of prompts to a ScalarLM deployment's inference queue, poll for
  results, and handle per-item failures correctly.
api: openapi/tensorwave-scalarlm-openapi.yml
provider: tensorwave
operations:
  - listModels
  - generate
  - getResults
  - listRequests
  - getRequestDetail
  - getGenerateMetrics
generated: '2026-08-02'
method: generated
source: openapi/tensorwave-scalarlm-openapi.yml + conventions/tensorwave-conventions.yml
---

# Run batch inference on a ScalarLM deployment

Use this when you have many independent prompts and want throughput rather than a single
low-latency turn. For one interactive turn, use `createChatCompletion` instead (see
`tensorwave-openai-compatible-chat.md`).

## Before you start

- You need the base URL of a ScalarLM deployment (for example `http://localhost:8000` from
  `./scalarlm up`, or the deployment URL your operator gave you).
- **There is no API key.** ScalarLM declares no security scheme; send no `Authorization` header.
  If the deployment rejects you, it is a network perimeter or gateway in front of ScalarLM, not
  the application. See `authentication/tensorwave-authentication.yml`.
- Set `Content-Type: application/json`.

## Steps

1. **Confirm what is deployed.** Call `listModels` (`GET /v1/models`). It proxies vLLM and returns
   the OpenAI model-list envelope. `listDeployedModels` (`GET /v1/megatron/list_models`) is the
   training-side view of the same question.

2. **Submit the batch.** Call `generate` (`POST /v1/generate`) with a `prompts` array. Each entry
   is one of three shapes, and you may mix them:
   - a bare string,
   - `{"prompt": "..."}`,
   - `{"messages": [{"role": "user", "content": "..."}]}` — rendered through the model's chat
     template at enqueue time.

   Optional body fields: `model`, `max_tokens` (default 16 — raise it, the default is very low),
   `temperature` (default 0.0), `tools`, `tool_choice`.

3. **Read the response envelope.** You get `{"results": [{"request_id": ..., "response": ...,
   "error": ...}]}`. `response` is either a completion string or an embedding vector (array of
   floats), depending on the request type.

4. **Poll for anything not yet finished.** Call `getResults` (`POST /v1/generate/get_results`)
   with `{"request_ids": [...]}`. It returns the same `results` envelope. Poll on a backoff; there
   is no callback or webhook.

5. **Check every item's `error` field.** This is the step that gets skipped and it is the one that
   matters. A batch can partially fail: the HTTP status is **200** while individual entries carry
   a non-null `error` string. Never infer success from the status code alone. See
   `errors/tensorwave-problem-types.yml`.

6. **Inspect and audit.** `listRequests` (`GET /v1/generate/list_requests`) pages through recent
   requests using an opaque `cursor` plus `limit` (default 50). `getRequestDetail`
   (`GET /v1/generate/request/{request_id}`) returns one request.

## Watching throughput

`getGenerateMetrics` (`GET /v1/generate/metrics`) returns `queue_depth`,
`total_completed_requests`, `total_completed_tokens`, `tokens_per_second`, `requests_per_second`
and `flops_per_second`. Rising `queue_depth` is the backpressure signal — **there are no
rate-limit headers on this API**, so if you are pacing a large job, pace it against `queue_depth`.
`getPrometheusMetrics` (`GET /v1/metrics`) exposes the same picture in Prometheus text format.

## Rules

- **No idempotency.** There is no `Idempotency-Key`. Re-POSTing `generate` after a timeout
  enqueues the work a second time. Record the `request_id`s you got back and reconcile with
  `getResults` before retrying anything.
- **Never call `clearQueue`** (`POST /v1/generate/clear_queue`) to recover from your own error. It
  discards queued work for every tenant of the deployment.
- `getWork`, `finishWork`, `getAdaptors`, `uploadGenerateData` and `downloadGenerateData` are
  worker-plane operations. Do not call them as a client; you will steal work from the inference
  workers.
- Send a `traceparent` header (W3C Trace Context) if you want your calls correlated in the
  deployment's logs. No request id is returned to you in a response header.
