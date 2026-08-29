# Core Knowledge

**Deck:** "GPU FinOps & Cost Optimization" — AICB-P2T2 · Ngày 25 · Chương 5: Vận Hành (VinUniversity, Phase 2 Track 2, Tuần 5). Course language: Vietnamese with English technical terms. 45 content slides (numbered 1/45–45/45) + title, Q&A, and thank-you slides ≈ 50 pages total in the file.

## 1. Core Concepts

- **GPU Cloud Cost Anatomy** — Cost breakdown 2026: Compute (GPU-hours) ~50%, Power ~15%, Storage ~12%, Network ~8%, Other ~15%. Power is the fastest-rising line item (H100 node ~1,400 W/GPU all-in; power ~10–20% of TCO but ~30–40% of opex). Hidden costs: egress $0.09/GB (AWS), NAT gateway $0.045/GB, Secrets Manager $0.40/secret/month, inter-AZ traffic. Correct cost unit is **$/1M-token** (inference) or **$/job** (training), not $/GPU-hr. (p.4, explicit)

- **$/GPU-hr as "Vanity Metric"** — Same chip can vary 4–5× in $/GPU-hr across providers (provider arbitrage), but the real budgeting unit should be $/1M-token. A high $/GPU-hr chip (e.g., GB300) can still be cheapest per token due to throughput. Rule: normalize to $/1M-token (inference) or $/job (training); $/hr high ≠ expensive per token. (p.7, explicit)

- **MFU (Model FLOPs Utilization)** — = achieved FLOPs / peak FLOPs (from PaLM paper). Good: 35–45%; excellent: 50%+. Llama-3 405B ~38–43%. Default 70B config <30%; sequence-parallel (TP 8→1) + selective checkpointing ≈2× MFU. Used for **training** (compute-bound workloads). (p.18, explicit)

- **MBU (Model Bandwidth Utilization)** — = achieved bandwidth / peak bandwidth; target ~60% (H100-80GB, batch-1). Used for **decode/inference** (memory-bound workloads). Ridge point H100 ~295 FLOP/byte (BF16); decode batch-1 only 1–2 FLOP/byte → memory-bound. Decode scaling follows HBM bandwidth: H100 3.35 → H200 4.8 → B200 8 TB/s. (p.18, explicit)

- **Roofline model** — Framework distinguishing compute-bound vs memory-bound regimes using the ridge point (FLOP/byte). Training is compute-bound (measure with MFU); decode/inference batch-1 is memory-bound (measure with MBU). (p.18, explicit)

- **GPU-Util% (nvidia-smi) vs real efficiency** — nvidia-smi "GPU-Util" = % of time ≥1 kernel is running; it does NOT measure compute throughput. A single thread on one SM can report 100% utilization. Training can show 100% GPU-Util while true MFU is only ~20–30% → billed for full GPU-hour while using <½ of real FLOPS. FinOps rule: never capacity-plan or buy more GPU based on GPU-Util alone — it shows "busy," not "efficient." (p.17, explicit)

- **Goodput** — = requests/sec that actually meet SLO (TTFT + TPOT targets), not raw throughput. E.g., 10 req/s but only 3 meet SLO → goodput = 3 (7 requests served but wasted spend). Optimizing goodput (via disaggregation) can raise req/GPU ~2×. Framed as the "final" FinOps cost-per-token measure. (p.19, explicit)

- **Purchasing/commitment tiers (2026 ladder)** — Spot/Preemptible (−40–70% discount, up to −80% on GCP; no commitment; for interruption-tolerant jobs + checkpointing), On-Demand/Serverless (0% discount, no commitment; spiky/<5–6h/day workloads), Capacity Block/AWS (medium discount; 1–182 days commitment, book ≤8 weeks ahead; for guaranteed bursts at fixed price), Reserved/CUD (~45–55% discount off GPU; 1–3 year commitment; for stable, high-utilization workloads). (p.12, explicit)

- **Break-even utilization** — ≈ 1 − discount%. E.g., 45% discount → need ~55% utilization (~13.2h/day) for a reserved commitment to "pay off" vs on-demand. Reserved instances run at only ~30% utilization still lose money relative to pay-as-you-go. Central decision rule for purchasing strategy. (p.12, p.4 Objectives, explicit)

