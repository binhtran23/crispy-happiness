# LEC_DAY23 — Quiz

## Q1 [priority: Essential]
**Question:** What is the core distinction between RTO and RPO?
- A) RTO measures data-loss tolerance; RPO measures downtime tolerance
- B) RTO measures downtime tolerance ("how long can we be down?"); RPO measures data-loss tolerance ("how much data can we lose?")
- C) RTO applies to stateful components only; RPO applies to stateless components only
- D) RTO and RPO are two names for the same recovery metric

**Answer:** B
**Explanation:** RTO = recovery *time* (downtime duration); RPO = recovery *point* (data-loss window = gap between last backup and failure). A reverses them. C is wrong — stateless components have an RTO but N/A RPO. D is wrong — they measure different things and each component gets its own values.

## Q2 [priority: Essential] [weak-area]
**Question:** A model registry stores weights that only change ~once a week. Which RTO/RPO profile is correct, and why?
- A) Tight RPO, loose RTO — losing any weight data is catastrophic
- B) Tight RPO, tight RTO — the registry is business-critical
- C) Loose RPO, loose RTO — weights change slowly, so a stale backup ≈ current state, and already-serving models keep running if the registry is briefly down
- D) Loose RPO, tight RTO — data loss is fine but the registry must never be down

**Answer:** C
**Explanation:** RPO tightness follows *rate of change*, not importance. Weekly-changing weights mean a 24h-old backup is essentially identical to live → loose RPO is safe. RTO can also be loose because already-loaded serving models keep running; the registry is only needed to deploy new models. A/B wrongly tie RPO to importance. D confuses the registry's RTO with the inference API's strict RTO.

## Q3 [priority: Essential]
**Question:** Which two factors make AI disaster recovery harder/more expensive than typical web-app DR?
- A) LLM non-determinism and unpredictable crash causes
- B) Huge state to move (model weights GB–TB) and expensive GPU standby cost
- C) More frequent outages and stricter legal requirements
- D) Larger transaction logs and more database rows

**Answer:** B
**Explanation:** The slide's comparison: state to recover is GB–TB weights (vs KB–GB rows) and standby means an idling GPU (expensive) vs a cheap CPU. A is a distractor — output non-determinism is unrelated to DR; infra failures are diagnosable. C/D describe web-app-scale concerns, not the AI-specific pain.

## Q4 [priority: Essential] [weak-area]
**Question:** What is the ONLY defining difference between an active-passive and an active-active deployment?
- A) Whether the backup region has model weights loaded
- B) Whether the backup region's GPU nodes are running
- C) Whether the backup region serves live traffic
- D) The size/budget of the project

**Answer:** C
**Explanation:** The single test is traffic: active-passive backup gets 0% traffic (even if GPUs are running warm); active-active (hot) serves live traffic in both regions. B is the trap — warm standby has running GPUs but zero traffic, so it's still active-*passive*. A is a partial detail. D is wrong — RTO target vs budget drives the choice, not project size.

## Q5 [priority: Essential] [weak-area]
**Question:** Cold, pilot-light, warm, and hot are the four standby strategies. Which grouping by deployment pattern is correct?
- A) All four are active-active variants
- B) Cold and pilot-light are active-passive; warm and hot are active-active
- C) Cold, pilot-light, and warm are active-passive; only hot is active-active
- D) Only cold is active-passive; the rest are active-active

**Answer:** C
**Explanation:** The dividing line is live traffic to the backup. Cold, pilot-light, and warm all keep the backup at 0% traffic (they differ only in how much is pre-warmed to speed failover) → active-passive. Only hot (full capacity serving in both regions, ~2× cost, RTO≈0) is active-active. B wrongly promotes warm to active-active.

## Q6 [priority: Essential]
**Question:** In an active-active setup, why must EACH region be provisioned to handle 100% of traffic (rather than 50% each)?
- A) To reduce GPU cost by sharing load
- B) Because when one region fails, the survivor must absorb 100% of traffic; a 50%-sized region would overload and crash
- C) Because DNS cannot split traffic evenly
- D) Because model weights can only load on full-capacity nodes

