# Core Knowledge

**Lecture:** Disaster Recovery & High Availability cho AI Infrastructure
**Course context:** AICB-P2T2 · Ngày 23 · Chương 5: Vận Hành (AI Infrastructure track, Phase 2 Track 2, Tuần 5)
**Deck length:** 42 pages (31 numbered slides + cover/Q&A)

## 1. Core Concepts

- **RTO (Recovery Time Objective)** — "Tối đa bao lâu được downtime?" Measured from when outage begins to when service is restored. Explicit examples: model serving RTO ~5–15 min (customer SLA); training pipeline RTO can be hours (not real-time). (p.6, p.7)
- **RPO (Recovery Point Objective)** — "Tối đa mất bao nhiêu dữ liệu?" Measured as the gap between the most recent backup and the moment of failure. Explicit examples: vector DB RPO ~minutes (embeddings ingested continuously); model registry RPO can be hours (models change infrequently). (p.6, p.7)
- **Per-component RTO/RPO table** — Core principle: there is no single RTO/RPO for a whole system; each AI-stack component needs its own number, because state size and freshness needs differ. Example values (p.7, illustrative, adjust to real SLA):
  - Inference API (serving): RTO 5 min, RPO N/A (stateless) — user-facing, strict SLA
  - Vector DB: RTO 15 min, RPO 5 min — needs fresh embeddings but replica lag tolerable
  - Feature store: RTO 30 min, RPO 15 min — batch features tolerate more lag
  - Model registry/weights: RTO 1 hour, RPO 24 hours — models change rarely between training runs
- **Availability tiers ("9"s)** — SLA percentage maps to allowed downtime/year, required architecture, and cost (explicit table, p.8):
  - 99% → 3.65 days/yr → single region + periodic backup → low cost
  - 99.9% → 8.76 hr/yr → multi-AZ + automated failover → medium cost
  - 99.95% → 4.38 hr/yr → multi-AZ + warm standby region → high cost
  - 99.99% → 52.6 min/yr → active-active multi-region → very high cost
  Explicit warning: don't design for 99.99% if real SLA only needs 99.9% — GPU standby cost scales non-linearly with each extra "9".
- **Active-Passive deployment** — One region (A) fully active (100% traffic), a second region (B) standby (0% traffic) that replicates from A. Cheaper, simpler; failover takes minutes (DNS cutover). (p.9)
- **Active-Active deployment** — Two regions (C, D) both serving live traffic (e.g., 50/50 split), kept in sync. RTO ≈ 0 but costs ~2x, requires conflict resolution for state. (p.9)
- **DNS/Global-LB failover (Route53/Cloud DNS)** — Health check hits endpoint every 10–30s; 3 consecutive failures → marked unhealthy; low DNS TTL (60s) enables fast cutover; latency-based routing sends traffic to nearest healthy region. (p.10)
- **Cross-Region Replication (CRR) for model weights** — Terraform-managed S3 replication (`aws_s3_bucket_replication_configuration`) copies checkpoints (prefix-filtered) from primary bucket to a DR-region bucket. Requires versioning enabled for rollback to older checkpoints. Has replication lag — not suitable for RPO < 1 minute. Requires checksum verification post-replicate since silent corruption is possible. (p.11)
- **Warm GPU standby / pool warm-up** — Keeping 1–2 GPU nodes "warm" in the secondary region (via Karpenter/NAP, referenced from Day 16) with model weights pre-loaded into node cache, scaled 0→N on failover, avoiding the cold-start penalty of provisioning + loading weights from S3. (p.19)
- **PITR (Point-in-Time Recovery)** — RDS/Aurora continuous backup + transaction log allows restoring to any second within a retention window (explicit: 35 days). Used for feature registry, experiment tracking DB, model registry metadata. Restoring creates a *new* instance; it does not overwrite the running one. (p.16)
- **Blameless postmortem** — Structured post-incident review template: timeline, RTO measured vs. target (and where the gap occurred), root cause via "5 whys" (no individual blame), action items with owner + deadline. Rationale: blaming individuals causes people to hide errors instead of reporting early; the right question is which system/process allowed the failure. (p.22)
- **Standby capacity strategies (cost/RTO spectrum)** — Explicit table (p.23):
  - Cold: provision from scratch on failover, RTO 15–30 min, cost 1x (baseline, only primary region running)
  - Pilot-light: metadata/config kept warm, GPU scaled on demand, RTO 8–15 min, cost 1.1x
  - Warm standby: 1–2 GPU nodes kept warm, scale fast, RTO 3–8 min, cost 1.3–1.5x
  - Hot (active-active): full capacity in both regions simultaneously, RTO ≈ 0, cost 2x
