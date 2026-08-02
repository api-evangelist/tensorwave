# TensorWave

TensorWave is a Las Vegas-headquartered AI cloud provider that builds and operates bare-metal GPU
infrastructure exclusively on AMD Instinct accelerators (MI300X, MI325X, MI355X, MI455X) with
AMD's open ROCm software stack. It sells dedicated GPU nodes, high-speed network storage, and
managed Slurm and Kubernetes clusters for AI training, fine-tuning and inference.

- Website — https://tensorwave.com/
- Documentation — https://docs.tensorwave.com/
- Trust Center — https://security.tensorwave.com/
- GitHub — https://github.com/tensorwavecloud

## API surface

The TensorWave compute platform itself has **no public REST control-plane API** — it is consumed
over SSH, Slurm and Kubernetes, with observability and alerting in-product.

TensorWave's public API surface is **ScalarLM**, the CC0-licensed unified training-and-inference
stack it maintains and sponsors alongside RelationalAI. One deployment exposes an
OpenAI-compatible inference endpoint backed by vLLM, a queue-backed batch generate surface, a
Megatron-LM training surface dispatched through Slurm, and health/metrics endpoints.

- ScalarLM docs — https://www.scalarlm.com/docs/
- ScalarLM source — https://github.com/tensorwavecloud/ScalarLM
- ScalarLM SDK/CLI — https://pypi.org/project/scalarlm/

## Artifacts in this repo

| Path | What it is |
|---|---|
| `openapi/` | OpenAPI 3.1, 45 operations. **Derived from the ScalarLM FastAPI source**, not provider-published — see `info.x-evidence`. |
| `overlays/` | API Evangelist enhancements over that spec (agentic-access classes, caveats, pagination). |
| `skills/` | Three packaged Agent Skills grounded in real operationIds. |
| `authentication/` | Auth profile. ScalarLM declares no security scheme; the platform uses SSH keys. |
| `conventions/` | Pagination, batching, streaming transports, chunked upload, tracing, error envelope, CORS. |
| `errors/` | Error catalogue. Not RFC 9457 — failures ride inline on 200 responses. |
| `data-model/` | Entity graph: inference queue, training jobs, checkpoints, publish jobs. |
| `lifecycle/` | Versioning; records the absence of a status page, SLA and deprecation policy. |
| `changelog/` | ScalarLM GitHub releases plus the PyPI release train. |
| `packages/` | The `scalarlm` PyPI package (SDK + CLI). |
| `cli/` | `./scalarlm` orchestration script and the pip-installed monitoring CLI. |
| `conformance/` | Standards conformance plus the published compliance program. |
| `security/` | Domain security probe, trust center (SOC 2 Type 2, ISO 27001 v2022, HIPAA), vulnerability disclosure policy. |
| `well-known/` | `/.well-known/` probe results across every host. None published. |
| `llms/` | Verbatim `llms.txt` harvested from the docs host. |
| `mcp/` | Design **candidate** only. TensorWave ships no MCP server, so no `MCPServer` pointer is wired. |

Not present, and correctly so: no AsyncAPI or webhooks (alerting is Slack/email, not a
subscribable event surface), no A2A agent card, no OAuth scopes, no sandbox test credentials, no
gRPC, no embedded components.
