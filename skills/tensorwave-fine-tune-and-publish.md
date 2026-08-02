---
name: Fine-tune a model on ScalarLM and publish it to Hugging Face
description: Upload a training dataset in resumable chunks, launch and monitor a Megatron-LM
  training job on Slurm, then push the resulting checkpoint to the Hugging Face Hub.
api: openapi/tensorwave-scalarlm-openapi.yml
provider: tensorwave
operations:
  - initChunkedUpload
  - uploadChunk
  - finalizeChunkedUpload
  - submitTrainingJob
  - getTrainingJob
  - getTrainingLogs
  - getTrainingDataset
  - listCheckpoints
  - publishToHuggingFace
  - getPublishStatus
  - getPublishLogs
  - cancelPublish
  - getMegatronSqueue
  - getGpuCount
  - getNodeCount
generated: '2026-08-02'
method: generated
source: openapi/tensorwave-scalarlm-openapi.yml + conventions/tensorwave-conventions.yml
---

# Fine-tune a model on ScalarLM and publish it to Hugging Face

The whole point of ScalarLM is that inference and training share one deployment and one checkpoint
store. This flow trains against the same instance you are serving from; the inference pod picks up
the new checkpoint without a restart.

## Before you start

- A ScalarLM deployment URL. No API key — see `authentication/tensorwave-authentication.yml`.
- **This flow spends real GPU cluster time.** Check capacity first with `getGpuCount`
  (`GET /v1/megatron/gpu_count`), `getNodeCount` (`GET /v1/megatron/node_count`) and
  `getMegatronSqueue` (`GET /v1/megatron/squeue`) before queueing anything.
- For publishing you need a Hugging Face access token with write permission.

## Part 1 — get the dataset in

For a small dataset, `submitTrainingJob` (`POST /v1/megatron/train`) takes the data and job config
in one streamed call and launches the job directly.

For anything large, use the three-phase resumable upload:

1. `initChunkedUpload` (`POST /v1/megatron/upload/init`) with `total_size`, `total_hash`,
   `chunk_size`, `num_chunks`, `compressed` (default true) and `params` (your job config). It
   returns an `upload_id` and `received_chunks` — an array of chunk indexes the server already
   has. **On a resumed upload, skip every index in that array.**
2. `uploadChunk` (`POST /v1/megatron/upload/chunk`) once per chunk. The metadata goes in headers,
   not the body: `X-Upload-Id`, `X-Chunk-Index` (zero-based), `X-Chunk-Hash`. The chunk bytes are
   the body. Missing any of the three headers is a 400. The response confirms
   `received` and `bytes_written`.
3. `finalizeChunkedUpload` (`POST /v1/megatron/upload/finalize`) with `{"upload_id": ...}`. This
   both closes the upload and launches the training job, returning `job_status`, `job_config` and
   `deployed`.

Chunk hashes plus the whole-payload `total_hash` are the integrity check — compute them honestly,
because a mismatch will surface as a failed job rather than a failed upload.

## Part 2 — run and watch the job

Everything downstream is keyed on the `job_hash` you got back.

- `getTrainingJob` (`GET /v1/megatron/train/{job_hash}`) — current status.
- `getTrainingLogs` (`GET /v1/megatron/train/logs/{model_name}`) — SSE stream of log lines. Resume
  with `starting_line_number`.
- `getTrainingDataset` (`GET /v1/megatron/train/{job_hash}/dataset`) — page the training rows the
  job actually ingested with `offset`/`limit` (default 50) and an optional `q` filter. Use this to
  verify the upload landed as intended before you burn hours of cluster time.
- `downloadTrainingDataset` (`GET /v1/megatron/train/{job_hash}/dataset/download`) — the raw
  `dataset.jsonlines`, whole file or first N rows.
- `listCheckpoints` (`GET /v1/megatron/train/{job_hash}/checkpoints`) — checkpoints written so
  far, named `checkpoint_<step>.pt`.

There is no completion callback. Poll `getTrainingJob` on a backoff, or tail the log stream.

## Part 3 — publish the checkpoint

`publishToHuggingFace` (`POST /v1/megatron/train/{job_hash}/publish`) submits a Slurm publish job
and returns immediately with a publish job id. Body:

- `repo_id` (**required**) — `owner/name` on the Hub.
- `hf_token` (**required**) — write-scoped Hugging Face token.
- `mode` — `merged` (fold LoRA into the base model, the default) or `adapter` (a PEFT-format
  adapter repo).
- `private` — create the repo private if it does not exist. Default `false`.
- `checkpoint` — basename of the `checkpoint_<step>.pt` to publish. Defaults to the latest.
- `lora_alpha` — override the alpha used at merge time.
- `commit_message` — auto-generated if omitted.

Then:
- `getPublishStatus` (`GET /v1/megatron/train/{job_hash}/publish/status`) — poll for progress.
- `getPublishLogs` (`GET /v1/megatron/train/{job_hash}/publish/logs`) — tail with
  `starting_line_number` / `starting_byte_offset` / `tail` / `limit`.
- `cancelPublish` (`POST /v1/megatron/train/{job_hash}/publish/cancel`) — `scancel`s the in-flight
  publish job and returns the resulting status.

## Rules

- **The `hf_token` is a secret in a request body.** ScalarLM forwards it to `sbatch` via env-var
  export and does not write it to disk or argv on the API pod, but your side of the call must not
  log the body, keep it in transcripts, or persist it.
- **`private` defaults to `false`.** Publishing a fine-tuned model to a repo that does not exist
  yet creates it **public** unless you set `private: true`. Decide deliberately.
- **No idempotency.** A retried `submitTrainingJob` or `finalizeChunkedUpload` starts a second
  training run and bills for it. Reconcile with `getMegatronSqueue` before retrying.
- `cancelTrainingJob`, `restartTrainingJob` and `deleteTrainingJob`
  (`POST /v1/megatron/{cancel,restart,delete}/{job_hash}`) are destructive and take effect
  immediately. `deleteTrainingJob` is irreversible. Confirm with a human first.
- `cancelSlurmJob` (`POST /slurm/cancel/{job_id}`) cancels by **Slurm job id**, not job_hash, and
  is unversioned (no `/v1` prefix). It can cancel jobs that are not yours on a shared cluster.