- **DR Drill / Game Day** — Deliberately triggering a controlled simulated outage to measure real RTO/RPO against targets and find runbook gaps before customers do. 4-step process: plan → notify team → trigger simulated outage → measure & debrief. (p.25–26)
- **Chaos engineering (lightweight, for AI infra)** — Low-risk fault injection examples: kill a GPU serving pod (verify HPA/K8s self-heals), inject latency into vector DB calls (verify timeout/fallback), block network to primary region via chaos mesh (verify DNS failover). Safety principles: always run in staging before production, have a kill switch, notify on-call in advance. (p.27)
- **DR Maturity Model** — 5 levels (explicit table, p.28):
  - Level 0: No plan — manual backup, nobody knows real RTO
  - Level 1: Runbook written — documented but never tested
  - Level 2: Partially automated failover — health check + DNS cutover, requires human confirm
  - Level 3: Periodic testing (game day) — quarterly DR drills, measured RTO, updated runbook
  - Level 4: Chaos-engineered — frequent fault injection, failover with no human intervention
  Explicit guidance: realistic target for most teams is Level 2–3; Level 4 only worth investing in when SLA requires 99.99%+.

## 2. Relationships & Mechanisms

- **AI infra vs. typical web app (why DR differs)** — Comparison table (p.4, explicit):
  | Dimension | Web app | AI system |
  |---|---|---|
  | State to recover | DB rows (KB–GB) | Model weights (GB–TB) |
  | "Restart" time | Seconds | Cold-start GPU pool: 5–15 min |
  | "Fresh" data that matters | Transaction log | Vector DB embeddings + feature store freshness |
  | Standby cost | Cheap (CPU instance) | Expensive (GPU instance idling) |
  Consequence: AI DR cannot copy-paste the web-app playbook — must separately account for GPU standby cost and state reload time.
- **Trade-off: RTO/RPO vs. cost** — Lower RTO/RPO → higher infra cost (explicit principle, p.6, and reinforced by the standby-strategy table p.23 and availability-tier table p.8).
- **Decision framework: choosing a standby strategy** (p.24, explicit flowchart):
  RTO target < 5 min? → Yes → Warm/Hot standby. → No → Is 15–30 min downtime acceptable? → Yes → Cold/Pilot-light. → No → re-budget or lower the SLA target.
  Framing: the real question isn't "best possible RTO" but "what RTO is good enough, at a cost the company accepts."
- **Reference architecture (full system, p.13)** — Ties together: Global DNS/LB (health check every 15s, triggers failover) → Region A (ACTIVE: serving + vector DB) replicates to → Region B (STANDBY: warm GPU pool), via two parallel state-replication paths: (1) S3 CRR for model weights + vector DB snapshot, (2) Postgres PITR for registry + metadata. Key point: DNS/LB, compute, and state (S3 + Postgres) are three separate layers that must all be replicated — missing any one layer makes failover incomplete.
- **Health-check-based failover mechanism (Input → Process → Output, p.18)**:
  Input: Health Checker pings primary region every 15s → Process: on failure, DNS/Global LB triggers cutover to secondary region (which was warm standby) + PagerDuty/Slack alerts on-call and triggers the runbook → Output: traffic served from secondary region.
  Principle: first-line failover should be semi-automatic (alert + one-click confirm), not fully automatic — avoids flapping (regions toggling back and forth).
