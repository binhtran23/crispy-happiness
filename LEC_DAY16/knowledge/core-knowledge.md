# Core Knowledge

**Source:** `1-Day 16_ Track 2_ Cloud infrastructure for AI.pdf` (36 pages, AICB-P2T2 · Ngày 16 · Track 2 · Chương 4: Hạ Tầng)
**Lecture scope (explicit, p.3, p.6-33):** 8 sections — (1) Cloud Provider comparison, (2) Cloud Foundation (IaaS/PaaS/AI-aaS), (3) GPU Instance Types & Cost, (4) Terraform IaC, (5) Docker→Kubernetes, (6) Networking & Storage Strategy, (7) Agent Infrastructure 8 Layers, (8) AI Serving Stack (vLLM/SGLang/etc).
**Stated learning objectives (p.4, explicit):** choose a cloud provider fit for a specific AI workload; design a cost-optimized GPU compute environment; deploy container orchestration for AI serving; deploy a production-ready AI endpoint on cloud.
**Stated end-of-day deliverable (p.5, explicit):** running cloud AI environment + cost estimate + live agent endpoint — cloud env with least-privilege IAM, GPU instance in private VPC via Terraform, working vLLM/SGLang inference endpoint, cost dashboard/estimate document.

---

## 1. Core Concepts

- **Cloud service models — IaaS / PaaS / AI-aaS / Physical (p.11)**
  - AI-aaS: OpenAI API / Bedrock / Vertex AI, pay-per-token — fastest, no infra needed.
  - PaaS: SageMaker / Azure ML / AI Platform — managed training + serving, less ops.
  - IaaS: EC2 GPU / GCE / Azure VM — full GPU control, self-managed, needs ops.
  - Physical: on-premise/colocation — max control, max effort.
  - Why it matters: determines the trade-off between control and operational burden for each part of an AI workload.

- **Hybrid cloud pattern for AI (p.11, explicit)** — real AI workloads typically mix models: Training on IaaS (GPU control) → Serving on PaaS (managed scaling) → Prototyping on AI-aaS (fastest). Related: multi-cloud strategy (below).

- **Shared Responsibility Model (p.12)** — cloud provider is responsible for physical infrastructure, hypervisor, network backbone/DDoS protection, hardware maintenance/availability. The team is responsible for model security, data encryption, IAM access control (least-privilege), prompt injection prevention, data privacy compliance (NĐ13, GDPR). Why it matters: clarifies where the team's security obligations start in an AI-specific context (prompt injection is called out as a team responsibility, not the provider's).

- **Landing Zone (p.12, explicit)** — the initial account/network/guardrail setup for a cloud environment. Components: account structure (workload accounts, shared services account, security account), networking (Transit Gateway hub-spoke topology, private subnets for GPU), centralized logging (CloudTrail + CloudWatch aggregation), guardrails (SCPs — Service Control Policies — enforcing security baseline). Explicit claim: getting this wrong costs "10x" to rework later.

- **GPU hardware tiers (p.14, explicit table)** — T4 (16GB, $0.35/hr, 320GB/s, small inference ≤7B), L40S (48GB, $0.40–0.86/hr, 864GB/s, "sleeper pick" for medium inference), A10G (24GB, $1.00/hr, 600GB/s, production inference), A100 (80GB, $1.79–2.70/hr, 2TB/s, fine-tuning), H100 (80GB HBM3, $2.99–4.31/hr, 3.35TB/s, pre-training), H200 (141GB HBM3e, $3.72–5.58/hr, 4.8TB/s, single-GPU 70B LLM), B200 Blackwell (192GB HBM3e, $6.84–8.64/hr, 8TB/s, ultra-scale, limited availability, 2026). Why it matters: VRAM + bandwidth + price jointly determine which task (inference/fine-tune/pre-train) a GPU is fit for.

- **MIG — Multi-Instance GPU (p.17, explicit)** — splits one A100/H100 into up to 7 isolated instances (e.g. 1×A100 80GB → 7×10GB instances). Used to maximize utilization and serve many small models in parallel on shared hardware. Explicitly contrasted with GPU overcommit (see Assumptions/Gaps).