- **Fractional GPU sharing** — Techniques to split one physical GPU across multiple tenants to raise cost-density: **MIG** (A100/H100 hardware partitioning, dedicated memory, high isolation), **Time-slicing/MPS** (higher density, lower isolation). Trade-off: cost density ↔ isolation. Real-world compute (MFU) utilization by enterprises is often only ~5% despite 60–70% time-active GPU-Util. Example outcomes: Run.ai reports 0.5 GPU = 77% throughput / 86% user capacity via co-location (~3× more users); KAI Scheduler (open-source, Apache 2.0, released 4/2025); K8s DRA GA 1.34 (8/2025) natively supports "request GPU ≥X GB." (p.22, explicit)

- **Kubernetes cost-autoscaling stack (2026)** — Karpenter (bin-packing/consolidation, drains empty nodes, ~40–60% reduction vs Cluster Autoscaler), KEDA (scale-to-zero: 0 replicas when idle after cooldown), Kueue (gang scheduling + team quotas + cohort borrowing, prevents GPU hoarding), DRA + KAI Scheduler (native fractioning + fair-share/over-quota). Attacks two waste layers together: idle within-card (fractioning) and idle between-nodes (autoscaling/spot). (p.23, explicit)

- **LLMflation** — Inference price for equivalent capability has fallen ~10×/year on average (Epoch AI), but NOT evenly — range is 9×–900×/year; median ~50×. GPT-3-level cost: $60/1M (2021) → $0.06/1M (Llama 3.2 3B, 2024) = 1,000× in 3 years. "GPT-4-level" capability dropped from $30–60/1M (3/2023) to <$1/1M. Implication: forecast price drops per capability tier, not as a flat curve. (p.24, explicit)

- **Reasoning/test-time compute cost** — "Reasoning tax": +5–10× tokens billed per query due to hidden "thinking tokens" in output pricing (e.g., ~10k reasoning tokens for a 500-token answer = ×21 effective cost). 21.8% of model catalog "re-prices" (up to 28×) — cheap on paper, expensive in practice. Benchmark should be $/task actually completed, not $/token nominal. (p.26, explicit)

- **Discount stack (inference cost levers)** — Batch API (−50%, requires 24h SLA tolerance / latency-tolerant workload), Prompt caching-read (−90% at 0.1× cost per Anthropic; requires ≥2 reads, 1.25–2× write cost), OpenAI/Gemini caching (−50 to −90%; Gemini has a storage fee). Stacked (Batch × Caching) ≈ 95% off (confirmed by Anthropic). Caveat: caching benefit is confirmed only on the read side; Gemini's storage fee can flip the math — cache deliberately, not blindly. (p.27, explicit)

- **Small-model cost levers** — Request batching (~8× throughput via vLLM continuous batching), semantic caching (30–40% hit rate typical for chatbots), model cascading (small model handles 80%, escalates 20% to larger model; FrugalGPT up to 98% cost cut, RouteLLM 85%+), quantization (FP8/FP4 for Blackwell is the frontier; AWQ 4-bit is the INT4 tier — example: FP16 A10G 1200 tok/s $0.83/M vs AWQ 4-bit 1800 tok/s $0.55/M = 34% savings). Stacked levers (batching+caching+cascading+quantization) → 70–85% total savings vs naive deployment. (p.28, explicit)

- **Disaggregated serving (prefill/decode split)** — Prefill (compute-bound) and decode (memory-bound) have different resource profiles; running both on the same GPU wastes capacity for both. Splitting into separate right-sized pools yields 7–30× QPS on the same GPU fleet. Named systems: DistServe (OSDI'24, 7.4× request throughput / 12.6× under strict SLO, >90% SLO attainment), Mooncake (Kimi, FAST'25, +75% real throughput, reuses idle CPU/DRAM/SSD as a KV tier), NVIDIA Dynamo (up to 30× for DeepSeek-R1 671B on GB200 NVL72; >2× for Llama-70B on Hopper; Qwen3-235B-FP8 1.86×). Caveat: KV-transfer overhead grows with context length, so disaggregation wins most at high QPS + shared long context; small deployments may see no benefit or a loss. (p.29, explicit)

