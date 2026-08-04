# TensorWave

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

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