- **Terraform / Infrastructure-as-Code (p.19)** — defines cloud resources (e.g. `aws_instance` GPU resource block) as code. Core components mentioned: modules (for GPU instances, VPC, security groups — reusable), state management (S3 backend + DynamoDB lock, for team environments), workspaces (dev/staging/prod isolation). Why it matters: reproducibility, avoids "works on my machine" (p.33, explicit).

- **Alternatives to Terraform (p.19, explicit)** — Pulumi (Python/TypeScript native, "popular with AI teams"), OpenTofu (open-source fork, post-HashiCorp BSL 2023 relicensing), AWS CDK (for pure-AWS + TypeScript teams).

- **NVIDIA GPU Operator (p.20, p.22)** — automates installing GPU drivers, CUDA toolkit, and device plugin inside a Kubernetes cluster.

- **Karpenter (AWS) / NAP (GCP) (p.16, p.22, explicit)** — smart/auto node provisioners: Karpenter auto-provisions the correct GPU instance type per pod's resource request; NAP is GCP's smarter Cluster Autoscaler. Both support scale-to-zero outside peak hours (stated saving: 60%+ on idle GPU cost).

- **K8s GPU resource requests (p.22, explicit)** — `nvidia.com/gpu: 1`; rule stated: requests must always equal limits for GPU (no fractional overcommit).

- **K8s namespace segmentation for ML teams (p.22, explicit)** — `ml-training/` (separate resource quotas, cost tracking), `ml-serving/` (separate RBAC, production isolation), `ml-experiments/` (sandbox, no strict limits).

- **Service Mesh (Istio/Linkerd) (p.24)** — provides mTLS, canary routing, and tracing between services (inference, orchestrator, vector DB nodes shown in the architecture diagram).

- **API Gateway (p.24)** — sits between client and ALB/Ingress; handles rate limiting, request queueing, SSE streaming.

- **GPU-to-GPU networking (p.24, explicit)** — NVLink (900 GB/s, intra-node) vs InfiniBand (400 Gbps, inter-node, used for multi-node training); EFA (AWS) given as an InfiniBand alternative.

- **VPC Endpoints / PrivateLink (p.24, explicit)** — keeps traffic off the public internet; stated benefits: security + reduced egress cost.

- **Storage tiering for AI systems (p.25, explicit)** — Hot: Redis/GPU memory, sub-ms latency, for active KV cache/embedding cache/session state. Warm: S3 Standard/EBS, $0.023/GB/mo, for model weights/recent checkpoints/training data. Cool: S3 Infrequent Access, $0.0125/GB/mo, for old checkpoints/infrequent datasets. Archive: S3 Glacier Deep Archive, $0.00099/GB, for compliance data/model archaeology. Best practices stated: S3 versioning for model artifacts, lifecycle policies to auto-archive after 90 days, S3 Intelligent-Tiering for mixed access patterns.

- **8 Layers of a production AI agent (p.27, explicit — numbered 1 lowest to 8 highest)**
  1. Compute — GPU (LLM inference agents) + CPU (orchestrator) + Serverless (tool-calling).
  2. Orchestration — LangGraph / CrewAI / AutoGen running on CPU pods; manages lifecycle & retry.
  3. Message Queue — Redis Streams (low latency) | Kafka (high throughput, replay) | RabbitMQ.
  4. Cache — L1 in-process LRU dict | L2 shared Redis (TTL) | L3 embedding cache.
  5. Storage — PostgreSQL (conversation history), pgvector (long-term memory), Redis (short-term, TTL), S3 (tool outputs).
  6. Networking — API Gateway, internal gRPC (high perf), HTTP+SSE (stated as the MCP transport).
  7. Observability — OpenTelemetry → Jaeger traces, LangSmith, Prometheus KPIs.
  8. Secrets & Config — Vault / AWS Secrets Manager, feature flags for A/B testing.
  - Design principle stated (p.27, explicit): agents should be stateless (externalize state to Redis/Postgres) to enable horizontal scaling.