**Answer:** B
**Explanation:** After one region dies, all traffic lands on the survivor. If it was sized for only 50%, it gets overwhelmed → latency spikes, failures, often a total outage — the failover makes things worse. This is why active-active genuinely costs ~2×; the headroom is deliberate, not waste. A inverts the reasoning.

## Q7 [priority: Essential]
**Question:** A team configures DNS failover and a warm GPU pool in the backup region, but forgets to replicate state (S3 + Postgres). The primary dies and failover triggers. What happens?
- A) Failover works perfectly — DNS and compute are all that matter
- B) The backup GPU pool spins up but has no model weights/data to serve → user requests fail
- C) The database automatically rebuilds from the DNS records
- D) Traffic silently returns to the dead primary region

**Answer:** B
**Explanation:** Complete failover needs all three layers — DNS/LB, compute/GPU, AND state. GPUs without weights = "a car with no steering wheel." Missing the state layer means the backup serves nothing. A ignores the state layer; C/D are invented behaviors.

## Q8 [priority: Important]
**Question:** Why is S3 Cross-Region Replication (CRR) unsuitable for a component needing RPO < 1 minute?
- A) CRR only copies files once per day
- B) CRR's replication lag can exceed 1 minute, so a write just before failure may not have reached the DR bucket → lost data
- C) CRR does not support model weights
- D) CRR overwrites older versions, violating RPO

**Answer:** B
**Explanation:** Replication lag *is* the RPO floor — if lag can exceed 1 min, you can't guarantee sub-minute RPO. A is false (CRR is continuous, not daily). C is false (weights are the main CRR use case). D is false — versioning preserves old copies.

## Q9 [priority: Important]
**Question:** At 3 PM a valid-but-wrong database migration corrupts a column. You have a live cross-region replica AND PITR. Which recovers you, and why?
- A) The replica — it has a clean untouched copy
- B) PITR — restore to just before 3 PM; the replica cannot help because it faithfully propagated the successfully-committed corruption
- C) Both work equally — corruption is corruption
- D) Neither — logical corruption is unrecoverable

**Answer:** B
**Explanation:** A replica mirrors *everything*, including successful-but-wrong operations, so the corruption propagates to it — you get two bad copies. PITR rewinds to a timestamp before the migration. Replicas protect against region/hardware failure; PITR protects against logical corruption. (True cross-region DR needs both.)

## Q10 [priority: Important]
**Question:** The MLflow registry keeps metadata in Postgres and weights in S3, replicated independently. What is the "most common restore failure" on failover?
- A) The GPU pool runs out of memory loading weights
- B) Postgres metadata (replicated faster) points to an S3 object that hasn't replicated yet → dangling pointer → NoSuchKey/404
- C) The DNS TTL is too high
- D) The vector DB re-index takes hours

**Answer:** B
**Explanation:** Metadata races ahead of artifacts; the pointer references a weight file not yet in the DR bucket, so the load fails with a missing-object error. Fix: back up in sync, repoint DR metadata to the DR bucket, verify objects exist before completing failover. The other options are unrelated failure modes.

## Q11 [priority: Important]
**Question:** Why does the lecture recommend SEMI-automatic (alert + one-click confirm) failover as the first-line default rather than fully automatic?
- A) Fully automatic failover is technically impossible
- B) A human confirmation prevents catastrophic wrong-way failover / flapping when health checks are noisy, while the system still auto-handles the slow prep
- C) Semi-automatic is cheaper on GPU cost
- D) Regulations forbid automatic failover

**Answer:** B
**Explanation:** Noisy health checks can trigger unnecessary cutovers and flapping (regions toggling); a human go/no-go call filters false alarms while the system does detection + warmup automatically. Full auto is reserved for maturity Level 4 with a circuit breaker. The 3 AM on-call cost is real but judged worth it.

## Q12 [priority: Important] [weak-area]
**Question:** A business needs 99.9% availability. An engineer proposes active-active hot standby (≈0 RTO, 2× cost) "to be safe." What's wrong?
- A) Nothing — more availability is always better
- B) Active-active only reaches 99% availability, which is too low
- C) It over-engineers: 99.9% only needs multi-AZ + automated failover (medium cost); active-active is the 99.99% tier — paying 2× for a "9" the SLA never requires
- D) 99.9% requires active-active; the engineer under-provisioned

