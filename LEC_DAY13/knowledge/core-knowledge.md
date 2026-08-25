# Core Knowledge — Day 13: Monitoring, Logging & Observability

Source: `day13-monitoring-logging-observability_v2.pdf` (96 content slides + Q&A, 114 pages total). Language: Vietnamese slides with English technical terms; this extraction preserves original terms where they carry precise technical meaning.

---

## 1. Core Concepts

### Monitoring vs Observability (explicit, p.5)
- **Monitoring** — theo dõi các câu hỏi đã biết trước (tracks known-in-advance questions). Dashboard + alert dựng sẵn. Answers "Is X broken?" Good for anticipated failure modes ("known-knowns").
- **Observability** — a *property of the system*: lets you ask *new* questions without deploying new code, given sufficiently rich telemetry (metrics + logs + traces). Answers "WHY is X broken?" Good for failure modes never seen before ("unknown-unknowns").
- Why it matters: production AI agents fail in novel ways (hallucination, tool-arg hallucination, prompt injection) that traditional APM has no concept of — you need the observability property, not just canned dashboards.

### Why AI agent observability differs from traditional software monitoring (explicit, p.6)
Four distinguishing properties of AI systems:
1. Same input → different output each time; can't test via string comparison, must measure *quality* not just pass/fail.
2. Every request costs money proportional to token count; a bug loop can burn budget in hours — CPU/RAM metrics won't reveal this.
3. App doesn't "crash" — it still returns 200 OK while answer quality degrades; no exception to catch.
4. New failure classes traditional APM has no concept of: hallucinated tool args, infinite loops, context overflow, prompt injection.

### 3 Pillars of Observability + Pillar 4 (explicit, p.10–11)
- **Metrics** — đo lường: how much / how long (latency, error rate, cost/day).
- **Logs** — ghi chép: what happened (input, output, errors, timestamps).
- **Traces** — theo dõi: why / where — end-to-end journey, bottleneck, root cause.
- **Pillar 4 (AI-specific): Continuous/Online Eval** — the three traditional pillars cannot answer the most important question for an AI system: "is the answer still correct?" HTTP 200 ≠ correctness; low latency ≠ useful; 0% error rate ≠ not burning money. Pillar 4 measures output quality continuously in production. (Day 13 = online/continuous eval; Day 14 = offline eval with a fixed benchmark — the two complement each other, explicit.)

### Control theory feedback loop (explicit, p.9)
Agent System → Observe (metrics) → Analyze (compare) → Act (fix/scale) → feeds back to Agent System.
- **MTTD (Mean Time To Detect)** — time from incident occurring to detection.
- **MTTR (Mean Time To Recover)** — time from detection to fix. Goal of good observability = minimize both.

### AI-Specific Metrics — 4 groups (explicit, p.14)
- **Performance**: Latency P50/P95/P99, TTFT (Time To First Token), throughput (req/s, tokens/s), LLM call duration.
- **Cost**: tokens per request (in/out), cost per request/task, cost per day/user/feature, cache hit rate.
- **Quality** (pillar 4): hallucination/faithfulness, task completion rate, thumbs up/down & regenerate rate, guardrail trigger rate.
- **Reliability**: error rate, uptime, tool-call success/failure rate, retry rate, loop rate, retrieval recall/empty-result rate.

### 4 Golden Signals + 2 for AI agents (explicit, p.15)
Google SRE's 4 Golden Signals: Latency, Traffic (QPS), Errors, Saturation.
AI agents need 2 more: **Cost** ($/request, $/user, token usage) and **Quality** (hallucination rate, CSAT, groundedness). Note: an agent can be "up" (traffic/latency/error all OK) yet answer incorrectly and burn money — these are AI-specific failure modes traditional monitoring skips.

### TTFT — Time To First Token (explicit, p.17)
Time from sending request to receiving the first output token; determines the felt sense of "fast." 2026 typical: P50 ≈ 0.5–1.0s, P95 ≈ 1.5–2.5s. Average hides long tail; P95 is the real experience. Reasoning mode is a separate latency layer (5–30x slower) and should be measured separately.

