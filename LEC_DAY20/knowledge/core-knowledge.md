# Core Knowledge

**Source:** `stream.pdf` — "Model Serving & Inference Optimization" (AICB-P2T2 · Ngày 20 · Chương 4: Hạ Tầng, VinUniversity Phase 2 Track 2, Tuần 4). 59 PDF pages / 53 content slides (slide numbering "X/53"). Confirmed AI Infrastructure track content (Day 16+ specialization), and confirmed as the correct Day 20 deck matching Day 19's forward-reference (§ "Bài tiếp theo" of Day 19 → "Model Serving & Inference Optimization").

---

## 1. Core Concepts

### Latency & throughput vocabulary (p.4)
- **TTFT (Time To First Token)** — time from request to first output token; depends on prefill compute + queue wait time. (explicit)
- **TPOT (Time Per Output Token) / ITL** — steady-state spacing between output tokens. (explicit)
- **E2E Latency** = TTFT + TPOT × (N − 1); SLOs typically defined at P95/P99. (explicit)
- **Throughput** — tokens/s for the whole system at saturation, with no SLO constraint. (explicit)
- **Goodput** — requests/s that satisfy the TTFT+TPOT SLO jointly; the primary production metric (explicit — "metric production quan trọng nhất"). Throughput @saturation ≠ Goodput @SLO; reporting throughput alone while ignoring SLO is called out as misleading.
- **Queue Depth** — requests waiting for prefill; a saturation indicator. Target 100–2,000; >2,000 signals saturation (p.25).

### Pre-LLM serving stack vs LLM serving era (p.5)
- Pre-LLM stack (2017–2022): TF Serving, Triton Inference Server, ONNX Runtime, TorchServe, BentoML — static batching, contiguous KV memory, no token streaming, no continuous batching.
- Why insufficient for LLMs: memory fragmentation (KV cache needs contiguous blocks → 60–80% VRAM waste), static batching adds 200–500ms latency, no streaming, one long request blocks the whole queue, fixed-length I/O can't handle variable-length generation.
- **The Shift (Jun 2023)**: vLLM PagedAttention — KV cache via non-contiguous virtual-memory-style pages + continuous batching → start of "LLM serving era"; 24× throughput vs naive HF Transformers.

### Quantization formats (p.6–9)
- **FP16/BF16** — baseline precision, no quality loss (reference point).
- **FP8** — <1% accuracy drop, ~2× memory reduction vs FP16; native from Hopper onward.
- **AWQ 4-bit** — activation-aware weight quantization: scales salient weights before INT4 rounding → better accuracy than GPTQ at same bit-width; ~1pt MMLU drop on 8B+ models, notably worse on <7B models.
- **GPTQ 4-bit** — inverse-Hessian layer-by-layer quantization (128 calibration samples); slow to quantize, fast at inference.
- **NF4 (bitsandbytes)** — 4-bit NormalFloat + double quantization, suited to normally-distributed weights; used with QLoRA for single-GPU fine-tuning.
- **GGUF k-quant (Q4_K_M, Q2_K, etc.)** — Q4_K_M mixes Q4/Q6 per tensor ("k" = quant method, "m" = medium size); CPU-only / edge recommended format; Q2_K is "last resort" for extreme constraints.
- **NVFP4 (Blackwell)** — 3.5× memory vs FP16, 1.8× vs FP8, <1% loss; requires Blackwell Tensor Core gen 5.
- **MXFP4** — OCP open standard 4-bit format (E8M0/32 values), portable across NVIDIA/AMD; gpt-oss ships in MXFP4 for portability, not accuracy.
- **NVFP4 vs MXFP4**: both 4-bit E2M1 element format but different scale schemes (NVFP4 = E4M3/16 values + FP32 tensor-level scale — smaller blocks, more accurate, more scale overhead; MXFP4 = power-of-2 scale). (explicit, p.8)
- **KV cache quantization** — separate lever from weight quantization; memory savings scale with context length × concurrency (bigger payoff at long context). **TurboQuant** (Google, arXiv 2504.19874): random rotation + QJL 1-bit calibration; 3.5 bit/channel ≈ quality-neutral, 2.5 bit = slight degradation.
- Selection guidance (explicit, p.6–7): Production Hopper → FP8/AWQ 4-bit; Production Blackwell → NVFP4 (default); Max quality → BF16/FP16; Edge/laptop → GGUF Q4_K_M; NVIDIA-only + accuracy-critical → NVFP4; cross-platform/gpt-oss → MXFP4; extremely constrained → GGUF Q2_K.
- Warning: quantizing both weights and KV cache simultaneously on small reasoning models can harm reasoning quality — quantize weights first, KV second, and re-measure rather than enabling both at once (p.8).