- **GPU cold-start chain (why it threatens RTO, p.19)**: Provision new GPU instance (3–8 min) → pull image + load model weights (2–10 min) → total cold-start can exceed the RTO target. Mitigated by warm standby (pre-loaded weights in node cache via Karpenter/NAP).
- **Runbook: Region-Down Failover (checklist, p.20)** — Sequential process: (1) confirm outage via health check + cloud provider status page → (2) notify incident channel + start RTO clock → (3) scale secondary-region GPU pool from warm to full capacity → (4) verify model weights + vector DB replica are recently synced → (5) DNS/LB cutover traffic to secondary region → (6) verify golden signals (latency, error rate) stable in secondary region → (7) post-incident: measure actual RTO vs. target, write postmortem.
- **Model registry ↔ artifact storage dependency (p.15)** — MLflow Model Registry (Postgres metadata) and S3 model artifacts must be backed up in sync/together; the most common restore failure is registry metadata pointing to an S3 path that no longer exists (async replication mismatch). Recovery flow: S3 Primary Region → async replicate → S3 DR Region (CRR replica); DR region registry metadata (Postgres) is restored from snapshot and repointed to the S3 DR bucket.
- **Vector DB recovery mechanisms depend on hosting model (p.14)**: managed (Pinecone: near-real-time replica pod in another region; Weaviate: snapshot backup to S3/GCS + restore to new cluster) vs. self-hosted (Qdrant/Milvus: periodic snapshot + WAL shipping). Re-indexing from raw documents is always a fallback but is slow (hours, not minutes).
- **Backup schedule ↔ RPO alignment (p.17, explicit table)**: Model weights — S3 CRR + versioning, continuous, 90-day retention; Vector DB — snapshot to S3, every 6 hours, 30-day retention; Metadata (Postgres) — PITR + cross-region replica, continuous, 35-day retention; Feature store (offline) — table snapshot, daily, 14-day retention. Principle: adjust frequency to each component's target RPO.

## 3. Examples & Distinctions