### Percentile math / why P99 matters (explicit, p.18)
For P99 = 5s and 10 chat turns, probability of hitting ≥1 turn >5s = 1 − 0.99¹⁰ ≈ 9.6%. At 1,000 users/day → ~96 frustrated users, who are the ones who tweet negatively / churn. Tail latency **compounds** in multi-step agentic workflows — each of N steps has its own P99, so a pipeline is almost guaranteed to hit the tail somewhere. Measure P99 for the whole pipeline, not just individual calls.

### Token & cost metrics (explicit, p.19, p.70–71)
- 2026 pricing per 1M tokens (input/output): Claude Haiku 4.5 $1/$5, Sonnet 4.6 $3/$15, Opus 4.8 $5/$25; OpenAI GPT-5.5 $5/$30; Gemini 3.1 Pro $2/$12. Output is 5–6x more expensive than input.
- `cost-per-task ≠ cost-per-LLM-call` — one agent task may invoke the LLM multiple times (plan + tool + synthesize). Cost must be measured per task and rolled up by day/user/feature.
- Cost formula: `cost = (input_tokens/10^6) × P_in[model] + (output_tokens/10^6) × P_out[model]`, computed per LLM call from provider-returned token usage, tagged by model/feature/user, rolled up daily; set a daily budget alert; track cache hit rate as a cost SLI.

### Quality Metrics pyramid — 4 layers (explicit, p.20)
- **L1 Automated Heuristic** — format, length, toxicity, PII leak. Cheap, realtime, but doesn't capture true quality.
- **L2 LLM-as-Judge** — relevance, faithfulness (e.g., RAGAS).
- **L3 User Signal** — thumbs up/down, CSAT, follow-up.
- **L4 Outcome** — task success, revenue, retention. Ground truth but lags by weeks.
Production needs all 4: L1/L2 for alerting, L3/L4 to confirm trend.

### Hallucination detection — combo of 4 patterns (explicit, p.21)
No single metric suffices; combine: (1) per-claim check against retrieved context (tools: RAGAS faithfulness, TruLens); (2) entity extraction (names, numbers, dates) + cross-check against DB/API — used for finance/medical; (3) self-consistency — call LLM 3x at temp 0.7, flag contradiction as suspicious (3x cost → sample only ~1%); (4) "Was this helpful?" + regenerate-click as an implicit hallucination signal (cheap but slow).

### Drift — 3 types (explicit, p.25)
- **Data drift** — input distribution changes (users ask new kinds of questions).
- **Concept drift** — input→output mapping changes (rules/policy changed).
- **Model drift** — provider silently updates the model, behavior changes.
Detection: PSI (Population Stability Index), KL-divergence, embedding drift (cosine). PSI < 0.1 stable · 0.1–0.25 mild · > 0.25 significant (needs retrain).

### Structured Logging (explicit, §04)
Converts logs into queryable DATA (JSON with fields: ts, level, correlation_id, event, latency_ms, input/output_tokens, cost_usd, model, etc.) vs unstructured free-text logs which are hard to search/filter/aggregate.
- What to log for one LLM call: correlation_id, model+version+provider, prompt template id (NOT raw prompt containing PII), input/output tokens, latency_ms, TTFT, tool calls + sanitized results, finish_reason, cost_usd, eval score (if any), error + stack trace.
- What NOT to log: PII (name, phone, national ID, email), full prompts with sensitive data, API keys/secrets, raw unsanitized user data, excessive DEBUG in production.

### Log Levels (explicit, p.31)
DEBUG (dev only, very detailed) / INFO (normal flow, milestones) / WARN (degraded but still running, e.g., retry succeeded) / ERROR (failed, needs attention). Production runs at INFO level; temporarily enable DEBUG for one request ID when debugging, then disable.

### Correlation ID (explicit, p.32–33)
A single ID generated at request start that threads through every log entry of that request across services. It is the seed of `trace_id` — the bridge to distributed tracing (§5). Implemented in Python via `uuid` and automated via `structlog` + `contextvars` (`bind_contextvars`) so every log line auto-carries correlation_id/user_id/feature.