- **KV-cache economics** — Cache-hit% is framed as a cost KPI, similar to token count. Anthropic read-cache: 0.1× cost (~90% off); OpenAI 50→90% off. DeepSeek cache hit rate ~1/10 (V4 Flash ~98%). Miss vs hit cost differs 50–100×. KV offload/tiering for vLLM: −69% prefill cost at 80% hit rate (4×H100, 128K prompt context), up to 15× throughput; reached GA 1/2026 (adopted by GKE Inference, CoreWeave, Cohere). Practice: structure prompts with a static prefix (system prompt, RAG context, few-shot) to stabilize caching; monitor cache-hit% as a primary cost KPI. (p.30, explicit)

- **Per-lever cost validity windows** — Every optimization technique has a narrow regime where it actually helps, and can backfire outside it: Speculative decoding (−48% $/1M-token when latency-bound, but +19% cost when throughput is already saturated; ROI depends on acceptance rate, e.g. EAGLE-3 ~2.5× on vLLM). Chunked prefill (protects inter-token latency, but chunk=512 adds +25% overhead; chunk≈2048 approaches zero overhead). NVIDIA NIM (enterprise license ~$4,500/GPU/year ≈ $1/GPU-hr add-on; only pays off above a break-even utilization threshold). Rule: every lever has a load regime it fits — measure unit economics before/after, don't assume "faster = cheaper." (p.31, explicit)

- **FinOps 2025/Cloud+ framework** — FinOps scope expanded from Public Cloud/SaaS/Data Center to also include **AI, Licensing, Private Cloud** ("Cloud+"). Lifecycle stays Inform → Optimize → Operate (repeated across slides as the maturity model most AI teams start at "Inform"). Stats: 98% of teams now track AI cost (vs 31% two years prior); "AI value management" ranked #1 needed skill; cloud waste rose to 29% (Flexera 2026) — first increase in 5 years, partly because AI makes forecasting hard. (p.2, p.32, explicit)

- **AI Unit Economics KPIs** — Cost/Token = Total $ / Tokens (e.g., $2,500/1M = $0.0025); Cost/Inference = Total $ / #inferences; Utilization = Actual/Provisioned (e.g., 800/1000 = 80%); ROI = (Benefit−Cost)/Cost; Token yield rate = % of tokens that produce business value. Point: "a cheap token" ≠ "cheap total spend" — enterprise GenAI spend grew $1.7B (2023) → $37B (2025); optimize token *yield*, don't just minimize token cost; structured output can cut 30–60% of tokens via model routing/cascading. (p.33, explicit)

- **FOCUS (FinOps Open Cost & Usage Spec)** — Open, vendor-neutral schema to normalize multi-cloud billing data. Versions: 1.2 (5/2025) added virtual-currency/token column; 1.3 (12/2025) added Contract Commitment + Split Cost Allocation; 1.4 (6/2026) reached 47 columns. Native export support from >12 providers (AWS, Azure, GCP, Oracle, Alibaba, Nebius, Databricks, etc.). New professional credential: "FinOps Certified for AI" (launched 6/2025, exams from 3/2026; maturity levels Crawl/Walk/Run). Gap: full AI token consumption (model identity, input/output tokens) is only scoped for FOCUS 1.5, which is NOT yet ratified as of 6/2026. (p.34, explicit)

- **Cost observability tool stack** — Two parallel tracks: (a) Infra/GPU: DCGM → Prometheus → Kubecost 3.0/OpenCost, cost broken down per pod/namespace by actual usage, separates Workload Idle vs Infrastructure Idle. (b) LLM/API: LiteLLM proxy (100+ providers) + Langfuse/Helicone, tracked as $/request and $/1M-token per API-key/team, with hard budget caps blocking over-limit requests. FOCUS is meant to unify both into one schema once AI-token fields (1.5) are ratified. (p.35, explicit)