- **AI serving engines (p.30, explicit comparison table)** — vLLM (PagedAttention; broadest ecosystem, OpenAI-compatible API; best for broad compatibility/easy deploy), SGLang (RadixAttention + Prefill-Decode disaggregation; multi-turn +20%, JSON 3× faster, powers 400K+ GPUs globally; best for agents/multi-turn/structured output), LMDeploy (TurboMind C++ engine, zero Python overhead; 1.8× throughput vs vLLM, Int4 2.4× faster; best for quantized/latency-sensitive apps), TensorRT-LLM (NVIDIA-optimized kernels; 30–50% faster at high concurrency; best for ultra-scale production), TGI (HuggingFace native; quick deploy, built-in Prometheus; best for prototyping/HF ecosystem), Ollama (llama.cpp backend; CLI + easy model switching; best for edge/laptop/dev). Stated 2026 update: SGLang & LMDeploy have surpassed vLLM by ~29% raw throughput for many use cases.

- **PagedAttention (vLLM) (p.30, p.32)** — named as vLLM's core technique; not explained mechanistically in the source (name/attribution only — flagged as a gap below).

- **RadixAttention (SGLang) (p.17 is not this; see p.31, explicit)** — reuses KV cache across requests that share a common prefix; drives the multi-turn 10–20% latency/throughput gain and lower TTFT vs vLLM in multi-turn scenarios. Also: Compressed FSM (JSON output 3× faster than naive) and Prefill-Decode Disaggregation (separates GPU roles for prefill vs decode). v0.4 adds a "zero-overhead" batch scheduler (<2% CPU).

- **TurboMind (LMDeploy) (p.31, explicit)** — engine written entirely in C++ (zero Python overhead), with persistent batch inference + blocked KV cache; delivers 1.8× request throughput vs vLLM baseline and 2.4× faster Int4 inference vs FP16.

---

## 2. Relationships & Mechanisms

- **Cloud provider choice ← workload type × budget × compliance × latency (p.7, p.8, explicit decision framework)**
  - Decision tree (p.8): need broadest ecosystem? → AWS. Else heavy PyTorch+TPU interest? → GCP. Else OpenAI-exclusive need? → Azure. Else VN data residency requirement? → Viettel/VNG/FPT.
  - AWS: broadest ecosystem, Bedrock/SageMaker/HyperPod, P5 (H100 8x)/P5e (H200) — best for broadest ecosystem + enterprise compliance.
  - GCP: PyTorch/JAX, GKE GPU auto-provisioning, Vertex AI, A3 Mega (H100 8x)/TPU v5p — best for heavy PyTorch training + TPU interest.
  - Azure: OpenAI Service exclusive, Prompt Flow LLMOps, ND H100 v5/ND H200 v5 — best for Microsoft stack + OpenAI API.
  - VN Cloud (Viettel/VNG/FPT): T4/V100, 60–70% of global price, NĐ13 data residency — best for ND13 compliance + data residency.
  - Specialized (Lambda/RunPod/GMI/CoreWeave): H100/H200, 40–70% cheaper, pure GPU — best for cost-sensitive teams with in-house infra skills.
  - Explicit rule (p.9): specialized cloud for training (cost optimization), hyperscaler for serving (managed scale + SLA).

- **Multi-cloud strategy (p.8, explicit)** — many organizations train on provider A (cheapest GPU), serve on provider B (best latency), store data on provider C (compliance) → requires an abstraction layer (Terraform/Pulumi) for portability. Depends-on: IaC tooling.