### KV Cache & attention mechanisms (p.9–11)
- **PagedAttention (vLLM)** — treats KV cache like OS virtual-memory pages (non-contiguous); eliminates the fragmentation waste of contiguous allocation; 24× vs naive HF Transformers; dynamic memory allocation.
- **RadixAttention (SGLang)** — KV reuse via a radix tree for prefix sharing; ideal for RAG, multi-turn, agents.
- **Attention head variants and KV memory**: MHA (baseline, KV 1×) → GQA (KV 4× less, used in LLaMA-2/3) → MQA (KV 8× less, PaLM/Falcon) → **MLA (Multi-head Latent Attention)** (KV 10× less, DeepSeek-V3). MLA compresses Q/K/V into a small latent vector before attention; kernels: FlashMLA, CutlassMLA, FlashInfer (2025). Enables longer context on same VRAM.
- **Long-context stack**: YaRN (RoPE interpolation, no retrain needed), StreamingLLM (attention sinks, near-infinite context), Jamba (AI21 Labs, SSM/Transformer hybrid, 256K context), FA3+MLA backend kernels.
- **Hybrid/SSM attention (2026 models: Qwen3-Next, Kimi Linear, MiniMax, Nemotron)** — interleave O(N²)/KV-O(N) attention layers with O(N)/state-O(1) SSM/linear layers; state is fixed-size and updated in-place, so KV/request doesn't grow with context. SGLang uses two memory pools (Mamba pool per-request, KV pool per-token).
- **Sparse attention — DeepSeek DSA (V3.2)**: "lightning indexer" selects top-k KV blocks before attention, reducing complexity from O(L²) to O(Lk); reported >50% API price reduction.
- Three techniques together (MLA compression, linear/SSM constant-KV, DSA sparse-read) are framed as three separate attacks on the same memory-bandwidth wall (inferred synthesis from p.11 text, stated explicitly as a summary line).
- Complication: in-place state (hybrid models) breaks assumptions of prefix caching (no rollback → needs MambaRadixCache with two separate LRU pools), speculative decoding (draft rejection can't roll back state → each draft token needs its own slot), and P/D disaggregation (state must transfer atomically as a whole block, not paged) (p.11).

### Speculative decoding family (p.12–16)
- **Speculative decoding (general mechanism)**: a draft model proposes 4–8 tokens; the target model verifies them in parallel in one forward pass. Lossless/distribution-preserving via rejection sampling — output distribution is identical to non-speculative decoding.
- **EAGLE-3** (NeurIPS'25) — draft at feature space + draft tree; 3.0–6.5× speedup, +20–40% over EAGLE-2; still the newest EAGLE version as of the lecture (no EAGLE-4).
- **DeepSeek MTP (Multi-Token Prediction)** — ~1.8× speedup, 85–90% acceptance (DeepSeek's own eval).
- **Lookahead/NGRAM** — self-drafting without a separate draft model (no training/VRAM cost); built into vLLM/SGLang/TRT-LLM.
- **Acceptance Length (AL, τ)** — average tokens accepted per verification round; the real comparison metric (not "N× speedup" claims).
- **Block size γ** — number of tokens the drafter proposes per round.
- **Why autoregressive (AR) drafting plateaus**: drafting K tokens = K sequential forward passes of the drafter → draft latency scales linearly with K, while AL gains diminish → practical optimum around K≈3.
- **P-EAGLE / DFlash (2026, parallel drafting)** — draft an entire block in one forward pass instead of K sequential passes. P-EAGLE peaks at K=7, up to 1.69× faster than EAGLE-3 (gpt-oss-20B, B200). New cost: loss of inter-token dependency within a block → "suffix decay" (later tokens in block more likely rejected).
- **DFlash** — block-diffusion drafter (non-causal attention, generates whole block in one forward pass) + **KV injection** (target's hidden state injected directly into drafter's KV cache, avoiding re-encoding context). Reported >6× lossless speedup, up to 2.5× vs EAGLE-3 (arXiv 2602.06036); ablation shows the diffusion mechanism itself (not KV injection) drives the win via lower draft latency, not better draft quality.
- **DSpark (2026)** — semi-autoregressive + confidence scheduling. **Markov head**: low-rank logit bias (r=256) restoring inter-token dependency to fix suffix decay (cost: +0.2–1.3% latency; rank 0 degrades to DFlash). **Confidence head**: predicts per-position accept probability, enabling **confidence-scheduled verification** — per-request verify-length trimmed by estimated survival probability rather than verifying a fixed block length for every request (treats batch capacity as a schedulable resource). Reported +26–31% AL vs EAGLE-3, +16–18% vs DFlash; DeepSeek-V4 live traffic: 60–85% faster per-user vs MTP-1 baseline at equal throughput, or +51–52% total throughput at moderate SLA.
- **Selection guide (p.16)**: NGRAM/Lookahead — zero training/VRAM, good for repetitive output (code/JSON). MTP/NEXTN — when model ships with an MTP head. EAGLE-3 — most mature/widest support, latency-first small batches. P-EAGLE — parallel version of EAGLE-3 architecture. DFlash — lowest draft latency, high-interactivity needs. DSpark — highest AL, concurrency-heavy serving.
- Three distinct cost categories addressed by different techniques: (1) draft latency → parallel drafting; (2) host-device sync overhead → SGLang overlap scheduler (default since v0.5.16; measured +33% on one config); (3) verify waste → confidence scheduling.
- Warning: DSpark exists in the `speculators` library but has no corresponding flag in SGLang yet — check engine support before promising it to a client (p.16).

### Continuous (in-flight) batching (p.12)
- Static/legacy batching waits to fill a batch → +200–500ms padding latency.
- Continuous batching: requests enter/exit at every step, no padding → ~5× latency reduction.
- Token streaming lets clients receive tokens incrementally, improving perceived TTFT.
- Terminology note: vLLM's "continuous batching" = TRT-LLM's "in-flight batching"; SGLang uses piecewise CUDA graphs for variable-length batches.

### FlashAttention & compilation (p.17–18)
- **FlashAttention lineage**: FA1 (Dao 2022, NeurIPS'22) — tiling + SRAM, O(N) memory, 3–4× speedup, core insight is IO-awareness (standard attention does 3 HBM round trips; FA tiles Q/K/V into SRAM with online softmax, writes output once). FA2 (2023) — sequence parallelism, 2× FA1. FA3 (Hopper, 2024) — TMA async pipeline + FP8/warp specialization. FA4 (Blackwell, arXiv Mar'26) — 2-CTA design, ~1613 TFLOPs BF16; shipped in vLLM v0.27 (Aug'26) with FP8 KV on SM100.
- **FlexAttention** (PyTorch 2024) — `torch.nn.attention.flex_attention` with BlockMask API; expresses causal/sliding-window/document-boundary/prefix+causal patterns; compiles via `torch.compile` to Triton kernels without custom CUDA.
- **torch.compile** — captures the computation graph via `torch.fx`, TorchInductor generates fused Triton kernels; `mode="max-autotune"` (slow compile, fast run) vs `mode="reduce-overhead"` (removes Python overhead); `dynamic=True` avoids recompilation on variable shapes. Speedup: 1.1–1.5× on LLM decode.
- **CUDA Graphs** — record one forward pass's GPU command stream, replay it to skip Python overhead on every subsequent step (saves 0.5–2ms/step); ideal for LLM decode (same ops repeated N_tokens times). vLLM v1 uses CUDA graph replay for decode, eager execution for prefill (variable shape); SGLang's "piecewise CUDA graph" handles mixed static/dynamic batch sizes. Reported +10–20% throughput stacked on top of other optimizations.
- **TensorRT compilation** — ONNX → layer fusion → kernel selection → FP8/INT8 calibration → `.trt` engine; 3–5× vs vanilla PyTorch; compile time 5–30 min/model; used inside TensorRT-LLM and NVIDIA Triton Inference Server.

### Serving engines landscape 2026 (p.19)
Eight engines compared: **vLLM** (PagedAttention, Auto Prefix Cache + chunked prefill; general LLM production), **SGLang** (RadixAttention, structured generation, MLA backend; multi-turn/chat), **NVIDIA Dynamo** (disaggregated P/D orchestrator, GA 1.0; multi-tenant cloud), **llm-d** (K8s-native, KV-aware routing; production at scale), **LMDeploy** (TurboMind engine; high throughput), **TensorRT-LLM** (NVIDIA-native FP8/FP4 optimized; NVIDIA GPU fleets, Triton backend), **Ollama** (wraps llama.cpp, single-command; local dev/testing), **llama.cpp** (GGUF native, CPU+GPU mixed offload, Apple Metal; local/CPU/edge). Production guidance (explicit, p.19): vLLM v1 or SGLang for general production; llm-d/Dynamo for disaggregated scale; llama.cpp for local CPU/Mac; Ollama for containers; TensorRT-LLM for NVIDIA fleets. Note on volatility: both vLLM and SGLang moving frontend to Rust (Python GIL bottlenecks request-handling); flags/versions drift ~every 2 weeks — "teach the technique, treat the flag as disposable."

### Prefix caching (p.22)
- **vLLM v1 APC (Automatic Prefix Caching)** — on by default.
- **RadixAttention (SGLang)** — radix-tree cache; a hit skips prefill for the entire shared prefix.
- **HiCache** — 3-tier cache (GPU VRAM → host RAM → external store e.g. HF3FS/Mooncake), attach/detach without restart.
- Cross-instance sharing: LMCache, Mooncake global KV pool across a cluster.
- Reported savings: −70% TTFT on repeated system prompts.
- Pricing tier (industry-wide, explicit p.22): Anthropic cached read −90%, DeepSeek cache-hit ~98% off, OpenAI cached input −75%, Google cached read −90%. Framing: prefix caching is both an engineering optimization and a pricing lever; cache misses are billed at full price, so monitor cache-hit rate.

### Attention backend selection (p.23)
Hardware auto-selects the attention kernel: H100/H200 (Hopper) → FA3; B200 (Blackwell) → trtllm_mha or FA4; A100/A40 → FlashInfer; DeepSeek V3/R1 MLA → FlashMLA (page=64); ROCm/Ascend/CPU → Triton (cross-platform fallback). MLA-specific backends (FlashMLA, CutlassMLA, FlashInfer-MLA, TRTLLM-MLA) give 3.1× throughput vs MHA and 10× less KV memory. Override only for benchmarking/debugging via `--attention-backend`.

### Structured generation (p.24)
- Grammar-constrained output formats: JSON Schema, EBNF, Regex, Pydantic models.
- Grammar backends: XGrammar (default, fastest), Outlines, Llguidance.
- Tool Parser (`--tool-call-parser`) supports 15+ model families; Reasoning Parser (`--reasoning-parser`) separates `<think>` content into `reasoning_content` vs `content`. For reasoning models, grammar constraints apply *after* the `<think>...</think>` block.

### Production tuning knobs (p.25–26)
- Memory: `--mem-fraction-static`/`--gpu-memory-utilization` (leave 5–8GB baseline headroom), `--chunked-prefill-size` (lower to 2048–4096 if prefill OOMs), `--max-running-requests`/`--max-num-seqs` (cap burst to avoid decode OOM).
- Scheduling aggressiveness: `--schedule-conservativeness` (0.3 aggressive · 1.0 default · 1.3 conservative).
- Observability: `--enable-metrics` (Prometheus endpoint) → Grafana; key metrics = num_running_reqs, num_queue_reqs, TTFT/TPOT histograms, cache-hit rate; `--log-requests` + rolling crash dump buffer + replay tool for debugging.
- **Request scheduling policy**: both vLLM (`--scheduling-policy fcfs|priority`) and SGLang (`--schedule-policy`, default fcfs, plus `lpm`/priority/lof/routing-key) default to FCFS; cache-aware scheduling (`lpm` = longest-prefix-match) must be explicitly enabled. On KV exhaustion, vLLM v1 defaults to RECOMPUTE preemption (cheaper than SWAP). Research-stage (not shipped): VTC fairness (OSDI'24), SJF via predicted output length (ELIS), priority-based preemption of running requests (vLLM feature request #40004, not shipped).

### Disaggregated Prefill/Decode serving (p.27)
Splits prefill and decode onto separate GPU pools connected by KV transfer (NVLink/IB, ~10GB/s overhead) instead of running both phases on the same GPU (monolithic vLLM v0, where prefill contends with decode). Systems: NVIDIA Dynamo 1.0 (GA 2026, cross-engine orchestrator, KV-aware router, NIXL), Mooncake (Kimi, FAST'25, 100B+ tok/day, RDMA zero-copy global KV pool), llm-d (K8s-native, vLLM + Gateway API + NIXL, scale-to-zero); foundational papers DistServe (OSDI'24), Splitwise (ISCA'24). Benefit is clear for prefill-heavy workloads (long context, RAG); not worth it for short unique prompts.

### Multi-LoRA serving (p.28)
Serves N adapters (e.g., SQL/Med/Code) on one base model + one GPU instance instead of N separate deployments. Mechanisms: Punica/SGMV fused batched-adapter GEMM kernels; S-LoRA (paged LoRA weights, per-request swap); `vllm --enable-lora`; SageMaker LMI-Dist managed hosting; SGLang Chunked SGMV (20–80% latency↓) and LoRA overlap loading (35% TTFT↓). Reported 12× throughput vs N separate single-model servers; overhead ~+2ms/token for adapter application.

### Expert Parallelism / MoE scaling (p.29)
Splits MoE expert weights across GPUs (no replication). Forward pipeline: dispatch → pre-permute → core runner → combine. All-to-all (A2A) backends: deepep (NVLink/IB), mooncake, nixl. MoE runner backends: deep_gemm or cutlass. Constraint: most backends require `ep_size = tp_size`. Key optimizations: **Two-Batch Overlap (TBO)** interleaves A2A communication and GEMM compute across 2 micro-batches, hiding all-to-all latency (+27–35% prefill throughput, −50% peak memory, SGLang); **EPLB** load balancer reduces GPU utilization variance. DeepSeek-V3/R1 671B production config: prefill EP32 / decode EP144 + DeepEP + EPLB (~$0.20/1M output tokens).

### Data Parallelism & DPA (p.30)
- **DP** — replicates the whole model + KV cache (memory duplication).
- **DPA (DP Attention)** — replicates only the attention component; MoE/FC layers share via Expert Parallelism, so KV cache is not duplicated. Combined with MLA, enables larger batch sizes with significant VRAM savings (DeepSeek V3/R1).
- **sgl-router (cache-aware load balancing)** — Rust-based production router that sends requests to the instance with the best-matching KV prefix cache; benchmark (8×A100 80GB): +92% throughput, cache hit 20%→75% (+275%).
- DeepSeek-V3 production combo: DP+EP with `--enable-dp-attention`, `--tp 8 --dp-size 8 --ep 8` → +92% throughput.

### Distributed parallelism strategy (p.31–32)
Comparison table: **DP** (splits requests; multi-user replicated model; tradeoff: no KV cache sharing), **TP** (splits weights/layers; large model single-node; tradeoff: all-reduce sync every layer), **PP** (splits layers; multi-node/128K+ context; tradeoff: pipeline bubble latency, micro-batching), **EP** (splits MoE experts; Mixtral/DeepSeek-V3 671B; tradeoff: expert routing overhead), **Disaggregated P/D** (splits prefill/decode; long RAG/prefill-heavy; tradeoff: KV transfer bandwidth cost).
- Multi-node vLLM via Ray cluster: `ray start --head`/`ray start --address=...`, `--tensor-parallel-size`/`--pipeline-parallel-size`, NCCL collectives, `NCCL_IB_HCA=mlx5` for InfiniBand.
- Placement rules: TP within a node (NVLink 900GB/s, all-reduce not a bottleneck); PP across nodes (P2P activation transfer tolerates IB latency); don't do TP across nodes unless on an NVLink fabric (NVL72/GB200); EP holds expert subsets per GPU with on-demand A2A routing.
- Rule of thumb (explicit): TP ≤ GPUs/node; PP = number of nodes; EP for MoE.
- **Pipeline Parallelism deep dive**: chunked prefill lets nodes process token chunks concurrently, reducing TTFT for long context. `--enable-dynamic-chunking` auto-adjusts chunk size to prefix length (smooth factor env var, default 0.75, range 0.6–0.85). Example optimal chunk sizes: DeepSeek-V3.1 4K fixed/12K dynamic; Qwen3-235B 6K fixed/18K dynamic. Piecewise CUDA Graph automatically disables when PP is enabled. Use PP when the model is too large for single-node TP, or for 128K+ context needs.

### Serving regimes 2026 — specialized workloads (p.33–39)
- **Agentic serving**: measured workload (TraceLab, arXiv 2606.30560; ~4,300 coding-agent sessions, ~350K LLM steps, ~430K tool calls from Claude Code/Codex) shows long autonomous loops, long context with short output, heavy-tailed tool-call distribution (e.g., Codex trace turn 30 ≈ 80K token context, input:output ≈ 131:1). Prefix-cache hit is high but imperfect. Why caches break: capacity limits (100K context = GBs of KV, one instance can run out), cross-instance miss (load balancer routes session to an instance without the cached prefix → full recompute), evict-on-completion is wrong for agents (they return after tool calls), and "tool gaps" (pauses between turns, sometimes human-paced). Fix: shared KV pool (e.g., vLLM × Mooncake Store with GPUDirect RDMA); reported Codex trace result on 12×GB200: 3.8× throughput, 46× P50 TTFT reduction, 8.6× E2E reduction, cache hit 1.7%→92.2%. Shared pools convert cache locality from a routing problem into a storage problem — affinity routing breaks under autoscale, but a good pool tolerates round-robin.
- **Reasoning model serving**: long reasoning chains push inference from compute-bound prefill into a capacity-bound regime (ISCA'26, arXiv 2605.19775). DP's capacity trap: fragmented KV causes early admission throttling while compute sits idle; TP reclaims wasted memory with sublinear benefit around ~32B threshold. Dense models (e.g., Llama-405B) favor higher-order TP; MoE (e.g., R1) needs a hybrid strategy due to routing/sync overhead. Four production symptoms: memory volatility (KV swings within a single run), stragglers (one long-"thinking" request blocks the batch), unpredictable latency (output-sequence-length unknown in advance → can't schedule on it), domain-dependent behavior. Counter-intuitive: quantization and speculative decoding help reasoning models, but prefix caching and KV quantization can hurt accuracy/throughput on small reasoning models (arXiv 2510.18672) — no optimization is universally safe. Chat models size KV by p95(ISL+OSL); reasoning models have an uncontrolled long-tail OSL → admission control + token budgets are mandatory.
- **Serving for RL (rollout)**: RLHF/GRPO generates trajectories using the serving engine itself (vLLM/SGLang), not the training backend (FSDP/Megatron) — so RL pipelines separate trainer from inference. Colocated (train+infer share GPU) saves GPUs but must yield memory each round; disaggregated avoids that tradeoff. Real bottleneck is weight sync per step, not token generation speed; engines that understand LoRA push only the adapter delta (sub-millisecond), not the full parameter set. vLLM mechanism: **sleep mode** (`--enable-sleep-mode`) — level 1 moves weights to CPU RAM and drops KV; level 2 also drops weights (for weight updates); frees >90% VRAM without shutting down the server. Endpoints: `/sleep?level=1`, `/wake_up`, `/is_sleeping`; partial wake via tags (e.g. `tags=["weights"]`). Weight transfer API lets a trainer push new weights into the engine; partial rollout preserves in-progress samples during sync to avoid wasting GPU time on stragglers. Explicit division of labor: Day 20 owns the serving side (rollout engine, sleep/wake, weight transfer, elasticity/ROSE arXiv 2605.06534); RL algorithms (PPO/GRPO/DPO) belong to Days 21–22.
- **Diffusion LLM (dLLM) serving**: generates a whole block in parallel then denoises it, rather than one token per forward pass — TPOT is no longer a natural unit; block size becomes the quality↔latency knob; number of denoising steps is a new quality/speed knob with no AR equivalent. Built by initializing from an existing AR model and continuing training with a diffusion objective (not trained from scratch). Commercial examples: Mercury (Inception) — first commercial dLLM, >1000 tok/s on H100; Mercury 2 (13/08/2026) claims "10× faster," $0.25/$1.00 per 1M in/out (vendor-reported numbers, flagged as unverified/self-reported in the source). LLaDA 2.0 (arXiv 2512.15745) converts an existing AR model to dLLM via a 3-phase schedule; mini 16B / flash 100B, MoE, open-source. Connects back to §3: diffusion already appears in the stack as a drafter (DFlash) before appearing as a primary model. Consequence: PagedAttention/continuous-batching assumptions (KV growing by exactly one token per step) don't hold for dLLM — don't directly reuse the capacity-sizing formulas from §9 (p.36).
- **Multimodal (VLM) serving**: image-token explosion — one 1024² image ≈ 1,100–4,100 tokens (Qwen3-VL ~1,139, Pixtral 4,096); video at 30fps ≈ 350K visual tokens/minute (pre-compression). TTFT is now primarily a function of number of images, not output length. Vision encoder (ViT) doesn't benefit from TP — actually slows down at TP=8. **EPD (Encode-Prefill-Decode) disaggregation** (2026): splits into 3 independently-scalable phases; SGLang "2E1P" gives ~6–8× TTFT reduction on Qwen3-VL-235B; vLLM v0.11+ `--mm-encoder-tp-mode data` for encoder disaggregation; CPU AMX can encode in parallel with GPU prefill/decode. Multimodal prefix caching hashes pixel/image-embeddings (SHA-256) — reported cache hit reduces TTFT 18s→1s (LMCache). Early-fusion architectures (Llama 4) eliminate the separate encoder stage entirely.
- **Embedding & reranker serving**: prefill-bound — single forward pass, no KV cache, no decode loop; throughput achieved via large static (token-sorted) batches, not continuous batching. Cross-encoder rerankers score (query, doc) pairs and are heavier than bi-encoder embedding models. FP8 gives ~50% throughput increase at >99% cosine similarity retention. Stack: HF TEI (Rust, serves both embedding+reranker), Snowflake Arctic Inference (16× vLLM via disaggregated tokenize + FP8); models: Qwen3-Embedding-8B (MTEB rank 1), BGE-M3 (dense+sparse+ColBERT); MRL allows flexible embedding dimension truncation; late chunking preserves context. Self-hosting break-even vs API (OpenAI/Voyage) is roughly 50–100M tokens/month.
- **Semantic caching (3-layer cache stack)**: (1) semantic cache (meaning-based match; hit = 100% compute saved), (2) prefix/KV cache (hit skips prefill for shared prefix), (3) full inference (complete miss). Semantic cache = embed prompt → vector search → return cached response if similarity > threshold. Real hit rates: 30–68% for FAQ/support workloads, 10–25% for open-ended queries — "95%" hit-rate claims are flagged as marketing exaggeration. vCache (ICLR'26) uses adaptive per-prompt thresholds with error bounds. Products: AWS ElastiCache+Bedrock, Azure APIM llm-semantic-cache. Risks: cache poisoning (NDSS'26, ~90% attack success reported) and KV timing side-channels — mitigation is careful threshold tuning + per-tenant cache salting.
- **Model routing & cascades**: **Routing** (pre-generation) uses a classifier to pick the model before generation runs. **Cascade** runs a cheap model first and defers to a stronger model only when confidence is low. RouteLLM (ICLR'25): −85% GPT-4 cost at 95% of MT-Bench quality. FrugalGPT: cascade matches GPT-4 quality at −98% cost. Production 3-tier pattern: nano (~$0.10/M) classifies → mid ($1–3/M) drafts → frontier ($10–15/M) handles the hard tail; reported −60–80% cost at <5% added routing latency. Production systems: Azure AI Foundry Model Router (GA), OpenRouter Auto. 2026 trend (explicit): pre-generation routing is preferred over cascading, because cascading pays for the cheap model's generation before deciding to defer. Framing: routing is described as the single largest cost lever at the serving layer; reasoning models cost 13–25× more energy per query, so routing "easy" queries to small models matters.

### Power & energy (p.41)
- **Power wall**: GB200 NVL72 draws 120–132kW/rack, GB300 135–150kW, Vera Rubin NVL144 ~190kW. IEA forecast: datacenter electricity use ~485→950 TWh (2025→2030).
- **Tokens-per-joule** is now a first-class benchmark metric (MLPerf Power v5.1). Median real-world energy ~0.31 Wh/query — flagged as far lower than older inflated estimates (4–20× overestimate).
- Energy levers: FP8 ≈ −30% energy at batch ≥64; FP4 gives 25–50× vs H100 FP16; MoE sparsity (GPT-OSS-20B) −26% energy/1K tokens vs a dense 32B model; GreenLLM phase-specific DVFS gives −10–34% energy at <3.5% SLO miss rate; carbon-aware temporal shifting can offset up to ~70% carbon.
- Reasoning models use ~15× more tokens → median energy rises from 0.31 to 3.91 Wh/query (13×). Explicit framing: power, not FLOPs, is the binding 2026 scaling constraint.

### Confidential inference (p.42)
- GPU TEEs (Trusted Execution Environments) encrypt data even while it's being computed on ("in-use" encryption), not just at rest/in-transit.
- Hopper PPCIE (8-GPU HGX, 2025) leaves NVLink traffic in plaintext; Blackwell adds NVLink encryption + TEE-I/O (first multi-GPU-encrypted generation).
- Overhead: <9% throughput on large models (~0% on Llama-70B), ~19% TTFT.
- Inference-layer attacks: **PROMPTPEEK** (NDSS'25) — KV timing side-channel reconstructs prompts with 99% success by probing TTFT on shared Automatic Prefix Cache; Stanford ICML'25 study found 7/8 caching APIs studied shared cache cross-user. Mitigation: per-tenant cache salting (vLLM), SafeKV. ZK-proof inference remains 10⁴–10⁵× slower — not practical yet.
- Use case: regulated industries (healthcare, finance, government) — now feasible given <9% overhead; should be paired with cache salting to block cross-tenant KV leakage.

### Auto-scaling (p.43)
Architecture: clients → load balancer (least-busy routing) → GPU pool (auto-scaled). Triggers: scale out at GPU util >80% or queue depth >10; scale in at GPU util <30%. KEDA + Knative provide event-driven autoscaling including scale-to-zero. Optimizations: least-busy routing (+30% vs round-robin), request batching with a 50ms window (+40% throughput), warm pools of idle instances for spikes. Core tradeoff: cost vs. cold-start latency.

### Cold start & weight loading (p.44)
Cold start in production can exceed 40 seconds vs ~30ms/token once warm. Scale-to-zero means the first request after idle absorbs that entire cold-start cost (or errors out). vLLM startup is a 6-step, mostly CPU-bound process (arXiv 2606.07362) — cold start is described as a computable/predictable engineering problem, not a mystery. Four levers, in order: (1) stream weights directly from object store to GPU, bypassing disk (Run:AI Model Streamer, native in vLLM/SGLang, reported up to 6× improvement); (2) format change — CoreWeave Tensorizer loads straight to GPU, ~53–60% of safetensors load time; (3) cache the compiled artifacts (torch.compile/CUDA-graph capture) across restarts; (4) avoid true scale-to-zero via warm pools or scale-to-zero-with-auto-wake. Rule (explicit): interactive-SLA workloads should keep ≥1 warm replica and pay for it; batch/dev/internal traffic is a good fit for scale-to-zero since no one is waiting. Warm-up time must be excluded from benchmark numbers.

### Edge deployment (p.45)
Two deployment paths from a PyTorch model: (a) ONNX export → TensorRT (NVIDIA) → GPU server, with FP8/INT8 calibration giving 3–5× speedup; (b) GGUF conversion (llama.cpp) → Ollama container → edge/laptop. GGUF quality/size levels: Q2_K (extreme, quality drop) < Q4_K_M (recommended) < Q6_K (near-lossless) < Q8_0 (max quality). Example models: Llama-3-8B, Qwen-3-8B, Phi-3-mini; one-command deployment via `ollama run llama3`.

### Hardware landscape 2026 (p.46)
Comparison table (chip / FP4-FP8 peak / memory / niche): NVIDIA H200 SXM (FP8 only, 141GB HBM3e, cost-effective baseline, +43% decode vs H100); NVIDIA B200 (9 PFLOPS FP4, 192GB HBM3e, GB200 NVL72 = 72-GPU NVLink domain); NVIDIA B300/GB300 (15 PFLOPS FP4, 288GB HBM3e, "Blackwell Ultra," current gold standard); NVIDIA Vera Rubin (50 PFLOPS NVFP4, 288GB HBM4, full production 01/06/26); AMD MI355X (20 PFLOPS FP4, 288GB HBM3e, B200-parity per MLPerf v6.0, GA Oct'25); Google TPU v7 "Ironwood" (4,614 TFLOPS FP8, 192GB HBM3e, powers Anthropic Claude); AWS Trainium3 (2.52 PFLOPS FP8, 144GB HBM3e, GA Dec'25, ~50% cost cut reported by Uber). Key point (explicit): HBM bandwidth, not FLOPs, is the real bottleneck; HBM supply for 2026 is sold out; memory bandwidth determines decode throughput. Frontier labs diversify silicon (Anthropic runs Claude on Google TPU v7 + AWS Trainium, not only NVIDIA). As-of-Aug'26 status note: Vera Rubin held its 50 PFLOPS NVFP4 compute target but NVIDIA lowered the HBM4 bandwidth spec from 22TB/s to ~20TB/s — compute targets landed on schedule, bandwidth targets did not.

### Capacity planning (p.47)
KV cache size per token formula: `2 × n_layer × n_kv_head × d_head × b` (factor 2 = K and V; b = bytes/element). Worked example: Llama-3.3-70B, 80 layers, GQA 8 KV heads, d_head=128, FP16 → 2×80×8×128×2 = 327,680 bytes ≈ 320KB/token → 8K context ≈ 2.5GB KV per request. VRAM budget = weights + KV + activations + overhead. Worked example: H200 (141GB), 70B model at FP8 (~70GB weights), ~10GB activations/overhead → ~61GB available for KV → 61/2.5 ≈ 24 concurrent 8K-context requests — presented as the concrete answer to "why can't I serve 100 users on one GPU." Levers that multiply that 24 upward: FP8 KV (×2), NVFP4 KV (×4 vs FP16), MLA replacing GQA (×2.7–4.7), linear/hybrid layers making KV O(1), halving context length (×2); prefix caching doesn't raise the hard ceiling but increases realized throughput. Note: PagedAttention reduces fragmentation waste to <4% (small but nonzero). Day 25 (GPU FinOps) covers cost; this lecture only answers "does it fit / how many users."

### Benchmarking methodology (p.48)
Numbers are meaningless without: ISL/OSL (input/output sequence length) reported (TTFT is an ISL story, TPOT is an OSL story); closed-loop (fixed N in-flight) vs open-loop (Poisson arrival) noted — closed-loop can't reveal queueing collapse; percentiles reported, not means (batching distorts the mean); warm-up (cold start + CUDA graph + compile) excluded, with engine version/flags/hardware documented. Common client-side pitfall: single-process asyncio benchmark clients create their own client-side queueing bottleneck (arXiv 2605.24217) — under an M/G/1 model, the GIL inflates measured TTFT/TPOT as request rate increases, meaning you may be benchmarking your own client, not the server; fix is a multi-process client + NTPOT metric. Tools: `vllm bench serve`, AIPerf, InferenceMAX; for spec decoding specifically, SPEED-Bench (NVIDIA, arXiv 2604.09557) shows random tokens inflate measured throughput ~23% when spec decoding is enabled, and low-entropy domains (code, math) show higher AL than roleplay — workload composition determines the benchmark result. Lab 20 uses locust with multiple worker processes.

### SLA / production best practices (p.49–50)
Example SLA dashboard targets: P50 <200ms, P95 <500ms, P99 <1000ms; example measured: P50 120ms, P95 380ms, P99 850ms, 1,800 tokens/s/GPU, 99.9% uptime (=8.7h downtime/year). Best practices: multi-AZ deployment + health checks; 10–60s timeout for LLM generation; circuit breakers with fallback under overload; graceful degradation (cached/shorter response, route to smaller model); spot instances for batch inference; scale-to-zero when idle (KEDA); right-size GPU choice (don't run a 7B model on an A100). Explicit principle: benchmark with locust/k6 before production — don't guess latency, measure it.

---

## 2. Relationships & Mechanisms

- **Throughput vs Goodput**: throughput is necessary but not sufficient — a system can have high throughput and low goodput if it violates SLOs; production reporting should default to goodput (p.4).
- **Pre-LLM stack → LLM serving era**: PagedAttention + continuous batching jointly solved the two structural problems (memory fragmentation, blocking on long requests) that made pre-LLM serving stacks unsuitable for LLMs (p.5, causal "shift" narrative).
- **Quantization as a dial, not a single choice**: precision (FP32→FP16→FP8→INT8/AWQ→NVFP4) trades VRAM for accuracy loss; weight quantization and KV quantization are independent, stackable levers, but stacking both aggressively can compound accuracy loss on small/reasoning models — sequence: quantize weights first, measure, then consider KV quantization (p.6–8).
- **KV cache growth → attention architecture innovation → hybrid/sparse pressure relief**: MHA→GQA→MQA→MLA is a chain of increasingly aggressive KV-memory reduction; when even MLA isn't enough, the response is architectural (SSM/hybrid layers with O(1) state) or algorithmic (DSA sparse top-k KV read) rather than further attention-head compression (p.9–11).
- **In-place state (hybrid models) breaks three downstream mechanisms**: prefix caching (no rollback), speculative decoding (draft rejection needs per-token slots), and P/D disaggregation (state must move as one atomic block, not paged) — this is presented as a systemic side-effect of the hybrid-attention design choice, not an isolated bug (p.11).
- **Speculative decoding evolution as Input→Process→Output**: Input = target model's need to reduce per-token latency → Process = draft cheap tokens (AR, then parallel/diffusion) → verify losslessly in one target forward pass → Output = accepted tokens appended, with acceptance length (AL) as the efficiency signal. The chain AR (EAGLE-3) → parallel (P-EAGLE/DFlash) → confidence-scheduled (DSpark) is presented as addressing three different bottlenecks in sequence: draft latency, then sync overhead, then verify waste (p.12–16).
- **Continuous batching depends on PagedAttention-style memory management**: non-contiguous KV allocation is a prerequisite that makes admitting/evicting requests mid-batch feasible without wasting memory (inferred connection between p.5 and p.12 material).
- **Compilation stack layering**: torch.compile (kernel fusion) + CUDA graphs (replay, skip Python overhead) + attention-backend selection (FA3/FA4/FlashInfer/FlashMLA) are presented as independently stackable optimizations, each contributing roughly additive throughput gains (p.17–18, p.23).
- **Prefix caching as both an engineering technique and an economic layer**: the same mechanism (skip recompute for a shared prefix) shows up as an engine feature (APC/RadixAttention) and as a vendor pricing tier (cached-token discounts) — the lecture frames these as two views of the same underlying capability (p.22).
- **Disaggregated P/D serving depends on / trades off against**: enables independent scaling of prefill vs decode pools, at the cost of KV-transfer bandwidth between pools; net benefit is workload-dependent (prefill-heavy → good; short unique prompts → not worth it) (p.27).
- **DPA + MLA combination**: DPA alone reduces KV duplication; combined with MLA's already-compressed KV representation, the two compound to allow much larger batch sizes on the same VRAM — explicitly demonstrated by DeepSeek V3/R1 production configs (p.30).
- **Parallelism strategy selection depends on hardware topology**: TP requires NVLink-class intra-node bandwidth (breaks down across nodes without NVLink fabric); PP tolerates higher inter-node (IB) latency, so PP is the multi-node lever and TP is the single-node lever; EP is orthogonal, driven by MoE architecture rather than by hardware interconnect (p.31–32).
- **Agentic serving cache-locality problem is reframed as a storage problem**: a shared KV pool changes the requirement from "route the request to the instance that has the cache" (routing problem, fragile under autoscale) to "any instance can reach the pool" (storage problem, tolerant of round-robin) (p.33).
- **Reasoning models shift the bottleneck regime**: from compute-bound prefill (pre-LLM/typical chat serving) to capacity-bound (KV memory) — this reframes which optimizations help: quantization/spec-decoding still help, but prefix-caching/KV-quantization can hurt small reasoning models, because the accuracy-sensitive long reasoning chain is more fragile to compression (p.34).
- **RL rollout uses the serving engine as the RL environment**: this is why serving-layer mechanisms (sleep mode, weight-transfer API, LoRA delta push) directly enable RL training efficiency — the serving engine's ability to free/reclaim VRAM (sleep mode) substitutes for provisioning a second GPU cluster for training (p.35).
- **Diffusion serving breaks core serving assumptions**: PagedAttention and continuous batching assume KV grows by exactly one token per forward-pass step; dLLMs violate this assumption (whole blocks generated/denoised at once), so capacity-sizing formulas built for AR models (§9, p.47) don't transfer directly to dLLM serving (p.36).
- **VLM serving reframes what drives TTFT**: for text-only serving TTFT is driven by prefill compute on prompt tokens; for VLM serving, TTFT is dominated by image-token count, which motivates disaggregating the vision encoder (ViT, doesn't benefit from TP) from prefill/decode (which do) (p.37).
- **3-layer cache stack ordering**: semantic cache sits above (checked before) prefix/KV cache, which sits above full inference — each layer trades a different kind of match precision (meaning-based vs exact-prefix vs none) for compute savings, with corresponding staleness/collision risk increasing the higher up the stack (p.39).
- **Routing vs cascading trade-off**: cascading always pays the generation cost of the cheap model before deciding to escalate; pre-generation routing decides before any generation cost is paid — this is given as the explicit reason 2026 practice favors routing over cascading (p.40).
- **Power constraint reframes the optimization target**: energy-per-token (tokens-per-joule) becomes a first-class metric alongside latency/throughput because power delivery (kW/rack), not FLOPs, is the binding constraint on data-center scaling — this changes which levers matter (FP8/FP4 precision, MoE sparsity, DVFS) versus a pure-latency framing (p.41).
- **Capacity planning → concrete formula → Input/Process/Output**: Input = model architecture params (layers, KV heads, d_head, precision) + hardware VRAM → Process = KV-per-token formula, then VRAM budget subtraction (weights, activations, overhead) → Output = max concurrent requests at a given context length; each of the "levers" (FP8/NVFP4 KV, MLA, hybrid layers, shorter context, prefix caching) plugs into this formula as a multiplier (p.47).
- **Benchmarking mechanism**: client-side single-process asyncio architecture can itself become the bottleneck (via GIL), meaning a naive benchmark measures the client, not the server — this is a methodological trap that invalidates naive throughput/latency numbers unless corrected with multi-process clients (p.48).

---

## 3. Examples & Distinctions

- **TTFT vs TPOT**: TTFT = latency to the first token (prefill + queue-bound); TPOT = steady-state per-token spacing (decode-bound). Distinguishing criterion: which phase of generation dominates the number (p.4).
- **Throughput vs Goodput**: similarity — both are tokens/requests-per-time metrics; difference — throughput ignores SLO constraints (measures saturation), goodput only counts requests meeting the TTFT+TPOT SLO. Distinguishing criterion: SLO-awareness (p.4).
- **Static batching vs continuous batching**: static waits to fill a batch (adds 200–500ms, one long request blocks the queue); continuous admits/evicts requests every step (no padding, ~5× latency improvement) (p.5, p.12).
- **GPTQ vs AWQ (both 4-bit weight quant)**: similarity — both are post-training weight quantization to ~4 bits. Difference — GPTQ uses inverse-Hessian layer-by-layer calibration (slow to quantize, fast to run); AWQ scales salient weights before rounding, giving better accuracy at the same bit-width (p.7).
- **NVFP4 vs MXFP4 (both 4-bit E2M1 formats)**: similarity — same 4-bit element format. Difference — NVFP4 uses smaller blocks + FP32 tensor-level scale (more accurate, more scale overhead, Blackwell-only); MXFP4 uses power-of-2 block scaling (OCP open standard, portable to AMD, chosen by gpt-oss for portability not accuracy) (p.8).
- **MHA vs GQA vs MQA vs MLA**: all are attention-head variants trading off KV memory vs modeling capacity; distinguishing criterion is the KV memory multiplier — MHA (1×, standard) > GQA (4× less, LLaMA-2/3) > MQA (8× less, PaLM/Falcon) > MLA (10× less, DeepSeek-V3, via latent-vector compression) (p.9).
- **PagedAttention vs RadixAttention**: similarity — both are KV-cache management techniques from the vLLM/SGLang lineage. Difference — PagedAttention solves memory fragmentation (paging non-contiguous KV blocks); RadixAttention solves prefix redundancy (reusing shared KV via a radix tree). They address different waste sources (p.9).
- **EAGLE-3 (autoregressive drafting) vs DFlash (block-diffusion drafting) vs DSpark (semi-AR + confidence)**: similarity — all are lossless speculative-decoding drafters. Difference — EAGLE-3 drafts K tokens via K sequential forward passes (latency scales with K); DFlash drafts a whole block in one non-causal forward pass (latency ~constant in block size, but suffers suffix decay from lost inter-token dependency); DSpark adds a Markov head to restore that dependency and a confidence head to schedule per-request verify length. Distinguishing criterion: how many forward passes the draft step costs, and whether inter-token dependency within the block is preserved (p.13–16).
- **vLLM "continuous batching" vs TRT-LLM "in-flight batching"**: explicitly stated as the same mechanism under different vendor names (p.12).
- **DP vs DPA**: similarity — both are data-parallel strategies (replicate for multi-user throughput). Difference — DP replicates the entire model + KV cache; DPA replicates only attention while sharing MoE/FC layers via EP, avoiding KV duplication (p.30).
- **TP vs PP**: similarity — both split a large model across multiple devices. Difference — TP splits weights/layers within a layer (needs NVLink-class bandwidth, used single-node); PP splits by layer across pipeline stages (tolerates higher inter-node latency, used multi-node or for very long context) (p.31–32).
- **Colocated vs disaggregated RL serving**: colocated (train+infer share GPU) saves hardware but must yield memory every round; disaggregated avoids that but needs separate GPU pools (p.35).
- **AR (autoregressive) LLM serving vs diffusion LLM (dLLM) serving**: similarity — both generate text token-by-token conceptually. Difference — AR generates one token per forward pass (TPOT-defined); dLLM generates a whole block per forward pass then iteratively denoises it (block size and denoising-step count are the new tunable knobs, TPOT isn't a natural unit) (p.36).
- **Embedding/reranker serving vs LLM decode serving**: similarity — both are neural-network inference serving problems. Difference — embedding/reranker is prefill-bound (single forward pass, no KV cache, no decode loop, large static batches); LLM decode is an iterative loop with KV cache and continuous batching (p.38).
- **Bi-encoder embedding vs cross-encoder reranker**: bi-encoder embeds query and doc independently (cheaper); cross-encoder reranker scores the (query, doc) pair jointly (heavier, more accurate) (p.38).
- **Semantic cache vs prefix/KV cache**: similarity — both skip compute on a cache hit. Difference — semantic cache matches by meaning (embedding similarity, catches paraphrases) with 100% compute saved on hit; prefix cache matches by exact shared prefix tokens, only skipping prefill for the matched portion (p.39).
- **Routing vs cascading**: similarity — both are cost-saving strategies for selecting model tier. Difference — routing decides model choice before generation (via a classifier); cascading runs a cheap model first and defers based on output confidence, paying for the cheap generation regardless of outcome (p.40).

---

## 4. Assumptions, Boundaries & Gaps

- **Vendor-reported numbers flagged as unverified**: Mercury 2 dLLM's "10× faster" claim and its tok/s figures are explicitly flagged in the source as self-reported by the vendor, not independently verified (p.36).
- **Marketing vs measured hit rates**: the source explicitly flags "95%" semantic-cache hit-rate claims as marketing exaggeration versus real measured rates of 30–68% (FAQ/support) and 10–25% (open-ended) (p.39).
- **DSpark engine-support gap**: DSpark is available in the `speculators` library but not yet exposed via a flag in SGLang — the deck explicitly warns not to promise this capability to a client without checking engine version (p.16).
- **Documentation drift warning**: the deck notes that some mirrored documentation from the v0.4.x era incorrectly claims `lpm` scheduling is the SGLang default — it is not; the source instructs checking `server_args.py` for the actual current-version default (p.26).
- **Version/flag volatility acknowledged as a boundary of the material itself**: flags and specific version numbers (vLLM v0.27.1, SGLang v0.5.17, etc.) are stated as drifting roughly every 2 weeks; the lecture explicitly frames itself as teaching technique over memorizing specific flags (p.19).
- **Diffusion serving compatibility gap (flagged, not resolved in slides)**: the deck states that PagedAttention/continuous-batching sizing formulas (§9) do not directly apply to dLLM serving, but does not provide an alternative capacity formula for dLLM — flagged as an open gap in the source material (p.36).
- **No single "safe" optimization for reasoning models**: the deck explicitly states no optimization (quantization, spec decoding, prefix caching, KV quantization) is universally safe for reasoning models — effects are model-size- and technique-dependent, and the slides don't give a decision formula, only the warning (p.34).
- **RL algorithm content explicitly out of scope**: PPO/GRPO/DPO algorithmic detail is explicitly deferred to Days 21–22; this lecture only covers the serving-side mechanisms (sleep mode, weight transfer) — flagged boundary, not a gap to fill from outside knowledge (p.35).
- **Cost modeling deferred**: $/1M token cost modeling is explicitly deferred to Day 25 (GPU FinOps); this lecture's capacity-planning section only answers "does it fit / how many users," not "what does it cost" (p.47, p.50).
- **Prerequisite assumption**: the lecture assumes familiarity with prior Day 16–19 infrastructure content (cloud infra, data pipelines, lakehouse, vector/feature store), referenced directly in the Chapter 4 recap and Milestone 1 integration requirement (p.53, p.52).
- **ZK-proof inference limitation stated outright**: zero-knowledge-proof inference is explicitly noted as still 10⁴–10⁵× slower than normal inference — presented as a hard current limitation, not a near-term solution (p.42).
- **Hardware spec target miss flagged**: Vera Rubin (VR200) met its 50 PFLOPS NVFP4 compute target on schedule (Aug'26) but NVIDIA lowered its HBM4 bandwidth spec from 22TB/s to ~20TB/s — flagged in-source as evidence that compute targets and bandwidth targets don't move in lockstep (p.46).
- **Benchmark methodology gap warning**: the deck flags that popular single-process asyncio benchmark clients silently distort TTFT/TPOT measurements due to client-side queueing (GIL-driven, per arXiv 2605.24217) — implying that many published/self-run benchmarks in the wild may be measuring the client, not the server; the deck doesn't quantify how widespread this issue is beyond noting the citation (p.48).
- **Edge case explicitly called out**: low-spec laptops (8GB RAM, no GPU) are stated to fully support Lab 20's core tracks — an explicit boundary condition confirming minimum-hardware viability (p.51).

---

## 5. Learning Priorities

### Essential
- Latency/throughput vocabulary: TTFT, TPOT, E2E latency, Throughput vs Goodput@SLO, Queue Depth (p.4).
- Why pre-LLM serving stacks failed for LLMs, and the PagedAttention + continuous batching shift (p.5).
- Quantization formats and tradeoffs (FP16/FP8/AWQ/GPTQ/NVFP4/MXFP4/GGUF), and selection guidance by hardware/use case (p.6–8).
- KV cache mechanics, PagedAttention, RadixAttention/prefix caching, and the MHA→GQA→MQA→MLA progression (p.9–10, p.22).
- Speculative decoding core concept, Acceptance Length, and why AR drafting plateaus vs parallel drafting (EAGLE-3 → DFlash/DSpark) at a conceptual level (p.12–16).
- Continuous/in-flight batching mechanism and benefit (p.12).
- The 8-engine serving landscape and production selection guidance (vLLM/SGLang for general production, llm-d/Dynamo for disaggregated scale, llama.cpp for edge) (p.19).
- Distributed parallelism strategy table (DP/TP/PP/EP/disaggregated P/D) and the placement rules (TP within node, PP across nodes) (p.31–32).
- Capacity planning formula (KV cache per token, VRAM budget) and the worked example (p.47).
- Benchmarking correctness requirements (ISL/OSL, closed vs open loop, percentiles not means, warm-up exclusion) and the client-side-bottleneck pitfall (p.48).
- SLA best practices and the framing that Goodput@SLO — not peak throughput — determines production success (p.4, p.49–50).

### Important
- Attention backend auto-selection logic (FA3/FA4/FlashInfer/FlashMLA by hardware) (p.23).
- FlashAttention lineage (FA1 IO-awareness insight through FA4) and compilation stack (torch.compile, CUDA graphs, TensorRT) (p.17–18).
- Disaggregated Prefill/Decode serving rationale and when it pays off (p.27).
- Multi-LoRA serving mechanism and use case (p.28).
- Expert Parallelism / MoE scaling basics (A2A, TBO, EPLB) (p.29).
- DP vs DPA distinction and cache-aware routing (sgl-router) (p.30).
- Hybrid/SSM attention and sparse (DSA) attention as responses to the KV-memory wall, and the three downstream mechanisms they break (prefix caching, spec decoding, disaggregation) (p.11).
- Reasoning-model serving's capacity-bound regime shift and the "no optimization is universally safe" warning (p.34).
- Agentic serving's cache-locality problem and the shared-KV-pool fix (p.33).
- RL rollout serving mechanisms: sleep mode, weight transfer, colocated vs disaggregated (p.35).
- Model routing vs cascading as the largest cost lever, and the 2026 preference for routing (p.40).
- Power/energy as a first-class 2026 constraint (tokens-per-joule, power wall) (p.41).
- Auto-scaling architecture and cold-start mitigation levers (p.43–44).
- Hardware landscape and the HBM-bandwidth-is-the-real-bottleneck framing (p.46).

### Supporting
- Diffusion LLM (dLLM) serving as an emerging paradigm and its serving-assumption break (p.36).
- Multimodal (VLM) serving specifics (image-token explosion, EPD disaggregation) (p.37).
- Embedding & reranker serving specifics (p.38).
- Semantic caching 3-layer stack and its risks (p.39).
- Confidential inference / TEE mechanics and inference-layer attacks (p.42).
- Edge deployment flow specifics (ONNX/TensorRT vs GGUF/Ollama paths) (p.45).
- Structured generation (grammar backends, tool/reasoning parsers) (p.24).
- Production tuning knob specifics (exact flag names/values) (p.25–26).
- Lab 20 and Milestone 1 logistics (p.51–52) — administrative, not conceptual.