- **Case study: AWS us-east-1, Dec 2021 (p.5)** — Internal network incident disrupted us-east-1 for ~7 hours; many AI/SaaS companies running inference there had full downtime because they had no secondary region, or had one but had never tested failover. What happened: dashboard/monitoring was co-hosted in the same region (so teams didn't know they were down); DNS failover existed but hadn't been tested, so the first real cutover failed. Lessons: observability stack must live in a different region from the workload it monitors; "having a DR plan" and "DR plan works" are two different things.
- **Live Demo: Region Failover Drill (p.29)** — 2 staging regions with serving + vector DB replica in both; traffic to primary region is blocked to simulate outage while RTO clock starts; observe health check detecting failure → alert → DNS cutover; verify new requests served from secondary region with stable latency/error rate; compare measured RTO against the 5-minute target and record the gap.
- **Active-Passive vs Active-Active: when to use which (p.12, explicit)**:
  - Active-Passive fits: target RTO > 5 min acceptable; limited GPU budget (can't run two full-time pools); state has low conflict risk (no multi-master needed); suits most AI startups / mid-size teams.
  - Active-Active fits: target RTO ≈ 0 (fintech, real-time healthcare); budget allows double GPU capacity; team has a conflict-resolution strategy for vector DB/feature store; suits enterprise SLA 99.99%+.
- **Anti-patterns (p.21, explicit, four distinct failure modes)**:
  1. Runbook exists only on paper, never tested → ~90% chance of executing wrong steps during a real, panicked incident.
  2. Automated failover with no circuit breaker → two regions flap back and forth when health checks are unstable.
  3. DR region backed up under the same account/credentials as primary → one IAM/billing incident can take down both regions simultaneously.
  4. Nobody knows the real RTO — only the "theoretical" number on a slide, never measured via an actual drill.
- **RTO vs RPO (distinction)** — similarity: both are DR target metrics defined per component. Difference: RTO measures allowed *downtime duration*; RPO measures allowed *data loss window*. Distinguishing question: RTO answers "how long can we be down?", RPO answers "how much data can we lose?" (p.6)
- **Cold vs Pilot-light vs Warm standby vs Hot (active-active)** — similarity: all are standby-capacity strategies trading cost for RTO. Difference: they form a spectrum from "provision only on failure" (Cold, 1x cost, 15–30 min RTO) to "always fully running in both regions" (Hot, 2x cost, ≈0 RTO), with Pilot-light and Warm standby as intermediate points. (p.23)
- **PITR restore vs. replica (distinction, p.16)** — A cross-region *replica* continuously mirrors data including logical corruption (e.g., a bad migration propagates to the replica too). *PITR* restore recreates state as of a specific past timestamp, which can recover from logical corruption that a live replica cannot. Note: PITR alone restores within the same region — true cross-region DR needs PITR combined with a cross-region read replica.

## 4. Assumptions, Boundaries & Gaps

- **Assumption**: RTO/RPO figures given throughout (e.g., serving 5 min, vector DB 15 min/5 min RPO, availability tier table) are explicitly labeled as illustrative ("Số liệu minh hoạ") — to be adjusted to the actual system's real SLA, not treated as universal defaults. (p.7, p.8)
- **Boundary/limitation — DNS failover**: DNS caching at client/ISP level doesn't always respect TTL, so some users still experience the old (down) endpoint even after cutover; failover is not instantaneous — adds an extra 30–90s to RTO on top of health-check detection time. Mitigation mentioned: combine with a global load balancer (Cloudflare/Anycast) for faster cutover than DNS alone. (p.10)
- **Boundary — S3 CRR**: replication lag makes CRR unsuitable for RPO targets under 1 minute; corruption during replication can be silent (must checksum-verify). (p.11)
- **Gap (flagged, not explained in source)**: The deck does not explain *how* conflict resolution for active-active state (vector DB / feature store) is actually implemented — it's named as a requirement (p.9, p.12) but no mechanism, algorithm, or tool is given.
- **Gap (flagged)**: "Karpenter/NAP" is referenced as "đã học Ngày 16" (already covered Day 16) but this deck does not re-explain what it is beyond "keeps 1–2 warm GPU nodes" — assumes prior knowledge from an earlier lecture.
- **Gap (flagged)**: The specific mechanics of Pinecone/Weaviate/Qdrant/Milvus replication (e.g., exact consistency guarantees, how WAL shipping restores state) are only briefly named, not detailed. (p.14)
- **Edge case explicitly called out**: A model registry pointing to a non-existent S3 path (due to async replication mismatch) is named as "the most common restore failure." (p.15)
- **Edge case explicitly called out**: Logical corruption (e.g., a bad migration) will propagate through a synchronous replica into the DR region too — a live replica alone is not sufficient DR protection; a separate point-in-time backup is needed. (p.16)
- **Prerequisite/assumption underlying entire lecture**: Backup or DR plans that have not been restore-tested / drilled are treated as invalid — repeated explicitly at multiple points (p.14: "backup chưa test = không có backup"; p.21 anti-pattern 1; p.25: "Runbook chưa test = giả định, không phải sự thật"; DR Maturity Model p.28 distinguishes Level 1 "written but untested" from Level 3 "periodically tested"). Real RTO is reported to typically run 2–3x higher than the "paper" RTO before drilling. (p.25)
- **Constraint stated for chaos engineering**: must run in staging first, have a kill switch, and notify on-call in advance — not to be run as a surprise on production without these safeguards. (p.27)

## 5. Learning Priorities

**Essential**
- RTO vs RPO definitions and why each AI-stack component needs its own value (no single number for the whole system)
- AI infra vs. web app DR differences (state size, cold-start time, standby cost)
- Active-Passive vs Active-Active patterns and when each is appropriate
- The three-layer replication requirement (DNS/LB, compute/GPU, state = S3 + Postgres) — reference architecture
- Runbook checklist for region-down failover
- DR Drill / Game Day concept: untested plans ≠ real RTO; must measure to know actual RTO
- Cost/RTO trade-off across Cold / Pilot-light / Warm / Hot standby strategies

**Important**
- Availability tiers ("9"s) mapped to architecture and cost
- Cross-region model weight replication via S3 CRR (mechanism, limitations: lag, checksum, versioning)
- PITR mechanism and its limitation for true cross-region DR
- Backup schedule cheatsheet (method/frequency/retention per component)
- Blameless postmortem process and rationale
- DNS/Global-LB failover mechanics and real-world limitations (client DNS caching, added latency)
- DR Maturity Model levels (0–4) and realistic target (Level 2–3)
- Anti-patterns (untested runbook, no circuit breaker/flapping, shared-account DR, unmeasured RTO)

**Supporting**
- AWS us-east-1 Dec 2021 case study (illustrative, reinforces "test your failover" lesson)
- Vector DB vendor-specific backup approaches (Pinecone/Weaviate/Qdrant/Milvus)
- Chaos engineering fault-injection examples and safety principles
- Live demo walkthrough structure (p.29) — illustrative of the drill process already covered conceptually
- Decision-framework flowchart (p.24) — restates the cost/RTO trade-off already covered