- **GPU selection process (p.15, Input → Process → Output, explicit decision tree)**
  - Input: task type (Inference / Fine-tune / Pre-train) and, for inference, model size.
  - Process: Inference branches by size — ≤13B → L40S/T4 ($0.35–0.86/hr); 13B–70B → A100/H100 ($1.79–4.31/hr); 70B+ → H200 ($3.72–5.58/hr). Fine-tune → A100/H100 ($1.79–4.31/hr). Pre-train → H100/H200 cluster, 8× nodes.
  - Output: a GPU/family choice matched to cost and workload. Rule of thumb (p.14, explicit): inference→T4/L40S/A10G, fine-tuning→A100, pre-training→H100 cluster.
  - Why each step matters: mismatched GPU choice (over- or under-provisioning VRAM/bandwidth for the task) drives the cost-inefficiency case study cited at the top of the deck ($50K/mo → $12K/mo via right-sizing + spot).

- **Cost strategy depends on commitment horizon (p.17, explicit decision framework)** — <6 months → Spot/On-demand (flexible, no commitment); 6–12 months → 1-year Reserved (30–40% savings); >12 months → 3-year Reserved (50–60% savings). Also: Training (interruptible) → Spot; Serving (needs stability) → Reserved; ROI turns positive after ~6 months. Worked example (p.15, explicit): GPT-2 1B-token fine-tune costs $45 on-demand vs $14 spot vs $27 reserved on A100.

- **Landing Zone → prevents rework cost (p.12)**: correct account/network/guardrail setup upfront is presented as a prerequisite that avoids a stated 10x rework cost later — a dependency/prerequisite relationship, not just a best practice.

- **Docker image optimization pipeline (p.22, Input → Process → Output, explicit)**
  - Input: base image `nvcr.io/nvidia/cuda:12.1-runtime`.
  - Process: multi-stage build; cache the pip layer separately (before `COPY source`); `.dockerignore` excludes datasets/checkpoints.
  - Output: image size reduced from 18GB to 6–8GB, and reduced cold-start time.

- **Kubernetes AI-serving architecture (p.21, explicit diagram)** — Ingress/ALB → K8s Cluster containing vLLM Pods (GPU: 1×A10G each) and an SGLang Pod (GPU: 1×H100); HPA scales on GPU metrics; GPU Operator (NVIDIA) manages driver/toolkit install; Karpenter auto-provisions nodes; an Init container pre-downloads model weights from S3 before pod start. Relationship: Init container (pre-download) is a prerequisite step before the serving pod can become ready.

- **Networking request path for AI workloads (p.24, Input → Process → Output, explicit diagram)** — Client → API Gateway (rate limit / queue / SSE streaming) → ALB/Ingress → Service Mesh (Istio/Linkerd, mTLS) routing to Inference, Orchestrator, and Vector DB services. Each mesh hop is mTLS-secured.

- **8-Layer agent infra as a dependency stack (p.27)** — layers are numbered 1 (Compute, lowest) to 8 (Secrets & Config, highest), implying each upper layer depends on the lower ones being in place (e.g. Orchestration (2) runs on Compute (1); Observability (7) instruments the stack running on layers 1–6).

- **Cache hit rate → reduced LLM API calls (p.28, explicit)** — target 60–80% cache hit rate (across L1/L2/L3) is stated to directly reduce LLM API call volume/cost.

- **Serving engine trade-offs (p.30–31)**: RadixAttention (SGLang) trades some architectural complexity (prefill-decode disaggregation) for KV-cache reuse gains specifically in multi-turn/shared-prefix scenarios; TurboMind (LMDeploy) trades Python flexibility (fully C++) for raw throughput and quantized-inference speed. vLLM is positioned as the general/compatibility-first baseline against which the 2026 gains of SGLang/LMDeploy (~29% raw throughput) are measured.

- **GPU memory utilization ceiling → OOM risk (p.31–32, explicit)** — 80% GPU memory utilization is the stated "safe zone"; 95% risks CUDA OOM during graph compilation. This is a direct constraint on the `--gpu-memory-utilization` deploy parameter shown in the vLLM launch command (p.32).

---

## 3. Examples & Distinctions

- **Case study (p.2, explicit, used as the framing hook for the whole lecture):** a startup burned $50K/month on GPU due to no optimization; right-sizing + spot instances brought it down to $12K/month.