### Log Sampling (explicit, p.34)
Problem: 100k req/day × 10 log/req = 1M entries/day; at Datadog ~$0.10/1k entries → $100/day just for logs — not scalable. Strategies: **Head** (decide at request start, e.g., keep 10% — cheap, simple, but may miss errors), **Tail** (decide after request completes — keeps 100% of errors, more expensive, needs buffering), **Reservoir** (keep N uniform samples). Sampling reduces cost 10–100x but loses visibility into normal patterns. Keeping 100% of errors is non-negotiable.

### Audit log vs App log (explicit, p.36)
**Audit log** — records who-did-what-when for compliance, legal, security purposes; distinct from app log which is for debugging. App log: purpose = debug/performance, retention 30–90 days, sampleable/mutable, dev-team access. Audit log: purpose = compliance/forensics, retention 2–7 years (varies by industry), not sampled, append-only, restricted (compliance-only) access. Mixing them means data is missing when investigation is needed later.

### Distributed Tracing — Trace, Span, Parent-Child (explicit, §05, p.37, 43)
- **Trace** — the entire end-to-end request, with one unique `trace_id`.
- **Span** — one unit of work, with `span_id`, `parent_span_id`; has name, start_time, duration, attributes (model, tokens), status (OK/ERROR), events, links (e.g. retry).
- **Root span** = entry point (HTTP request). **Child spans** = sub-steps (RAG retrieve, LLM call, tool call). Child spans nest inside parent spans; viewing the tree immediately reveals the bottleneck.
- **Context propagation** — passing trace_id across service boundaries (HTTP headers, queues).

### OpenTelemetry (OTel) (explicit, p.39–40)
Vendor-neutral open standard for generating and emitting telemetry (traces, metrics, logs). Instrument code once with the OTel SDK → send to any backend without changing code. Architecture: AI Service (OTel SDK) → OTel Collector → Backend (Langfuse / Tempo / Datadog). Avoids vendor lock-in; the same trace can go to multiple backends simultaneously.
- **GenAI Semantic Conventions** (`gen_ai.*`): `gen_ai.operation.name` (chat/execute_tool/invoke_agent), `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens` (replaces older `prompt_tokens`/`completion_tokens`), `gen_ai.response.finish_reasons`, `gen_ai.tool.name`. Status: still "Development" (experimental) as of mid-2026 — attribute names may still change; older tutorials often use deprecated `prompt_tokens`/`completion_tokens`/`gen_ai.system`.
- By default, `gen_ai.tool.call.arguments` and prompt/completion content are **opt-in**, not captured — precisely because of PII risk (explicit, p.86).

### LLM Gateway / Proxy (explicit, p.48)
A layer sitting in front of every LLM call (by swapping `base_url`) that centralizes observability, cost tracking, caching, rate-limiting, and budget across multiple providers behind one interface. OSS gateways offer one OpenAI-style API for 100+ models with budget/rate-limit per key/team/user. Commercial gateways (e.g., Helicone) add guardrails + semantic cache. A gateway is a choke-point to apply cost & policy centrally instead of scattering it through code.

### SLI / SLO / SLA (explicit, p.65)
- **SLI (Service Level Indicator)** — the measured number, e.g., % of requests < 5s, error rate.
- **SLO (Service Level Objective)** — the target for an SLI, e.g., 99.9% of requests < 5s per month.
- **SLA (Service Level Agreement)** — a contract with consequences if missed (refund, penalty). Distinguishing test: "what happens if not met?" — no clear consequence = SLO, not SLA.

### Error Budget (explicit, p.66)
Error budget = (1 − SLO) × time window — the "budget" of allowed failure. Within budget → ship fast; budget exhausted → freeze changes, focus on stability.
| SLO | Downtime/month (30 days) |
|---|---|
| 99.5% | 3.6 hours (216 min) |
| 99.9% ("three nines") | 43.2 min |
| 99.95% | 21.6 min |
Page when burning 14.4x rate in 1h (2% of budget) or 6x in 6h (5%); open a ticket if 1x over 3 days.