- **Tagging, quota, per-service cost breakdown** — Tag convention (team=, project=, env=, cost-center=) enforced via SCP/OPA policies; K8s labels flow into DCGM → Prometheus. K8s ResourceQuota per namespace (example: team budget max 4 GPUs, 100GB storage). Kubecost 3.0 (GA 9/2025): ClickHouse-based, dropped DGCM dependency requirement, GPU-aware, free tier <$1M. Gotcha: on AWS EKS, splitting GPU/accelerator cost is only available via CUR 2.0 SCAD / Data Exports, NOT via the Cost Explorer UI (EKS-only limitation as of 9/2025). (p.36, explicit)

- **Showback → Chargeback maturity path** — 4-stage pipeline: (1) Visibility (tag + DCGM cost) → (2) Showback (report cost to teams, 4–8 weeks) → (3) Chargeback (actually bill teams, requires tag-coverage >80%) → (4) $/Outcome (tie cost to win-rate ~70%). Allocation models: per-namespace/tenant/token/experiment. Insight: average AI-GPU utilization is only ~60–70%, with the worst-case teams at ~5% — this is framed as the core "waste" problem chargeback is meant to expose. (p.37, explicit)

- **AI cost tool landscape (token bill is only 1/9 of spend)** — Tools: IBM Apptio + Kubecost → Cloudability/Turbonomic/Kubecost; FOCUS-based tools: Vantage, CloudZero, Finout, nOps; LLM-token tools: LiteLLM, Langfuse (Helicone → maintenance mode as of 2026). Key claim (FinOps X 2026): the LLM token bill is only 1 of 9 total AI cost categories — the other 8 (retrieval, orchestration, KV-cache infra, evaluation, governance, human labor, waste/errors, integration) are typically NOT measured. Maturity ladder: visibility → cost/token → cost per verified outcome. (p.38, explicit)

- **Power/energy as the real bottleneck** — Grid capacity pricing (PJM) rose ~11× in 2 years ($28.92 → $329.17/MW-day); data centers absorb 63% of that increase. A typical AI site needs 100–750 MW, and new grid interconnects take 24–36 months (4–7 years in some hubs) — capacity, not chip supply, is the binding constraint. New cost unit: $/MW (was $10M/MW unpowered/shell → now $20–30M/MW AI-optimized → $30–44M/MW all-in). Power is ~10–20% of TCO but ~30–40% of opex. Strategy: site latency-tolerant/batch/training workloads where power is cheap and clean (e.g., ~5–6¢/kWh in E. Washington vs ~15¢/kWh in California; carbon ~6 vs ~660 gCO2/kWh Norway vs Poland, >100× difference). (p.39, explicit)

- **Tokens-per-Watt / sustainability unit economics** — Example figures: 0.24 Wh, 0.03 gCO2e, 0.26 mL water per query (implied scope not fully specified on slide); 33× energy and 44× carbon reduction claimed over 12 months; only 58% of energy comes from accelerators (rest is non-accelerator overhead, ~42%). Reasoning models cost 74–86× more energy per query (o3 ~39.2 Wh, DeepSeek-R1 ~33.6 Wh vs GPT-4.1 nano ~0.45 Wh). Lever: model routing + reasoning-token budgets can cut energy (and $) by 1–2 orders of magnitude; track Wh/query like $/1M-token. (p.40, explicit — note: exact scope/methodology behind the 0.24Wh baseline figure is not explained on the slide, flagged as a gap below)

- **Carbon & water as governed cost levers** — Google fleet PUE 1.09 vs industry ~1.56 (→ ~56% extra electricity for overhead cooling/power on every watt of IT load at typical facilities). SCI (Software Carbon Intensity, ISO/IEC 21031:2024) standard proposed to pair carbon accounting with $/1M-token dashboards/ESG reporting. Carbon-aware scheduling: deferrable jobs can cut carbon 20–50% by running at off-peak (often cheaper) times. Hyperscalers are hedging power via nuclear PPAs (Microsoft: Three Mile Island 835MW, ~2028 restart; Google: 500MW SMR; Amazon: 320MW + >5GW pipeline). Framing: sustainable AI = cost savings + carbon reduction simultaneously, not a trade-off. (p.41, explicit)

## 2. Relationships & Mechanisms