- **A100 vs H100 (p.14):** A100 (80GB, 2TB/s, $1.79–2.70/hr) is positioned for fine-tuning; H100 (80GB HBM3, 3.35TB/s, $2.99–4.31/hr) for pre-training. Similarity: same VRAM (80GB). Difference: H100 has newer HBM3 memory, ~1.67× the bandwidth, and higher price — matched to the more bandwidth-hungry pre-training workload.

- **Reserved vs Spot/On-demand (p.17):** Similarity — both are GPU capacity purchasing models. Difference — Spot/on-demand is flexible/no-commitment but interruptible (fits training jobs that can tolerate interruption); Reserved requires a 1–3 year commitment but is cheaper long-run (30–60% savings) and fits stability-needed serving workloads. Distinguishing criterion: commitment horizon (<6mo vs 6–12mo vs >12mo) and interruption tolerance.

- **MIG vs GPU overcommit (p.22, explicit warning):** slide explicitly states "NEVER overcommit GPU: fractional sharing phức tạp (complex), dùng MIG thay vì overcommit" — i.e., MIG (hardware-level isolated partitioning) is the recommended alternative to software-level GPU overcommit, which is called out as unreliable/complex.

- **vLLM vs SGLang vs LMDeploy (p.30–31):** vLLM = PagedAttention, broadest compatibility/OpenAI-compatible API, general-purpose baseline. SGLang = RadixAttention, optimized for multi-turn/agentic/structured-output workloads via prefix KV-cache reuse. LMDeploy = TurboMind (C++), optimized for raw throughput and quantized (Int4) inference. Distinguishing criterion: what you're optimizing for — compatibility (vLLM) vs multi-turn/agent efficiency (SGLang) vs raw throughput/quantization (LMDeploy).

- **Redis Streams vs Kafka vs RabbitMQ (p.28, explicit):** Redis Streams = low latency (<1ms), simple setup, "best for most cases." Kafka = high throughput, durability, replay — for large-scale agent systems. RabbitMQ = complex routing rules, dead-letter queues. Distinguishing criterion: latency need vs throughput/durability need vs routing complexity need.

- **Storage tiers (p.25):** Hot/Warm/Cool/Archive distinguished by latency requirement and access frequency, with an explicit price gradient (sub-ms/unspecified for Hot → $0.023 → $0.0125 → $0.00099 per GB).