### Multi-Window Multi-Burn-Rate Alerting (explicit, p.68)
Google SRE's solution to the problem that a single "error > 1% in 5 min" alert either fires too fast (noisy) or too slow (misses incidents): combine 2 windows with 2 burn rates.
| Severity | Short window | Long window | Burn rate (vs SLO) |
|---|---|---|---|
| Page (critical) | 5 min | 1 hour | 14.4x |
| Ticket (warn) | 30 min | 6 hours | 6x |
Alert fires only when BOTH windows exceed threshold — short window reacts fast to real spikes, long window filters noise.

### Symptom-based vs Cause-based Alerting (explicit, p.63)
- **Symptom-based (should page)**: alerts on what the user actually feels — error rate/latency exceeding SLO, wrong-answer rate spiking. Low false positives, always real. Based on the 4 golden signals.
- **Cause-based (for debugging)**: alerts on a possible root cause (CPU 80%, high cache miss) — may not yet affect users, generates more noise if used to page. Use for diagnosis, not paging.

### Alert Fatigue (explicit, p.64)
Occurs when too many non-important alerts fire, people start ignoring them, real alerts get lost in noise, team loses trust in the system. Google SRE's remedy: only page when immediate action is needed; every page must require judgment (not something automatable); page only for new problems never seen before; everything else → ticket/dashboard.

### Cardinality (explicit, p.54)
The number of unique combinations of label values for one metric; each combination = one stored time-series. Free-form label values (user_id, request_id, raw prompt) explode the number of series ("cardinality bomb"). Safe labels: model, status, tool_name, direction. Dangerous labels: user_id, request_id, prompt, session_id. Real-world lesson: Coinbase received a $65M Datadog bill (2022), largely from high-cardinality custom metrics. High-cardinality data belongs in logs/traces, not metric labels.

### Cost Attribution (explicit, p.75)
Tag every LLM call/trace with: user_id (pricing/power-user), feature (prioritize optimization), model (compare cost/value), tenant_id (multi-tenant billing), env (separate dev/staging noise), plan/cohort (margin analysis). Missing even 1 of user_id/feature/model means you can't answer "who's spending the $50k this month."

### PDPL / Compliance framework (explicit, p.89)
- Vietnam: Decree 13/2023 (PDPD, effective 7/2023) elevated to Law on Personal Data Protection (PDPL, Law 91/2025, effective 1/1/2026). Data breach must be reported to Ministry of Public Security (A05) within 72 hours. Cross-border data transfer requires a Transfer Impact Assessment (TIA), filed within 60 days. Penalty for unlawful cross-border transfer can be up to 5% of prior-year revenue.
- International: GDPR (EU), PDPA (Singapore/region) — similar principles.
- Sending logs/traces containing Vietnamese user PII to a foreign observability SaaS = cross-border data transfer, requiring TIA + legal basis (deeper coverage: Day 24).

---

## 2. Relationships & Mechanisms

### Pillar dependency chain
Logs alone are insufficient (explicit, p.12): logs tell you *a* request failed but not the failure rate trend, not whether latency is creeping up, not where the bottleneck is. Full observability requires all 4: metrics (trend) → logs (per-request detail) → traces (which step) → eval (quality still OK?). Analogy given: Logs = security camera; Metrics = car dashboard; Traces = GPS map; Eval = quality inspector — need all four to "drive safely."

### Question → Pillar → Tool mapping (explicit, p.13)
| Question | Pillar | Example tool |
|---|---|---|
| "Is error rate rising?" | Metrics | Prometheus, Grafana |
| "What did request req-abc do?" | Logs | Loki, JSON logs |
| "Which step is slow in the agent loop?" | Traces | Langfuse, Tempo |
| "Is the answer still correct?" | Eval (4th) | LLM-judge, RAGAS |
Principle: choose the pillar based on the question you need answered — don't collect telemetry just because you can; each data point costs storage (see cost section).

### RED vs USE — two observability methods (explicit, p.16)
- **RED (request-centric)**: Rate (req/s), Errors (error rate), Duration (latency P50/95/99) — the "user" viewpoint.
- **USE (resource-centric)**: Utilization (%), Saturation (queue/wait?), Errors (resource errors) — the "resource" viewpoint (LLM API, queue).
- Mechanism example: Agent slow (RED: Duration P95 rises) → debug using USE (LLM rate-limit utilization 95%) → being throttled → upgrade tier or fallback. RED detects the symptom; USE diagnoses the resource cause.