- **Purchasing decision flow:** Chip/workload choice → $/1M-token benchmark (not $/GPU-hr) → break-even utilization calc (≈1−discount%) → choose tier (spot/on-demand/capacity block/reserved) → lock commitment only if utilization forecast clears break-even. (p.4, p.12, explicit)

- **Utilization measurement chain:** nvidia-smi GPU-Util (time-active, misleading) → must be paired with MFU (training, compute-bound) or MBU (decode, memory-bound) → these determine real $/FLOP or $/byte efficiency → feed into right-sizing action. (p.17–20, explicit)

- **Right-sizing decision table (Input → Process → Output):** Input = workload type (inference single-model 20–40% util / inference batched 50–70% / fine-tuning 60–80% / pre-training 80–95%) → Process = compare against target util (>60% / >75% / >80% / >90%) → Output = action if below target (multi-model serving+MIG+fractioning / tune batch+continuous batching / larger batch+gradient accumulation / optimize data loading). Source: `nvidia-smi dmon` / DCGM Exporter → Prometheus → Grafana. (p.20, explicit)

- **Discount stack mechanism:** Batch API (−50%) × Prompt caching (−90%) compound multiplicatively (not additively) → ≈95% total discount when both apply and conditions are met (24h SLA tolerance + ≥2 cache reads). (p.27, explicit)

- **Full inference cost lever stack (dependency chain):** Batching → Caching → Cascading → Quantization, each layer adds independent savings; stacked = 70–85% vs naive. This depends on prior right-sizing/util work being done first (fractional GPU + autoscaling), since otherwise idle capacity masks any per-request lever gains. (p.28, p.23, inferred from repeated "stack" framing across slides)

- **Disaggregation mechanism (prefill vs decode):** Prefill = compute-bound (long-sequence FLOPs); Decode = memory-bound (per-token, low FLOP/byte). Running combined on one GPU pool wastes one or the other → splitting into separately right-sized pools (by SKU) raises QPS 7–30×. Trade-off: benefit scales with KV-transfer overhead vs QPS/shared-context volume — small/low-QPS deployments may not benefit or could regress. (p.29, explicit)

- **Governance chain (FinOps for AI):** Visibility (tag + DCGM/token cost) → Showback (report only) → Chargeback (bill teams, needs >80% tag coverage) → $/Outcome (tie cost to win-rate). This is presented as a maturity progression, each stage a prerequisite for the next. (p.37, explicit)

- **Lifecycle model (repeated across course):** Inform → Optimize → Operate — FinOps maturity lifecycle; most AI teams are still at "Inform" stage; slide advises auditing before attempting to optimize. (p.5, p.32, explicit)

- **Power constraint chain:** Grid interconnect lead time (24–36 months) is now the binding constraint on scaling AI compute → drives site-selection strategy (place latency-tolerant workloads where power is cheap/clean) → and drives new $/MW unit economics parallel to $/GPU-hr. (p.39, explicit)

- **Cost-per-token pipeline (synthesis, from Recap slide N25/44):** Re-baseline (2026 prices, $/1M-token not $/GPU-hr) → measure correctly (MFU/MBU/goodput, not raw GPU-Util) → apply discount stack (batch×caching≈95% off) → eliminate idle via fractioning+autoscaling → govern via FOCUS + $/outcome. This is the deck's own end-to-end synthesis (Slide "Recap Chương 5"). (p.44, explicit)

## 3. Examples & Distinctions

- **Case study opener (2026 prices):** 4×H100 idle 12h/night on a neocloud at $2.50/hr = $120/day = $43,800/year; same node on a hyperscaler ($7.44/hr) = $357/day = $130k/year. Illustrates idle-cost stakes and neocloud-vs-hyperscaler price gap. (p.2, explicit)

- **H100 price collapse example:** Peak ~$8/GPU-hr (2023) → ~$2/hr neocloud (late 2025), a 64–75% drop; but in 2026 contracts reversed, 1-year commitments rose +40% ($1.70→$2.35, 10/2025→3/2026); on-demand Hopper "sold out," hyperscaler on-demand rose to ~$7.44/hr. Lesson: don't assume "wait and it gets cheaper" — the market tightened, re-baseline TCO to 2026 prices (2023-era A100 numbers inflate cost estimates 3–4×). (p.6, explicit)

