# LEC_DAY23 — Disaster Recovery & High Availability for AI Infrastructure

## Keypoints
- **RTO (Recovery Time Objective)** = max tolerable downtime ("how long can we be down?"). **RPO (Recovery Point Objective)** = max tolerable data loss ("how much data can we lose?" = gap between last backup and failure).
- **No single system-wide RTO/RPO** — each AI-stack component gets its own numbers (state size + freshness differ). E.g. inference API RTO 5min/RPO N/A (stateless); vector DB RTO 15min/RPO 5min; model registry RTO 1hr/RPO 24hr.
- **RPO tightness follows RATE OF CHANGE, not importance.** Fast-changing data (vector DB embeddings, continuous) needs tight RPO; slow data (weights, ~weekly) can be loose because a stale backup still equals current state.
- **AI DR ≠ web-app DR**: (1) state to move is huge — model weights GB–TB vs DB rows KB–GB; (2) GPU standby is expensive vs cheap CPU standby; (3) cold-start 5–15min vs seconds.
- **"Stateless" ≠ "nothing to back up"** — stateless only means no per-request state; weights still must be moved + loaded into GPU (the cold-start).
- **Active-passive vs active-active**: ONLY dividing line = does the backup serve live traffic? Passive = 0% traffic (cold/pilot-light/warm all passive); active-active (hot) = both serve. Active-active needs each region sized for 100% (survivor absorbs all load).
- **Standby spectrum** (cost↔RTO): Cold (1×, 15–30min) → Pilot-light (1.1×, 8–15min) → Warm (1.3–1.5×, 3–8min) → Hot/active-active (2×, ≈0). Buy lower RTO with money.
- **Warm standby** solves cold-start by paying the slow steps (provision + weight-load) IN ADVANCE — keeps 1–2 GPU nodes running with weights pre-cached (via Karpenter/NAP). GPU instance types = g4dn/g5/p4d (NOT t3.micro — that's CPU-only).
- **DNS failover**: health check every 10–30s, 3 consecutive failures → unhealthy, low TTL (60s). Has a caching tail (ISP/resolver caches ignore TTL) → +30–90s. Require 3 failures to avoid **flapping**. Mitigate with global LB (Cloudflare/Anycast).
- **Three-layer replication** required for complete failover: DNS/LB + compute/GPU + state (S3 weights + Postgres metadata). Miss any layer = failover serves nothing.
- **S3 CRR**: replication lag → NOT for RPO<1min; checksum-verify (silent corruption possible); versioning for rollback.
- **PITR vs replica**: replica PROPAGATES logical corruption (bad migration commits successfully → mirrored); PITR rewinds to a timestamp before corruption. Different failure modes — true cross-region DR needs BOTH (PITR + cross-region replica).
- **Registry↔S3 skew** = #1 restore failure: Postgres metadata replicates faster than S3 weights → pointer references an object not yet in DR bucket → NoSuchKey/404 (dangling pointer). Fix: back up in sync, repoint DR registry to DR bucket, verify objects exist.
- **Semi-automatic failover** recommended first-line (alert + one-click confirm) — avoids catastrophic wrong-way auto-failover/flapping. Full auto only at maturity Level 4 with circuit breaker.
- **Blameless postmortem**: blame → engineers hide/delay reporting errors → worse DR. Ask "what system/process ALLOWED this?" not "who?". Use 5 whys.
- **Match architecture to SLA tier — don't overbuy 9s** (GPU cost scales non-linearly per 9): 99% → single region; 99.9% → multi-AZ + auto failover (medium); 99.95% → warm standby (high); 99.99% → active-active (very high).
- **Availability math**: downtime/yr = (1 − SLA%) × 525,600 min. 99.9% = 8.76hr/yr; 99.99% = 52.6min/yr. Actual downtime = RTO × incident frequency (outages are RARE, ~1–2/yr — not continuous).
- **Untested plan = hypothesis, not truth.** "Backup chưa test = không có backup." Real RTO typically 2–3× paper RTO (unmodeled delays + human fumbling + silent gaps).
- **DR Maturity Model**: L0 no plan → L1 written-untested → L2 semi-auto failover → L3 periodic drills → L4 chaos-engineered auto. Target L2–3 for most; L4 only for 99.99%+.
- **Chaos engineering** safety: staging first, kill switch, notify on-call — never a production surprise.

## Terms
- RTO — Recovery Time Objective (Mục tiêu thời gian phục hồi) — max tolerable downtime.
- RPO — Recovery Point Objective (Mục tiêu điểm phục hồi) — max tolerable data loss window.
- HA — High Availability (Tính sẵn sàng cao) — preventing downtime via redundancy (vs DR = recovering from disaster).
- AZ — Availability Zone (Vùng sẵn sàng) — physically isolated data center within a region; multi-AZ survives one DC failure.
- CRR — Cross-Region Replication — S3 auto-copy of objects to a DR-region bucket (has lag).
- PITR — Point-in-Time Recovery — restore DB to any second in a retention window (e.g. 35 days).
- Flapping (dao động/bập bênh) — regions toggling up/down repeatedly from noisy health checks.
- Warm standby — 1–2 GPU nodes kept running with weights pre-loaded, 0 traffic.
- Game day / DR drill — controlled simulated outage to measure real RTO/RPO.
- Blameless postmortem (không đổ lỗi) — systemic root-cause review, no individual blame.
- Karpenter/NAP — node autoscaler (from Day 16) keeping GPU nodes warm.

## Covered / To revisit
- [x] RTO vs RPO definitions and per-component principle — solid
- [x] RPO rate-of-change rule — solid after correction (applied cleanly to feature store)
- [x] AI DR vs web DR (weights size, GPU standby cost) — solid after dropping the non-determinism misconception
- [x] "Stateless ≠ nothing to back up" — solid, connected across sections
- [x] Active-passive vs active-active + 100% capacity sizing — solid
- [ ] Warm vs hot / standby-spectrum ↔ active-passive-vs-active-active mapping — corrected mid-session (had thought warm = active-active); re-verify this stays clear
- [x] DNS failover mechanics, TTL, caching tail, flapping — strong
- [x] Three-layer replication requirement — solid (car-without-steering-wheel analogy)
- [x] S3 CRR limits (lag, checksum, versioning) — solid
- [x] PITR vs replica for logical corruption — very strong, nailed first try
- [x] Registry↔S3 dangling-pointer trap — solid (minor refinement on 404 vs old-version)
- [x] Semi-automatic failover + runbook — strong trade-off reasoning
- [x] Blameless postmortem rationale — solid (+ git blame analogy)
- [ ] Match-architecture-to-SLA / don't-overbuy-9s — corrected mid-session (had thought 99.9% justified active-active hot); re-verify tier→architecture mapping
- [ ] RTO-per-incident vs yearly downtime budget — corrected mid-session (had multiplied RTO as if continuous); re-verify the availability math intuition
- [x] DR drills, untested=hypothesis, 2–3× real RTO, maturity model — solid (nailed final transfer, Level 1)

## Misconceptions / Examiner findings
- **RTO/RPO flipped for model registry** — initially said RPO tight / RTO loose for slow-changing weights; actually BOTH loose. Gap: was mapping RPO tightness to importance instead of rate-of-change. Fixed via feature-store transfer example.
- **AI DR "harder because LLM non-deterministic"** — misconception; output non-determinism is unrelated to DR (infra failures are diagnosable). Dropped.
- **Warm standby = active-active** — thought warm and hot were both active-active differing only by project size. Corrected: warm/cold/pilot-light are active-PASSIVE (0 backup traffic); only hot is active-active. The test = does backup serve live traffic.
- **Over-engineering 99.9% with active-active hot** — thought "99.9% is good, so active-active is valid." Missed that 99.9% only needs multi-AZ + auto failover (medium cost); active-active is the 99.99% tier. Don't overbuy 9s.
- **RTO × 15 days × 12 months** — computed yearly downtime by treating 30-min RTO as recurring every 15 days. Gap: conflated per-incident RTO with continuous/scheduled downtime. Fixed: yearly downtime = RTO × rare incident frequency (~1–2/yr).
- Minor: registry restore failure described as "pointer to older version"; actually a dangling pointer → 404/NoSuchKey (object missing), not silent downgrade.

## Session meta
- Session 1 (2026-08-25): Full lecture covered in one session (~1.5hr). Deck 42pp → extracted via lecture-extractor (Sonnet). Filename `stream.pdf` is misleading — content is DR/HA, not stream processing. Student engaged deeply, asked strong meta + grounding questions (AWS instance types, availability math). Four misconceptions surfaced and corrected in-session; three flagged for re-verify next time.
