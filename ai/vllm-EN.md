# vLLM

*Evaluated: vLLM 0.20.x (April–May 2026), V1 engine default. As of May 2026.*

## Summary

vLLM is the open-source reference LLM serving engine: a Python/C++/CUDA stack built around **PagedAttention** (KV cache organized into fixed-size physical blocks, addressed via per-sequence block tables) and **continuous batching** (per-iteration scheduling that admits and evicts requests between forward passes). It originated at UC Berkeley's Sky Computing Lab (SOSP 2023) and is now the de-facto generic serving target — running production traffic at Meta, LinkedIn, Mistral, and Hugging Face, supported on NVIDIA Hopper/Blackwell, AMD MI300X-class, Google TPU, AWS Neuron, and Intel Gaudi. Its differentiator versus peers is **breadth, not peak throughput**: TensorRT-LLM beats it on NVIDIA-only kernels, SGLang beats it on prefix-heavy workloads, but neither covers the model × hardware × deployment matrix vLLM does. Pick vLLM when you want one engine to serve diverse models on heterogeneous hardware with fast model adoption; reach for TensorRT-LLM, SGLang, or TGI when a specific dimension (NVIDIA peak perf, prefix cache hit rate, HF ecosystem fit) dominates.

## Comparison

| Dimension | **vLLM 0.20** | **SGLang** (latest 2026) | **TensorRT-LLM 0.19+** | **TGI v3** (HF) |
|---|---|---|---|---|
| Type / category | General-purpose OSS LLM serving engine | OSS serving engine optimized for prefix sharing | NVIDIA-proprietary (OSS license) compiled inference runtime | HF's production serving daemon (Rust + Python) |
| Core architecture | PagedAttention KV cache + continuous batching; V1 engine with async scheduling | RadixAttention (radix-tree prefix sharing) + continuous batching | Per-model compiled TensorRT engine + in-flight batching + plugin kernels | Rust router/scheduler over per-model Python workers; multi-backend (vLLM / TRT-LLM / native) |
| Primary interfaces | OpenAI-compatible HTTP, gRPC, Python `LLM` API; `vllm serve` CLI | OpenAI-compatible HTTP; `sgl.Runtime`; structured `sglang` DSL for prompts/programs | Triton Inference Server; `trtllm-serve` (OpenAI-compatible); C++/Python runtimes | OpenAI-compatible HTTP; `text-generation-launcher` |
| Hardware support | NVIDIA (Ampere/Hopper/Blackwell incl. B200/GB200), AMD MI300X/MI325X/MI350X/MI355X (first-class as of 2026), Google TPU v5e/v6e/v8i, AWS Neuron, Intel Gaudi | NVIDIA primary; AMD ROCm; some TPU work | NVIDIA only (SM80–SM103 incl. B300/GB300) | NVIDIA primary; AMD via ROCm; Inferentia/Gaudi via backends |
| Quantization | FP8 (E4M3/E5M2), AWQ, GPTQ, BnB, FP4/MXFP4 (recent), TurboQuant 2-bit KV cache | FP8, AWQ, GPTQ; FP4 emerging | FP8, NVFP4, MXFP4, INT4-AWQ, INT8 SmoothQuant; FP4 MLA on Hopper/Blackwell | FP8, AWQ, GPTQ, BnB |
| Best fit | Heterogeneous fleets serving many models with frequent model swaps; new model day-zero | Prefix-heavy traffic — RAG with shared docs, multi-turn chat, agent tool-calling traces | Squeezing peak tok/s/$ out of a fixed NVIDIA SKU + a stable model | Tight fit to HF Hub model cards, OpenTelemetry-first ops |
| Advantages | Widest model + hardware coverage; fastest community for new model architectures; mature ecosystem (LMCache, llm-d, production-stack) | Highest prefix-cache hit rate; up to ~6.4× wins on prefix-heavy workloads; programmable prompt language | Highest peak throughput on NVIDIA; lowest p95 TTFT under load on H100/B200 | Built-in Prometheus/OTel; smooth HF integration; long-prompt path improved in v3 |
| Disadvantages | Not absolute peak on NVIDIA (TRT-LLM ~13% ahead at 50 concurrent on 70B FP8); disaggregated P/D still labeled experimental; Python overhead in scheduler hot paths historically | Python router can hit GIL ceiling at high concurrency (~127% CPU cap reported under 150 concurrent); narrower hardware coverage; smaller ecosystem | NVIDIA lock-in; per-model engine build (~28 min per version per SKU); slowest to add brand-new model architectures | Now partly a thin layer — actively being unified with vLLM as a backend; less dev velocity than vLLM/SGLang directly |
| Numerical/eval model | FP16/BF16 reference; FP8 within ~0.5–1% of BF16 on standard evals; new quant modes shipped with eval reports | Same model accuracy as vLLM at matching precision (it's the same models) | Builder may fuse kernels that drift from reference; vendor publishes accuracy reports | Uses underlying backend's numerics |
| License | Apache 2.0 | Apache 2.0 | Apache 2.0 (engine), but built against proprietary CUDA/TensorRT | Apache 2.0 (runtime); HF Inference Endpoints commercial |
| Cost | Framework free. **Realistic infra**: 8×H100 SXM 80GB cluster, 1-yr reserved cloud ≈ $150–250k/yr (varies by provider). Self-managed K8s deploy free; managed offerings (Anyscale, RunPod, Modal, etc.) add ~10–30% overhead | Same cost shape as vLLM | Same compute cost; engineer-time cost for engine builds + recompiles on model/SKU change | Same compute; HF Inference Endpoints managed pricing per-replica/hour |

*Cost figures are public-list / order-of-magnitude estimates as of May 2026; cloud GPU pricing moves quickly.*

### Sub-comparison: throughput & latency on Llama 3.3 70B FP8, single H100 SXM5 80GB

Drawn from a third-party 2026 benchmark — directional, not authoritative. Input ~512 tokens, output ~256 tokens.

| Concurrency | vLLM tok/s | TRT-LLM tok/s | SGLang tok/s | vLLM p95 TTFT | TRT-LLM p95 TTFT | SGLang p95 TTFT |
|---|---|---|---|---|---|---|
| 1   | 120  | 130  | 125  | 68 ms   | 55 ms   | 61 ms   |
| 10  | 650  | 710  | 680  | 195 ms  | 170 ms  | 178 ms  |
| 50  | 1,850| 2,100| 1,920| 720 ms  | 620 ms  | 680 ms  |
| 100 | 2,400| 2,780| 2,460| 1,450 ms| 1,280 ms| 1,380 ms|

TensorRT-LLM leads peak throughput by ~13% at high concurrency; vLLM and SGLang are within ~5% of each other on this unique-prompt workload. SGLang's edge appears on **shared-prefix** workloads (RAG, multi-turn), where the same paper-cited setup shows up to 6.4× gains over vLLM that this table does not capture.

## In-depth report

### 1. Architecture deep-dive

```
                      ┌────────────────────────────────────┐
   HTTP/gRPC ───▶ Frontend (OpenAI-compatible)             │
                  │                                        │
                  ▼                                        │
            ┌──────────────┐    admit/evict      ┌────────┴────────┐
            │  Scheduler   │◀───────────────────▶│  Block Manager  │
            │  (V1)        │   per-iteration     │  (PagedAttn)    │
            └──────┬───────┘                     └────────┬────────┘
                   │ batched request                      │
                   ▼                                      ▼
            ┌──────────────┐                       ┌─────────────┐
            │ Model Runner │ ── attention ───────▶│  KV Cache   │
            │ (V2)         │ ── matmul/MoE ─────▶│  blocks     │
            └──────────────┘                       └─────────────┘
                   │
                   ▼
              Sampler (greedy / nucleus / spec)
                   │
                   ▼
                 Output
```

**Frontend.** Receives OpenAI-format `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`. Streams via SSE. Tokenization happens here (tokenizers can run in a worker pool to avoid blocking the event loop).

**Scheduler (V1).** Per-iteration scheduler running on the same process as the model runner. The V1 rewrite (default since 0.18-ish, no longer requires `VLLM_USE_V1=1`) eliminated GPU↔CPU tensor copies that the legacy scheduler did each step; it pins host memory and uses zero-copy DMA for sampling/output processing. The scheduler maintains three queues — `waiting`, `running`, `swapped` — and on each step fills a token budget. With **chunked prefill** enabled it can co-schedule prefill chunks alongside decode steps so prefill doesn't starve interactive decode.

**Block Manager (PagedAttention).** KV cache is partitioned into **physical blocks of 16 tokens** (configurable via `--block-size`, common values 8/16/32). Each request owns a block table mapping logical positions to physical block IDs. Because blocks are non-contiguous and reference-counted, fragmentation is essentially eliminated — published reduction in wasted KV memory is up to ~55% versus naive contiguous allocation. The same indirection enables **automatic prefix caching (APC)**: identical prompt prefixes hash to the same block IDs and the K/V tensors are reused across requests.

**Model Runner V2.** Owns CUDA graphs, attention backends (FlashAttention, FlashInfer, ROCm-native), MoE kernels (TritonMoE, fused experts), and pipeline/tensor-parallel comms (NCCL/RCCL, custom all-reduce). V2 added piecewise CUDA graphs for pipeline parallelism and a more flexible spec-decode rejection sampler.

**Speculative decoding.** Native support for multiple draft strategies: a small draft model, n-gram lookup, suffix decoding, MLP speculators, and EAGLE/EAGLE-3. Async scheduling now supports speculative decoding with **zero-bubble overlap** — the scheduler hides verification latency behind the next step's preparation.

**Disaggregated prefill/decode (P/D).** vLLM splits prefill and decode into separate engine instances connected by a KV-cache transfer connector — `NixlConnector`, `MooncakeConnector`, or `P2pNcclConnector`. The point is to scale TTFT (prefill) and ITL (decode) independently and stop long prefill jobs from blocking decode steps. As of 0.20, the feature is functional and run in production by Meta/LinkedIn/Mistral/HF, but the docs still label it experimental, and it is sensitive to interconnect bandwidth: a 70B FP8 prefill that needs to ship multi-GB of KV across nodes wants ≥4–5 GB/s sustained or you blow your 500 ms TTFT budget.

### 2. Key design patterns and trade-offs

- **Block-paged KV over contiguous KV.** The single biggest design choice. Pays back in memory utilization and prefix sharing; costs you a layer of indirection in the attention kernel that competing engines (early TGI, FasterTransformer-era) avoided. The cost is now negligible thanks to FlashAttention/FlashInfer kernels that are paged-aware.
- **Continuous batching over static batching.** Static batching wastes GPU on padding and forces tail-of-batch waits. Continuous batching admits new requests every iteration, paying a small per-step scheduling cost. V1's host-pinned DMA path was about reducing that cost to where it isn't visible at thousands of concurrent requests.
- **One Python process per engine, not micro-services.** Scheduler, block manager, and model runner co-locate to keep the hot path in-process. Multi-GPU is via TP/PP across workers under the same engine, not via multiple HTTP-served replicas. Trade-off: the GIL was historically a real ceiling on the scheduler — the V1 rewrite and recent C++ work in the request path push more off-Python, but very-high-concurrency setups still benefit from running multiple engine processes behind a router.
- **Generic over specialized.** vLLM consciously trades the last 10–15% of NVIDIA-only perf for portability — same code path on AMD, TPU, Neuron, Gaudi. TRT-LLM made the opposite trade and dominates the headline NVIDIA benchmarks.
- **Day-zero model support is a feature, not a side effect.** vLLM ships new model architectures (DeepSeek V4, Qwen3-Next, GPT-OSS, Granite 4.1 Vision) within days of the model card landing. TRT-LLM typically lags by weeks because new models need engine-build paths and kernel coverage.

### 3. Numerical / accuracy model

- **Default precisions:** BF16 for activations, model-dependent for weights. FP8 (E4M3) for both weights and KV is the common production setting on Hopper+; benchmarked accuracy delta versus BF16 is typically ≤1% on standard evals (MMLU, GSM8K, HumanEval), but workload-specific eval is mandatory before flipping the switch.
- **Quantization paths:** AWQ and GPTQ for weight-only INT4; SmoothQuant variants; BnB for QLoRA-style; **TurboQuant 2-bit KV cache** in 0.20.x for ~4× KV capacity, with a published accuracy report — useful for very-long-context workloads where KV dominates memory.
- **Determinism:** Greedy sampling with fixed model + fixed kernels is reproducible. Nucleus/temperature sampling is not. Continuous batching does **not** affect numerics — each request's forward pass is independent at the math level. Mixed-batch differences from kernel autotuning can introduce small bit-level drift; if you need bit-exactness across replicas, pin the autotuner.

### 4. Performance characteristics

- **Throughput scales with concurrency until KV memory or compute saturates.** On an 80GB H100 serving Llama-3-70B at FP8, you'll typically saturate KV before compute around concurrency 100–150 with 2K context.
- **TTFT vs ITL is the central tension.** Long prefill chunks crush ITL for in-flight decode; chunked prefill mitigates this; disaggregated P/D fully decouples it at the cost of operational complexity.
- **Prefix caching wins are workload-dependent.** RAG with a shared system/document prompt across users sees order-of-magnitude TTFT reductions; pure user-generated prompts see ~0.
- **SpecPrefill (ICML 2025)** — a draft-model-assisted sparse prefill — is a 2026 capability worth tracking: published up to 7.66× TTFT and 7× end-to-end QPS on Llama-3.1-405B-Instruct-FP8 with <5% accuracy loss. It composes with chunked and disaggregated prefill.
- **Scaling cliffs to watch:** (1) GIL pressure when scheduling thousands of concurrent very-short requests — run multiple engines + a router; (2) NCCL all-reduce latency on TP across PCIe vs NVLink — TP across PCIe is usually a mistake; (3) KV exhaustion on long-context workloads — turn on KV offload to CPU or aggressive eviction policies.

### 5. Operational model

- **Single-node:** `vllm serve <model>` is the entry point. Pip install on supported hardware; Docker images for NVIDIA and ROCm.
- **Multi-node / cluster:** Kubernetes via the official Helm chart and the `vllm-project/production-stack`. Common stacks pair vLLM with **LMCache** (KV cache backend with offload to CPU/SSD/remote), **llm-d** (request router with prefix-aware routing and P/D coordination), and a vector store for RAG.
- **Observability:** Prometheus metrics out of the box (`/metrics`); OpenTelemetry traces; the `--enable-server-load-tracking` flag for autoscaler signals. Less polished than TGI's metrics surface historically, but the gap has closed.
- **Model rollouts:** No engine compile step (unlike TRT-LLM), so swapping a model means restarting the server — fast in practice. Cold start under 90s for most models from local cache.
- **Common failure modes:**
  - **OOM on long contexts** → reduce `--max-model-len`, enable KV offload, switch to a smaller block size, or quantize KV.
  - **Throughput collapse under bursty load** → check `--max-num-seqs` and `--max-num-batched-tokens` budgets.
  - **Mysterious accuracy regression after upgrade** → kernel autotuner picked a different config; pin or re-tune.
  - **Disaggregated P/D KV transfer errors** → connector misconfig or interconnect bandwidth shortfall, not a vLLM bug.

### 6. Security and multi-tenancy

- **AuthN/AuthZ:** API-key gating via `--api-key` is provided but rudimentary; production deployments front vLLM with an API gateway that handles auth, rate limiting, and quota.
- **In-flight encryption:** TLS at the ingress; vLLM itself serves HTTP and expects you to terminate TLS at a sidecar/ingress.
- **Tenant isolation:** there is **no** in-process tenant isolation — all requests share the same KV cache pool, and **prefix caching means a tenant could in principle infer the existence of another tenant's prompt prefix via cache-hit timing**. If this matters, partition tenants across separate engine processes/replicas, or disable APC for sensitive paths.
- **Sandbox:** no built-in tool/code execution sandbox (vLLM is the model server, not an agent runtime). Tool calling is exposed; you sandbox the tool side yourself.

### 7. Ecosystem and integrations

- **Model coverage:** Llama 3/4, Mistral/Mixtral, Qwen 2/3 + Qwen3-Next, DeepSeek V2/V3/V4, Gemma 3/4, Granite 4, Phi 4, GPT-OSS, Hunyuan v3, plus VLMs (Llava-OneVision, InternVL, Qwen-VL, Gemma Vision, Granite Vision).
- **Frameworks that wrap or embed vLLM:** Ray Serve LLM, Anyscale, Modal, RunPod, BentoML, KServe, OpenLLM, LangChain (`langchain-vllm`), LlamaIndex, Triton (vLLM backend), TGI (multi-backend mode), SkyPilot.
- **Hyperscaler equivalents/managed:** AWS SageMaker (vLLM containers), GCP Vertex (community-supported), Azure ML (community), Databricks Model Serving (uses vLLM under the hood for several model families).
- **Adjacent OSS:** **LMCache** (KV cache layer), **llm-d** (Kubernetes-native distributed inference router), **production-stack** (reference deploy).

### 8. When to pick vLLM — and when not to

Pick it when:

- You serve **many models** or expect to swap models frequently — vLLM lands new architectures fastest.
- You run on **mixed hardware** (NVIDIA + AMD, or NVIDIA + TPU/Neuron) and want one engine.
- You want **OSS, Apache 2.0**, no vendor compiler in the loop.
- You need **tight-feedback experimentation** — short iteration time, no engine builds, OpenAI-compatible API.

Skip it (or pair it) when:

- You are **NVIDIA-only on a fixed model** and need every last percent of throughput → **TensorRT-LLM**.
- Your traffic is **prefix-dominated** (heavy RAG with shared documents, multi-turn agent traces) → **SGLang**, or vLLM with APC and llm-d's prefix-aware router.
- You are **deeply HF-native** and want HF's ops surface and they will use you as a TGI multi-backend → **TGI v3**.
- You need **CPU/edge** inference → **llama.cpp** / **Ollama** / **MLX** are different category, not vLLM.
- Your workload is **a single model, single SKU, hundreds of replicas** with stable traffic — TRT-LLM's per-model engine compile cost amortizes; vLLM's flexibility tax stops paying back.

### 9. Closing TL;DR

vLLM in 2026 is the safe default for self-hosted LLM serving. PagedAttention + continuous batching are now industry-standard ideas — what keeps vLLM in the lead is community velocity (day-zero models, fast hardware enablement) and breadth (NVIDIA + AMD + TPU + Neuron + Gaudi under one engine). It is not the throughput champion on NVIDIA-only fixed-model workloads (TensorRT-LLM is) and not the prefix-cache champion on RAG/multi-turn (SGLang is), but it is the only engine you can confidently standardize an entire fleet on without painting yourself into a vendor or workload corner.

## Sources

- [vLLM GitHub](https://github.com/vllm-project/vllm)
- [vLLM releases](https://github.com/vllm-project/vllm/releases)
- [vLLM docs (latest)](https://docs.vllm.ai/en/latest/)
- [PagedAttention design — official docs](https://docs.vllm.ai/en/latest/design/paged_attention/)
- [Automatic Prefix Caching — official docs](https://docs.vllm.ai/en/stable/design/prefix_caching/)
- [Disaggregated prefilling — official docs](https://docs.vllm.ai/en/latest/features/disagg_prefill/)
- [Optimization & tuning — official docs](https://docs.vllm.ai/en/stable/configuration/optimization/)
- [Disaggregated Inference at Scale with PyTorch & vLLM (PyTorch blog)](https://pytorch.org/blog/disaggregated-inference-at-scale-with-pytorch-vllm/)
- [vLLM × AMD ROCm attention backend (vLLM blog, Feb 2026)](https://blog.vllm.ai/2026/02/27/rocm-attention-backend.html)
- [SGLang GitHub](https://github.com/sgl-project/sglang)
- [SGLang vs vLLM benchmark (Particula, 2026)](https://particula.tech/blog/sglang-vs-vllm-inference-engine-comparison)
- [vLLM vs TensorRT-LLM vs SGLang H100 benchmarks (Spheron, 2026 — directional)](https://www.spheron.network/blog/vllm-vs-tensorrt-llm-vs-sglang-benchmarks/)
- [TensorRT-LLM GitHub](https://github.com/NVIDIA/TensorRT-LLM)
- [TensorRT-LLM release notes](https://nvidia.github.io/TensorRT-LLM/0.19.0/release-notes.html)
- [TGI v3 overview (HF docs)](https://huggingface.co/docs/text-generation-inference/en/conceptual/chunking)
- [TGI multi-backend support (HF blog)](https://huggingface.co/blog/tgi-multi-backend)
- [Comparative analysis of vLLM and TGI (arXiv 2511.17593)](https://arxiv.org/abs/2511.17593)
- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System (Aleksa Gordić)](https://www.aleksagordic.com/blog/vllm)