- **DeepSeek headline-cost caution example:** Reported $5.576M training figure is GPU pre-training only (~2,048 H800 × 55 days), excludes CapEx/R&D; SemiAnalysis estimate of true cost ≈ $1.6B CapEx + $944M opex (~50k GPUs) — nearly 100× the headline number. Lesson: always demand full TCO, don't trust a single headline figure. (p.25, explicit)

- **Neocloud vs Hyperscaler (2026):** Hyperscaler ~$7.44/GPU-hr vs neocloud ~$2.50/hr (~3×/67% cheaper); cheapest tier: AWS $6.88 vs RunPod $2.39 / Lambda $2.49 vs spot floor $1.03 (up to ~85% cheaper). But Frontier chips reversed the pattern: B200 neocloud ~$5.09/GPU-hr, +24% in Q1 2026, volatility 11.4% (vs 0.5% for H100 hyperscaler) — a thinner, less stable market. Old "40–70% cheaper" (A100-era) rule now understates savings on mainstream chips (~3×) and Frontier silicon (3–6×). One purchasing policy does not fit every chip generation. (p.9, explicit)

- **MFU vs MBU: similarity/difference.** Similarity: both are "achieved / peak" efficiency ratios used to judge true GPU cost-efficiency beyond nvidia-smi Util%. Difference: MFU = compute-bound measure (achieved FLOPs/peak FLOPs), applies to training; MBU = memory-bound measure (achieved bandwidth/peak bandwidth), applies to memory-bound decode/inference. Distinguishing criterion: which resource the workload saturates first (FLOPs → MFU; HBM bandwidth → MBU), per the Roofline model. (p.18, explicit)

- **Showback vs Chargeback: similarity/difference.** Similarity: both are cost-allocation reporting stages within the same governance pipeline. Difference: Showback = report costs to teams without billing (an informational/awareness stage, run 4–8 weeks to fix tag gaps); Chargeback = actually bills teams internally, and requires tag-coverage >80% as a prerequisite gate. (p.37, explicit)

- **Spot vs Reserved vs On-Demand vs Capacity Block: distinguishing criterion.** Commitment length (none / none / 1–182 days / 1–3 years) and use case (interruption-tolerant + checkpointed jobs / spiky <5–6h/day / guaranteed burst at fixed price / stable high-util workload). (p.12, explicit)

- **Spot interruption rates by chip generation (example):** AWS spot −40–70% discount, ~2min notice; GCP spot/preemptible up to −80%, ~30s notice. Interruption frequency by chip: H100 <5%, A100 15–20%, V100/RTX PRO 6000 >20% — newer chips are "safer" for spot use. Recommended mixed fleet: 20% on-demand (baseline) + 80% spot (burst). (p.13, explicit)

- **Checkpoint strategy example (spot training):** Save state every epoch (or every 30 min for long epochs) to S3/GCS so any new spot instance can resume; PyTorch Lightning `ModelCheckpoint` / async checkpointing automate this; best practice is to test the resume flow before a long training run. (p.14, explicit)

- **Multi-cloud spot arbitrage example:** Managed Spot ~3× (GPU training) to 6.5× (CPU batch) cheaper via auto-recover preemption + routing to cheapest GPU; SkyServe ~50% cheaper serving cross-cloud; `sky launch task.yaml` picks the optimal provider automatically. Reserved discount comparison: AWS H100 3yr ~45% (but 72% of that reflects "old CPU pricing," i.e., stale baseline), Azure ND H100 v5 3yr ~55%, Neocloud blocks up to 60%, GCP excludes accelerators from flexible CUD. Caution flagged: don't apply a blanket CUD % across all clouds for GPUs — check per-SKU pricing before committing. (p.15, explicit)

- **Blackwell scarcity example:** B200/GB200 sold out through mid-2026, backlog ~3.6M chips; AWS Capacity Block prices rose +~20% (1/7/2026): B300 $11.70→$14.04, B200 $10.30→$12.36/acc-hr; reserved Blackwell pricing keeps climbing (not getting cheaper). Recommended policy split: scarce/new chips (Blackwell) → lock via commitment/Capacity Block early; loosened-supply chips (H100/A100) → keep flexible via spot/on-demand. GPU pricing is cyclical, not one-directional. (p.16, explicit)