- **Karpenter (AWS) vs NAP (GCP) (p.16, p.22):** both are smart GPU node auto-provisioners; difference is platform — Karpenter for AWS, NAP for GCP (described as GCP's "smarter Cluster Autoscaler").

---

## 4. Assumptions, Boundaries & Gaps

- **PagedAttention mechanism not explained (gap).** The slides name vLLM's core technique (PagedAttention) and attribute benefits to it (broad compatibility, OpenAI-compatible API) but never describe how it works mechanistically — unlike RadixAttention and TurboMind, which get explicit mechanism bullets. Flagged, not filled from external knowledge.

- **RadixAttention "Prefill-Decode Disaggregation" mentioned but not detailed (gap, p.31).** Listed as a bullet ("tách GPU roles" / separates GPU roles) without explaining how roles are split across GPUs or what triggers the split.

- **B200 Blackwell figures marked as forward-looking/uncertain by the source itself (p.17, explicit).** "11–15× inference vs H100 (promise)" and "stabilize ~20-30% trên H200" are explicitly hedged in the source as a promise/projection, and pricing is marked "ramp-up 2026" — i.e., the source itself flags these numbers as not yet settled, not this extractor's inference.

- **GPU pricing figures are dated "2026" throughout and are point-in-time market prices (assumption, not flagged as such in source).** These are explicit numbers in the deck but inherently volatile; the source does not caveat that they may already be stale — worth treating as illustrative rather than durable facts.

- **"MCP transport" mentioned without explanation (p.27, gap).** HTTP+SSE is listed as "MCP transport" under the Networking layer, but MCP itself (presumably Model Context Protocol) is never defined or explained in this deck.

- **Compressed FSM (SGLang) named but not mechanistically explained (gap, p.31).** Stated to make JSON output 3× faster than naive, with no description of what a "compressed FSM" is or how it accelerates structured output.

- **Prerequisite assumption (implicit/inferred):** the lecture assumes learners already understand basic cloud concepts (VPC, IAM, EC2-type instances), containerization (Docker basics), and Kubernetes basics, since it jumps directly into AI-specific specialization (GPU Operator, Karpenter, MIG) without defining foundational terms like "VPC," "IAM," "subnet," or "pod" from scratch. This is inferred from the deck's density and pacing, not explicitly stated as a prerequisite.

- **Cost figures for Vietnam cloud providers are approximate/limited (p.9, explicit).** VNG Cloud is stated as "đang build GPU capacity" (still building GPU capacity — not yet fully available). Viettel/VNG/FPT are explicitly noted as lacking H100/H200 and having limited availability zones — a real capability boundary on the VN cloud option.

- **Specialized GPU cloud trade-off boundary (p.9, explicit):** fewer managed services, more self-management burden, fewer availability zones — explicitly framed as the cost of the 40–70% price advantage, i.e., not a strictly-better option.

- **Health check readiness probe value (60s) stated without justification (p.31–32).** The 60-second initial delay for readiness probes is given as a specific number but the reasoning (e.g., model load time) is only implicit from context (models must load into GPU memory before being ready), not spelled out.

---

## 5. Learning Priorities

**Essential**
- Cloud service model spectrum: IaaS / PaaS / AI-aaS / Physical, and the hybrid pattern (Training→IaaS, Serving→PaaS, Prototype→AI-aaS).
- Cloud provider decision framework (workload × budget × compliance × latency) and the AWS/GCP/Azure/VN-cloud/specialized-cloud positioning.
- GPU hardware tiers (T4→B200) mapped to task type (inference/fine-tune/pre-train) via VRAM, bandwidth, price.
- GPU selection decision tree (task type → model size → GPU choice).
- Cost strategy: Spot vs Reserved vs On-demand, tied to commitment horizon and interruptibility.
- Terraform/IaC role (modules, state management, workspaces) and why IaC matters (reproducibility).
- Kubernetes AI-serving architecture: GPU Operator, Karpenter/NAP, Init containers for weight pre-download, HPA on GPU metrics.
- The 8 Layers of production AI agent infrastructure and the "stateless agent" design principle.
- AI serving engines comparison (vLLM/SGLang/LMDeploy/TensorRT-LLM/TGI/Ollama) and their core techniques (PagedAttention, RadixAttention, TurboMind).
- GPU memory utilization safe zone (80%) and continuous batching as deploy fundamentals.

**Important**
- Shared Responsibility Model in an AI-specific context (team owns prompt injection prevention, model security).
- Landing Zone components (account structure, networking, guardrails) and the "10x rework cost" stakes.
- MIG (Multi-Instance GPU) vs GPU overcommit.
- Storage tiering (Hot/Warm/Cool/Archive) with cost/latency trade-offs.
- Networking stack for AI workloads (API Gateway, Service Mesh/mTLS, NVLink vs InfiniBand, VPC Endpoints).
- Message queue selection (Redis Streams vs Kafka vs RabbitMQ) and multi-level caching (L1/L2/L3) with the 60–80% hit-rate target.
- Docker image optimization workflow (multi-stage build, layer caching, .dockerignore).

**Supporting**
- Specific dollar figures for GPU pricing per provider/instance (illustrative, likely to date quickly).
- Vietnam cloud provider names and their specific GPU inventory limitations.
- Specific serving-engine throughput multipliers (1.8×, 2.4×, ~29%, etc.) beyond the general ranking.
- Exact CLI/launch commands for vLLM/SGLang (illustrative of deploy tips, not conceptually load-bearing).
- Next-lecture preview (Day 17: Data Pipeline Engineering, Airflow/Kafka/ETL) — out of scope for this lecture's content.