**Answer:** C
**Explanation:** Match architecture to the SLA tier. 99.9% (8.76 hr/yr) is met by multi-AZ + auto failover — even cold/pilot-light standby fits. Active-active is for 99.99%. GPU cost scales non-linearly per 9, so overbuying is pure waste. A ignores cost; "to be safe" is not a budget justification.

## Q13 [priority: Important] [weak-area]
**Question:** A cold standby has a 30-minute RTO. The team has ~2 major outages per year. Roughly what yearly downtime is that, and does it meet a 99.9% SLA (8.76 hr/yr budget)?
- A) 30 min × 24 per day = far over budget; fails 99.9%
- B) 30 min × ~2 incidents = ~60 min/year total; comfortably within the 8.76 hr budget
- C) 30 min continuous every day; fails badly
- D) Cannot be computed without knowing the RPO

**Answer:** B
**Explanation:** RTO is *per-incident*, not continuous. Yearly downtime = RTO × incident frequency. Outages are rare (~1–2/yr), so 30 min × 2 ≈ 60 min/year — well under 8.76 hr. A and C wrongly treat RTO as recurring constantly. D confuses RTO with RPO.

## Q14 [priority: Important]
**Question:** The lecture says real, measured RTO is typically what, relative to the "paper" RTO on the slide?
- A) About the same — plans are usually accurate
- B) 2–3× HIGHER, because untested plans miss real delays (DNS caching, unsynced state, human fumbling) that only surface in a drill
- C) 2–3× LOWER, because teams over-estimate to be safe
- D) Exactly equal once backups are configured

**Answer:** B
**Explanation:** An untested plan is a hypothesis. Real drills expose unmodeled delays and gaps, so actual RTO runs 2–3× the paper number. Hence "backup chưa test = không có backup." A/D assume documentation equals reality; C reverses the direction.

## Q15 [priority: Important]
**Question:** A team has a documented runbook, configured backups, and a warm standby region — but has never run a drill. What DR maturity level are they at?
- A) Level 0 — no plan
- B) Level 1 — runbook written but never tested
- C) Level 3 — periodic testing
- D) Level 4 — chaos-engineered

**Answer:** B
**Explanation:** Level 1 = documented but untested. They have all the artifacts but zero evidence any of it works (unproven RTO, possibly non-restorable backups). Level 3 requires actual periodic drills with measured RTO; Level 4 adds chaos engineering and automatic failover.

## Q16 [priority: Important]
**Question:** Why does the lecture insist postmortems be BLAMELESS?
- A) Blame is unprofessional and hurts morale, but has no operational impact
- B) Blaming individuals makes engineers hide or delay reporting errors, which worsens future DR; the useful question is which system/process allowed the failure
- C) Blameless postmortems are faster to write
- D) It avoids legal liability for the company

**Answer:** B
**Explanation:** Fear of punishment → people conceal incidents and report late, which is catastrophic since early reporting shrinks outages. Blameless review (with 5 whys) targets the systemic gap, not a scapegoat, so the actual hole gets fixed. A understates the operational harm; C/D miss the point.

## Q17 [priority: Supporting]
**Question:** The AWS us-east-1 Dec 2021 outage taught which specific lesson about observability?
- A) Always use multiple cloud providers
- B) Your monitoring/observability stack must live in a DIFFERENT region from the workload it monitors — otherwise you can't even tell you're down
- C) DNS failover is unnecessary if you have backups
- D) GPU standby should always be hot

**Answer:** B
**Explanation:** Teams had dashboards co-hosted in the region that went down, so they couldn't detect their own outage; their untested DNS failover also failed on first real use. Lesson: observability lives elsewhere, and "having a DR plan" ≠ "DR plan works." The others aren't the stated lesson.

## Q18 [priority: Supporting]
**Question:** Which is a REQUIRED safety practice for chaos engineering on AI infrastructure?
- A) Always run it first on production to get realistic results
- B) Run in staging first, have a kill switch, and notify on-call in advance
- C) Never notify anyone, to test true readiness
- D) Only inject faults during business hours without warning

**Answer:** B
**Explanation:** Chaos engineering must be safe: staging before production, a kill switch to stop the experiment, and advance on-call notification. Surprise production fault injection (A, C, D) is explicitly an anti-practice — it risks a real outage instead of a controlled test.