- **Goodput example:** 10 requests/sec served, but only 3 meet the SLO (TTFT+TPOT) → effective goodput = 3, meaning 7 requests were paid-for compute that delivered no usable output. (p.19, explicit)

- **Serverless/scale-to-zero pricing example:** Modal H100 ~$3.95/hr, B200 ~$6.25/hr; RunPod Flex H100 PRO ~$4.18/hr; Baseten (per-minute) H100 ~$6.50/hr. Cold-start = paid dead time (20–60s init+model-load billed at full rate); idle "keep-warm" adds +26–66 W/GPU (>98% of idle power). Break-even ~30% duty cycle: spiky load → serverless is cheaper; steady high uptime → reserved is cheaper (serverless premium ~1.3–3× vs bare-metal). (p.21, explicit)

- **Buy vs Rent vs Reserve example:** Buying H100 PCIe (~$25–30k) breaks even vs renting at ~8,300 GPU-hours (~16–18 months at 100% utilization). On-demand renting cheapest unless sustained high utilization (AWS cut EC2 GPU on-demand up to 44–45%, effective 1/6/2025) — but then AWS raised Capacity Block prices ~15% in 1/2026 (p5e.48xlarge $34.61→$39.80/hr) as a "scarcity premium." Reserve (neocloud blocks) sits in between — below on-demand price, no CapEx, good for 12+ month stable workloads. The ~7-month gap between the on-demand cut and the Capacity Block price hike shows price is a real-time supply/demand signal, not a one-way trend. (p.11, explicit)

## 4. Assumptions, Boundaries & Gaps

- **Pricing figures are all dated/scenario-specific "2026" snapshots** (e.g., H100 $7.44/hr hyperscaler on-demand, B200 $5.09/hr neocloud, AWS Capacity Block price hikes) — explicitly framed by the deck itself as time-sensitive ("re-baseline mọi TCO về giá 2026"); learners should treat exact $ figures as illustrative of *mechanisms and ratios* (discount %, break-even logic), not as durable facts. (explicit framing on p.6, p.16)

- **Underlying methodology gaps not explained on slides (flag, not filled in):**
  - The 0.24 Wh / 0.03 gCO2e / 0.26 mL water "per query" baseline (p.40) — scope (which model, which hardware, whose measurement) is not stated on the slide; treat as an unexplained/uncited figure.
  - "Tokens-per-Watt" — the deck asserts it as a new unit economics KPI but does not define a formula or measurement pipeline the way it does for MFU/MBU/goodput.
  - FOCUS 1.5 (AI token consumption fields) is explicitly flagged on-slide as **not yet ratified** as of 6/2026 — meaning the "unified AI+cloud cost schema" goal is aspirational/in-progress, not a finished capability. (p.34, explicit gap flagged by deck itself)
  - "9 cost categories" claim (FinOps X 2026, p.38) — the slide names only some of the 8 non-token categories (retrieval, orchestration, KV-cache infra, evaluation, governance) via a brief caption; full enumeration/definition is not given.
  - Slide 33's "$/token low but total cost not low" point references enterprise GenAI spend growth ($1.7B 2023 → $37B 2025) without stating source/methodology.

- **Assumed prerequisite knowledge:** the deck assumes familiarity with prior course chapters — CI/CD for AI (N21), LLMOps (N22), Monitoring (N23), Governance (N24) — as this is "N25 FinOps," the 5th/final module of "Chương 5: Vận Hành" (Operations). Recap slide (p.44) explicitly shows this as N21→N22→N23→N24→N25 pipeline; this deck does not re-explain those foundations. (explicit)