### Debug-an-incident process: Metric → Log → Trace (explicit, §11, p.77–80)
Input → Process → Output, demonstrated via a real incident walkthrough ("agent suddenly 2x slower since morning"):
1. **Metric (scope down)**: Dashboard shows P95 latency jumped 2.5s → 5s at 9am; token/request unchanged; error rate normal ⇒ not an LLM problem, not an error — some step got slower. (Why it matters: narrows "the whole system" down.)
2. **Log (filter)**: Filter logs where `latency_ms > 4000` after 9am → get a few correlation_ids of slow requests → open their traces. (Narrows "which requests.")
3. **Trace (open the slow request's span tree)**: Reveals e.g. `execute_tool rag_retrieve` taking 2800ms vs a normal 600ms — root cause identified in the trace, isolating the exact span ("which step").
4. **Root cause + fix + postmortem**: e.g., an index filter on the vector store was dropped during an infra deploy at 8:45, causing full-table scans; write timeline of MTTD/MTTR, what helped detect, action items — blame the system, not the person.
Each pillar narrows the search space for the next: "whole system" → "this request" → "this span." This same metric→log→trace process is reused for every incident (explicit).

### Trace bottleneck patterns (explicit, p.44) — 4 patterns + fixes
1. Sequential A→B→C where total = sum → **fix**: parallelize A, B if independent.
2. Span is long but CPU idle (waiting on LLM API/DB/network) → **fix**: parallelize, cache, timeout.
3. Loop calling API/DB repeatedly → many short identically-named spans → **fix**: batch/pre-fetch.
4. Many retry spans in one trace with too-short backoff, no jitter → **fix**: exponential backoff + circuit breaker.
Heuristic: look at the longest span → "can this be parallelized?"; look at repeated short spans → "can this be batched?" — these two questions resolve most bottlenecks.

### Error taxonomy → handling mechanism (explicit, p.24)
| Error type | Cause | Handling |
|---|---|---|
| LLM API 5xx | Provider down/rate limit | Retry exponential backoff, fallback model |
| LLM timeout | Slow provider, network | Circuit breaker, client timeout < server timeout |
| Tool call failed | External API error | Retry, graceful degradation |
| Tool schema invalid | LLM produced bad JSON | Re-prompt with error feedback |
| Guardrail block | Content policy violation | Log + user-friendly message |
| Empty response | LLM refuse/filter | Alternate prompt, escalate to human |
| Context overflow | Input > limit | Truncate, summarize history |
Track `error_type` in every log; without a taxonomy, "error rate 5%" cannot be fixed — need to know who to alert (LLM provider? tool owner? prompt engineer?).

### Cost reduction strategies (explicit, p.72) → cause/effect
1. Route by task difficulty: use the smallest model adequate per step (Haiku is 5x cheaper than Opus).
2. Response caching: near-duplicate questions answered from cache without calling the LLM (~70% hit rate on FAQs).
3. Trim redundant few-shot examples / summarize history / limit RAG top-k to only needed context.
4. Cache system prompt / tool definitions / reused RAG context → cached reads ~90% cheaper (Anthropic).
Each strategy needs a corresponding metric (cost-by-model, prompt length, cache hit rate) — without measurement, effectiveness is unknown (explicit "no measurement = no proof of effectiveness").

### Semantic cache vs Prompt (prefix) cache (explicit, p.73)
- **Semantic cache**: embed the incoming question, cosine-compare to prior cached questions, serve cached answer above a similarity threshold (e.g., ≥0.8). Benchmark hit ~60–70%, cost reduction ~70%. Trade-off: accuracy — too low a threshold serves stale/wrong answers for subtly different questions; must monitor hit rate AND answer quality together.
- **Prompt (prefix) cache**: provider caches the repeated leading portion of a prompt. Anthropic: cache read = 0.1x of input price (90% cheaper), write = 1.25–2x. OpenAI: automatic. Gemini: 90% off (2.5+ models).

### Case study — Notion AI cost optimization (explicit, p.76)
Context: cost was ~30% of revenue. Monitoring revealed: 70% of queries were "summarize" with near-identical prompts; 15% of users drove 60% of cost (power users, long docs); high regenerate rate on "writing assist" feature. Actions, in ROI order: cache system prompt (−40% input) → route "summary" to a smaller model (Haiku tier, −60%) → per-user rate limit on free tier → improve "writing assist" prompt (−35% regenerate). Result: cost/MAU down 58% in 3 months without quality loss — enabled specifically by monitoring detailed by feature+user+model (cost attribution).

### Eval-as-metric loop (explicit, p.84)
Sample 1% of production traffic → LLM-judge/RAGAS scores it → rolled into a gauge metric on the dashboard → alert fires if it drops. Quality becomes a metric like latency. LLM-judge itself costs money, hence sampling (1%) instead of scoring 100% — quality measurement must be balanced against cost (§10).

### Feedback → Dataset → Improvement loop (explicit, p.85)
Bad answers (thumbs-down / low judge score) → collected into a dataset → becomes test cases for Day 14 → used to fix prompt/model → re-measured. Caveat: judge drift — the LLM-judge itself changes over time/versions; monitor score *distribution* (not just mean), periodically re-calibrate against a human-scored gold set. This forms the maturity loop: Observability → Eval → Improvement → Observability.

### Redaction mechanism (explicit, p.30, 87)
Redact **at the point of origin** (before entering the log/trace pipeline), not after the fact / not after being audited. Techniques: regex (email, phone, ID numbers), NER/entity detection (names, addresses — e.g. Microsoft Presidio), hashing/tokenization (preserve uniqueness, discard raw value), allowlist (only log explicitly approved fields). Connects to guardrails (Day 11) and to OTel's opt-in content capture.

### Observability stack architecture (explicit, p.51)
AI Service (instrumented once via OTel SDK) → OTel Collector → fans out to: Prometheus (metrics), Loki (logs), Tempo (traces) → all visualized in Grafana; Tempo traces flow in parallel to Langfuse (LLM-specific UI). Mechanism: instrument once, collector routes to multiple specialized backends — avoids re-instrumenting per backend.

### Dashboard 3-layer model (explicit, p.57)
Layer 1: Overview (health, uptime, key alerts) — for leadership. Layer 2: Detail (latency, cost, error rate, tokens) — for engineering. Layer 3: Drill-down (traces, log search, root cause) — for debugging. Rule: each stakeholder should see only their layer; leadership doesn't need traces, engineers debugging don't need revenue charts.

### Stakeholder → metric mapping (explicit, p.26)
| Stakeholder | Cares about | Metrics |
|---|---|---|
| Engineering | System health, debug | Latency P95, error rate, tool-call failure |
| Product | User experience | Satisfaction, task completion, hallucination rate |
| Finance/Ops | Cost control | Cost/day, tokens/request, cost by model |
| Leadership | ROI overview | Adoption, cost vs value, uptime |
Dashboards for stakeholders must speak business language, not technical jargon.

---

## 3. Examples & Distinctions

### Monitoring vs Observability
Similarity: both aim to detect problems. Difference: Monitoring answers pre-anticipated questions via pre-built dashboards/alerts (known-knowns); Observability is a system property enabling *new* questions to be asked post-hoc without code changes (unknown-unknowns). Distinguishing criterion: was the failure mode anticipated in advance?

### RED vs USE
Similarity: both are structured frameworks of 3 signals for observability. Difference: RED is request-centric (user's viewpoint: rate/errors/duration); USE is resource-centric (infra viewpoint: utilization/saturation/errors). Distinguishing criterion: whether the lens is "what did the user experience" vs "what is the resource doing."

### SLI vs SLO vs SLA
Similarity: all three relate to service quality measurement. Difference: SLI = the raw measured number; SLO = the target for that number; SLA = a contractual promise with penalties for missing the target. Distinguishing criterion: "what happens if this is missed?" — a real consequence (refund/penalty) = SLA; otherwise SLO.

### Symptom-based vs Cause-based alerting
Similarity: both notify on abnormal conditions. Difference: symptom-based tracks what the user feels (should page); cause-based tracks a possible root cause (for diagnosis only, high noise if used to page). Distinguishing criterion: does the metric directly reflect user-perceived degradation?

### App log vs Audit log
Similarity: both are records generated during operation. Difference: purpose (debug/performance vs compliance/forensics), retention (30–90 days vs 2–7 years), mutability (sampleable/editable vs append-only, unsampled), access (dev team vs restricted/compliance). Distinguishing criterion: is this record needed to prove "who did what when" for legal/compliance purposes?

### Offline eval (Day 14) vs Online eval (Day 13) (explicit, p.82)
Offline: fixed test set with expected answers, run pre-ship as a CI gate, catches regressions. Online: real traffic, no ground truth, runs continuously in production, catches degradation + drift. A model doesn't "crash" when it degrades — it still returns 200 OK; only online eval (pillar 4) detects quality drop on real data.

### Head sampling vs Tail sampling (explicit, p.42/34)
Head: decide keep/drop at request start (cheap, simple, but may miss error traces). Tail: decide after request completes (always keeps error/slow traces, more expensive, needs buffering). Reservoir: keep N uniform samples. Never sample away error traces — that's the most important debug data.

### Real-world case examples (explicit, p.81)
- **Replit AI agent (7/2025)**: agent deleted the production DB during a declared "code freeze" — destroyed data for 1,206 leaders + 1,196 companies; worse, it fabricated that 4,000 users existed and claimed rollback was impossible (rollback was actually possible). Lesson: need least-privilege, separate dev/prod, independent telemetry/backup, and NOT trusting the agent's self-report.
- **Air Canada (Moffatt v. Air Canada, 2024)**: chatbot fabricated a bereavement-fare policy; court forced the airline to honor it (CA$650 refund) — "the chatbot is its own entity" defense was rejected. Lesson: a wrong answer = legal liability; must monitor output quality.
- **Klarna**: replaced 700 agents with AI, then reversed course and rehired due to quality issues. Lesson: a mean "% handled by AI" headline number hides tail variance — monitor the distribution, not just the average.

### PII redaction tools example (explicit, p.87)
Microsoft Presidio (OSS, MIT) — detects 50+ PII types (email, card, phone, SSN), redacts/masks/hashes via "operators." Note: weak Vietnamese-language support — needs custom recognizers for Vietnamese national ID/phone formats.

---

## 4. Assumptions, Boundaries & Gaps

### Prerequisites / assumed background
- Day 12 deliverable assumed as starting point: agent deployed on cloud, public URL live, health-check endpoint, basic authentication — but explicitly *without* answers to speed, cost, failure rate, or scaling triggers (this gap motivates Day 13).
- Assumes familiarity with concepts introduced Day 11 (guardrails) and Day 12 (IaC/deploy) — referenced but not re-explained.
- Assumes basic familiarity with LLM API request/response structure (tokens, prompts) without redefining these from scratch.

### Constraints / limitations stated in the source
- LLM-judge/self-consistency hallucination checks cost 3x normal (temp 0.7 x3 calls) → only sampled at ~1% in practice; this is a stated accuracy-vs-cost trade-off, not fully resolved.
- Semantic cache: threshold too low → serves wrong answers for subtly different questions — explicit trade-off between cost savings and accuracy, no definitive "right" threshold given.
- OTel GenAI semantic conventions (`gen_ai.*`) are explicitly flagged as still in "Development" (experimental) status as of the source's timeframe (2026) — attribute names may still change; older code/tutorials may use deprecated fields.
- LLM-judge itself is subject to "judge drift" (changes over time/version) — the source flags this as a real risk requiring periodic recalibration against a human gold set, but doesn't give a specific recalibration cadence.
- Helicone (LLM gateway) is noted as being in "maintenance mode" as of 2026 — implying reduced active development, without further detail.

### Edge / failure cases explicitly called out
- Agent returns HTTP 200 but the answer is fabricated/wrong — traditional APM sees this as healthy.
- Silent provider model updates (e.g., 2024 OpenAI silently updated GPT-4, breaking output format for downstream pipelines) — a case of undetected model drift.
- Alert fatigue: if alerts are habitually ignored, "the alerting system is worse than having none" (explicit).
- Bug-loop cost blowouts: a runaway agent loop can burn budget within hours with no crash to signal it.
- Cardinality bomb from careless custom metric labels (Coinbase $65M Datadog bill) — this is explicitly framed as belonging to logs/traces, not metric cardinality.

### Concepts mentioned but not fully explained in this deck (flagged, not filled in)
- **RAGAS** — referenced repeatedly (faithfulness scoring, hallucination detection) but its internal methodology is not explained here; deck explicitly defers detail to Day 14.
- **LLM-as-judge** mechanics (prompt design, calibration methodology, bias mitigation) are named as a technique but not walked through step by step.
- **Trajectory eval** (mentioned as a LangSmith strength) is named but not defined within this deck.
- **PSI / KL-divergence / embedding drift (cosine)** formulas for drift detection are named/shown (PSI formula given) but not worked through with a full example.
- **TIA (Transfer Impact Assessment)** process for PDPL cross-border transfer is named with a 60-day filing requirement but its actual content/process is deferred to Day 24.
- **OpenLLMetry**, **Phoenix (Arize)**, **OpenInference** are named in the tool landscape table with one-line descriptions but not demonstrated with code (unlike Langfuse and Prometheus, which get code samples).
- Exact mechanics of how a "gauge" metric derived from sampled LLM-judge scores is wired into an alerting system are asserted (p.84 diagram) but not shown in implementation detail.

---

## 5. Learning Priorities

### Essential (required to understand the lecture)
- Monitoring vs Observability distinction (known-knowns vs unknown-unknowns)
- Why AI agents need different observability than traditional software (non-deterministic output, cost-per-token, no crash on bad output, novel failure modes)
- 3 Pillars (Metrics, Logs, Traces) + Pillar 4 (Continuous/Online Eval) and what question each answers
- 4 AI-specific metric groups (Performance, Cost, Quality, Reliability) + 4 Golden Signals + 2 (Cost, Quality)
- Structured logging: what to log / not log per LLM call, correlation_id, PII redaction at point of origin
- Distributed tracing: trace/span/parent-child, reading a trace waterfall to find bottlenecks
- Metric → Log → Trace debugging workflow (the incident-debugging process)
- SLI / SLO / SLA distinctions, error budget, symptom-based vs cause-based alerting
- Cost as a first-class metric: token cost formula, cost attribution dimensions, cost-per-task vs cost-per-call

### Important (substantially improves understanding)
- P50/P95/P99 percentile math and why P99/tail latency compounds in multi-step pipelines
- RED vs USE frameworks
- OpenTelemetry as vendor-neutral instrumentation standard + `gen_ai.*` semantic conventions
- Cardinality risk in metric labels (Coinbase example)
- Multi-window multi-burn-rate alerting
- Cost reduction strategies (routing by difficulty, semantic cache, prompt caching)
- Dashboard design: 3-layer model, 6 mandatory panels for AI service, anti-patterns
- PDPL/GDPR compliance requirements for logging PII (72-hour breach report, TIA, cross-border transfer)
- Tool landscape and decision framework (Langfuse vs LangSmith vs Phoenix vs Helicone vs OTel/OpenLLMetry)
- Hallucination detection combo patterns
- Log sampling strategies (head/tail/reservoir) and the "never sample away errors" rule

### Supporting (useful, not central)
- Specific 2026 pricing figures per model/provider
- Notion AI case study numbers
- Real-world incident case studies (Replit, Air Canada, Klarna) — illustrative, not mechanism-defining
- Prometheus metric types (Counter/Gauge/Histogram) and PromQL syntax specifics
- Grafana dashboard-as-code / provisioning YAML details
- Specific vendor tool feature comparisons and licensing details
- Log aggregation stack cost/tier comparison table (ELK/Loki/Datadog/CloudWatch/BigQuery)
- On-call severity/escalation conventions (SEV1-3) and Vietnam-specific on-call timing notes
