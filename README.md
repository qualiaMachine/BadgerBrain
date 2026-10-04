# BadgerBrain

BadgerBrain is UW-Madison's (upcoming) service for open-weight AI/ML models, run by
DoIT Research Cyberinfrastructure. It puts an OpenAI-compatible gateway
(`https://llm-gw01.doit.wisc.edu/v1`) in front of models hosted on
campus GPUs, so labs, courses, and campus apps call one stable API with
a per-user key instead of each renting cloud capacity or standing up
their own GPU box.

This repo is the service's working home: the user quickstart, the admin
runbook for handing out keys, the cluster guide the service team uses to
host models, and the example applications built on top of the gateway.

> **Before you use the pilot:** read the
> [Usage Policy](docs/usage-policy.md) — public data only, no
> availability guarantees, access by request. It sets expectations
> while the service is still pre-review.

## Start here

**Want access?** Fill in the
[BadgerBrain access form](https://forms.gle/vkcLzApNrX7KbkTP9). You'll
get an API key through 1Password and a link to the quickstart. Keys are
for UW-Madison NetIDs and require the campus VPN.

| You are... | Read |
|------------|------|
| Someone with a **gateway API key** | [BadgerBrain Quickstart](docs/gateway-quickstart.md) — setup with 1Password, Python and R examples, which models are hosted, cold-start behaviour |
| **Handing out keys** to a team, lab, course, or hackathon | [Handing Out BadgerBrain Access](docs/admin-gateway-access.md) — roster CSV → teams → keys → 1Password share links → email, plus autoscaling and load-testing notes |
| On the **service team**, or hosting your own model by arrangement | The [Cluster Guide](#cluster-guide) below |

**Training and fine-tuning** are not a BadgerBrain offering. Submit
those jobs to [CHTC's GPU pool](https://chtc.cs.wisc.edu/uw-research-computing/gpu-jobs)
(HTCondor); a model fine-tuned there can be hosted through the gateway
afterwards. Interactive notebook sessions on shared GPUs go through
BadgerCompute.

## What's hosted

The gateway serves a small, curated catalogue — one always-on generalist
chat model plus wake-on-demand specialists (vision/OCR, embeddings).
The current list is in the [quickstart](docs/gateway-quickstart.md);
which models the pilot should host and what triggers a swap is in the
[Shared Model Catalogue](docs/model-catalogue.md) draft.

Hardware today: one Dell PowerEdge node with two NVIDIA RTX PRO 6000
Blackwell GPUs (96 GB each), scheduled by Run:ai. A Phase 1 expansion
onto NVLinked B300 nodes is proposed.

## Cluster guide

How to host models on the Run:ai cluster. Written as a progressive
walkthrough for the service team and for anyone granted direct cluster
access by arrangement — a gateway key does not include this, and most
users never need it.

| # | Doc | Read this if... |
|---|-----|-----------------|
| 00 | [Overview](docs/00-overview.md) | You're new to the Run:ai cluster and want to know what it is and isn't good for |
| 01 | [Access](docs/01-access.md) | You need a login, project assignment, or storage quota |
| 02 | [First workspace](docs/02-first-workspace.md) | You want a working Jupyter notebook on the cluster with this repo cloned and a shared model loaded, in ~15 minutes |
| 03 | [Share a model as a vLLM endpoint](docs/03-share-as-endpoint.md) | You want to host a model once and have multiple users / workloads hit it via HTTP, instead of every user loading their own copy onto a GPU |
| 04 | [Storage](docs/04-storage.md) | You need to know where data lives — short-term scratch through cluster-wide shared datasets — and how to get it from "a drive in my lab" to "mountable in a workload" |
| 05 | [Examples](docs/05-examples.md) | You're ready to deploy something — pointers to the OCR pipeline, the RAG/chatbot, and the patterns to copy when building your own |
| 06 | [Expose a vLLM endpoint outside the cluster](docs/06-external-endpoint.md) | A non-Run:ai client (Denodo, an institutional app, your laptop) needs to call a hosted model over `https://`, including the cross-VLAN firewall hand-off |
| 07 | [Submit workloads via the Run:ai CLI](docs/07-cli-submission.md) | You want scriptable, repeatable submissions from your own machine instead of the web UI — install/config on Windows, verified submit commands, and the gotchas |

The OCR-specific and RAG-specific deployment guides live in the app
READMEs — [`ocr_app/README.md`](ocr_app/README.md) and
[`rag_app/README.md`](rag_app/README.md) — with per-step details
under each app's `docs/`. Those assume you've already worked through
00–04 here. All workloads are created through the Run:ai web UI.

## Applications

Example use cases built on the gateway and the cluster — PoCs we are
building out as the pilot uncovers what labs actually need. Use them as
starting templates and adapt to your needs.

| Path | What it is | When you'd use it |
|------|------------|-------------------|
| [`ocr_app/`](ocr_app/README.md) | Vision-language document extraction (Qwen3-VL-32B). Turns PDFs/scans into structured JSON. | Grant administration, archival corpora, library digitization, anything where layout matters |
| [`rag_app/`](rag_app/README.md) | Retrieval-augmented chatbot over a curated corpus (Qwen 7B/14B/72B). | Q&A over institutional knowledge bases, research literature search, "ChatGPT for our docs" |
| [`image_app/`](image_app/README.md) | Text-to-image generation (Qwen-Image) with a browser UI. | Cartoon diagrams and illustrative graphics for presentations, posters with rendered text |
| [`scripts/`](scripts) | Shared utilities used by the apps and by the service team. | You usually don't touch this directly. |


### [Document Extraction (`ocr_app/`)](ocr_app/README.md)

Structured data extraction from institutional documents — grant award
notices, budgets, terms & conditions, archival scans, and other records.
Every page is rendered as an image and sent to a Vision Language Model
(Qwen3-VL-32B-Instruct-AWQ), so the model sees layout, tables, signatures,
watermarks, and annotations — not just raw text.

- **Chunk-based two-pass pipeline (notebooks):** overlapping page chunks
  sent in single VLM calls, then programmatic merge + continuation-flag
  stitching across chunks, then a doc-level pass-2 synthesis
- **Two variants:** grant administration schema
  (stakeholders/tables/narratives) and library/archival schema
  (bibliographic metadata, body text, marginalia, stamps)

**Status:** PoC validated on sample documents.

### [RAG Chat (`rag_app/`)](rag_app/README.md)

WattBot — retrieval-augmented generation over research paper corpora.
Chat interface for querying scientific literature with citations. 2025
WattBot Challenge winner.

- **4 services on 1 GPU:** vLLM (LLM), Jina V4 (embeddings),
  cross-encoder (reranker), Streamlit (UI) via fractional GPU allocation
- Multiple knowledge bases, hybrid search (vector + BM25)
- Supports Qwen, Llama, OpenScholar models

**Status:** Deployed on Run:ai, documented end-to-end.

## Infrastructure

```
  laptop / notebook / campus app
           | HTTPS + per-user key (OpenAI API)
           v
  +------------------------------+
  |  LiteLLM gateway             |  llm-gw01.doit.wisc.edu — keys, teams,
  |  (VM, docker compose)        |  usage, model routing; config in GitLab
  +--------------+---------------+
                 | HTTP (cluster-internal DNS)
                 v
  +------------------------------+
  |  vLLM inference workloads    |  Run:ai — GPU, fractional allocation,
  |  on the GPU cluster          |  scale-to-zero for on-demand models
  +------------------------------+
```

Each app in this repo includes:
- `app.py` and supporting scripts (Streamlit UI, FastAPI servers, batch
  CLI)
- `docs/` — step-by-step Run:ai deployment guides (storage setup,
  model provisioning, workspace config, troubleshooting)
- Requirements files split by role (UI/client vs. GPU server)

All apps use the same approach:

- **No Docker builds** — stock images (`vllm/vllm-openai`,
  `nvcr.io/nvidia/pytorch`) with deps installed at startup
- **Fractional GPU** — multiple services share one physical GPU
- **Shared model PVC** — download once, mount read-only everywhere
- **Knative DNS** — services addressed via FQDN
  (`workload.runai-project.svc.cluster.local`)

### Shared utilities (`scripts/`)

| Script | Purpose |
|--------|---------|
| `hardware_metrics.py` | GPU/energy profiling — VRAM, power draw, energy per request |
| `provision_shared_models.py` | Download HuggingFace models to the shared PVC; `vram` subcommand estimates serving VRAM (weights + KV-cache/`--max-model-len` guidance) before you commit GPU quota |
| `provision_gateway_keys.py` | Admin-only: create LiteLLM gateway teams and per-user keys from a roster CSV, filing each key in 1Password and emitting recipient-locked share links. See [Handing Out BadgerBrain Access](docs/admin-gateway-access.md) |



## Author

- **Chris Endemann** — Research Cyberinfrastructure Consultant, RCI/DoIT, UW-Madison