- **Boundary/limitation cases flagged explicitly in-deck:**
  - Disaggregated serving (prefill/decode split) benefits scale with QPS and shared long context; small/low-traffic deployments may see no benefit or a net loss due to KV-transfer overhead (p.29).
  - Speculative decoding: helps (−48% $/1M-token) only when latency-bound; hurts (+19% cost) once throughput is already saturated — same lever, opposite effect depending on load regime (p.31).
  - Chunked prefill: small chunk size (512) adds +25% overhead; larger chunk (~2048) approaches zero overhead — parameter choice matters (p.31).
  - Prompt/KV caching: benefit is confirmed only on the read side; provider storage fees (e.g., Gemini) can flip the cost math — "cache deliberately, don't cache blindly" (p.27, p.30).
  - Serverless/scale-to-zero: cold-start (20–60s) is billed as dead time, and "keep-warm" idle power draws +26–66 W/GPU — serverless is not free of idle cost (p.21).
  - CUD/reserved discounts should not be applied as a blanket % across all clouds/SKUs for GPUs — provider- and chip-specific verification needed before commitment (p.15).
  - AWS-specific reporting gap: splitting GPU/accelerator cost is only possible via CUR 2.0 SCAD/Data Exports, not the Cost Explorer UI, as of 9/2025 — a concrete tooling limitation, not a general principle (p.36).

- **The deck is a lecture/teaching slide deck (with embedded case studies), NOT purely a schedule/calendar** — despite the source filename "Lịch cohort" (Vietnamese for "cohort schedule"), the actual content is a full technical lecture on GPU FinOps and cost optimization for AI infrastructure. The filename does not reflect the content; content is treated as source of truth per instructions.

- **End-of-deck content is administrative, not conceptual** — the final slides (Lab #25 instructions, Q&A prompt, "Cảm ơn!" thank-you slide, and pointer to Chương 6 on MCP/A2A infrastructure) are logistics, not extractable domain knowledge, and are only summarized minimally below. (p.42–45 area)

## 5. Learning Priorities

**Essential**
- $/1M-token (not $/GPU-hr) as the correct cost-normalization unit, and why $/GPU-hr is a "vanity metric" (provider arbitrage).
- MFU vs MBU vs GPU-Util%: what each measures, when to use which, and why nvidia-smi Util% is misleading.
- Break-even utilization (≈1−discount%) as the core purchasing-decision formula, and the 4-tier commitment ladder (spot/on-demand/capacity block/reserved).
- Discount stack mechanics: batching (−50%) × caching (−90%) ≈ 95% off, and the conditions required for each to apply.
- Fractional GPU sharing (MIG vs time-slicing) + K8s autoscaling stack (Karpenter/KEDA/Kueue/DRA) as the mechanism for eliminating idle GPU spend.
- FinOps governance pipeline: Visibility → Showback → Chargeback → $/Outcome, and FOCUS as the standardization layer.

**Important**
- Goodput as the SLO-aware throughput metric, and disaggregated serving (prefill/decode split) as the mechanism to raise it.
- LLMflation (non-uniform ~9×–900×/yr price drops) and the "re-baseline to current-year pricing" habit.
- Reasoning/test-time compute cost multiplier (+5–10× tokens) and its impact on $/task budgeting.
- Neocloud vs hyperscaler price gap and its volatility/instability for frontier chips (Blackwell) vs commoditized chips (H100/A100).
- Power/grid capacity as the binding constraint on AI scaling (24–36 month interconnect lead times), and $/MW as an emerging cost unit.
- Spot instance interruption-rate differences by chip generation, and checkpoint/resume strategy for spot training.

**Supporting**
- Specific tool names and version/GA dates (Kubecost 3.0, KAI Scheduler, DRA GA 1.34, FOCUS 1.2–1.4, NVIDIA Dynamo, DistServe, Mooncake).
- Exact case-study dollar figures (H100/B200/B300 pricing snapshots, AWS Capacity Block price changes) — useful as illustration, not durable facts.
- Carbon/water/PUE sustainability figures and SCI standard (ISO/IEC 21031:2024) as supplementary governance context.
- Vendor-specific tool lists for cost observability (LiteLLM, Langfuse/Helicone, Apptio, CloudZero, Vantage, Finout, nOps).
- Administrative/logistics content: Lab #25 structure, Milestone 2 demo requirements, pointer to Chương 6 (MCP/A2A infrastructure).
