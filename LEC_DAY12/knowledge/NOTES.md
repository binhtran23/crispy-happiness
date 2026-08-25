# LEC_DAY12 — Deployment: Đưa Agent Lên Cloud

## Keypoints
- Agent deployment differs from traditional web apps due to 3 properties: long-running (10–60s+), stateful (conversation memory), costly (superlinear token cost)
- Externalize state (Redis/Postgres/checkpointer) resolves 12-factor stateless vs agent memory tension — also enables horizontal scaling, MCP stateless protocol, scale-to-zero
- Sticky session is best-effort only — Cloud Run kills instances anytime, losing in-memory state
- Containerization: python:3.12-slim is default for ML Python; avoid Alpine (musl libc breaks wheels); multi-stage + uv; target <500MB; non-root; pin digest; scan CVE
- SSE streaming is the default for agent responses (not the exception) — solves timeout + UX
- Async job pattern (submit-and-poll) for tasks exceeding timeout limits
- 4-tier deployment: Tier 0 (managed runtime) → Tier 1 (Railway/Render) → Tier 2 (Cloud Run/ECS) → Tier 3 (Kubernetes); most important axis is timeout, not price
- Serverless functions poorly suited for agents — hard timeout cap is the real issue, not cold start
- API Gateway: Auth → Rate Limit → Budget Check → Agent; "reject early = save tokens and money"
- Provider spend alerts only notify, never block — must build own pre-call admission control
- CI/CD eval gate: score against golden set with LLM-judge; compare against baseline branch, not just absolute threshold
- Don't pin temperature=0 — Anthropic deprecating temperature/top_p/top_k from Opus 4.7
- "Just retry" is dangerous for agents (side effects) — use idempotency keys
- LLM-specific trap: dedup by semantic intent (e.g. order_id), not byte hash — LLM won't regenerate identical parameters
- Shadow deploy doubles token bill — sample 5–10%, block side effects; Canary = real output to bounded % of users
- Rollback = 3 independent artifacts (image/prompt/model) with different lifecycles; prompt in .py file kills fast rollback
- Lethal Trifecta: private data + untrusted content + outbound channel — remove any one to block leak; egress allowlist is strongest defense
- Fail-closed not fail-open — CVE-2025-66479 (Claude Code) showed allowedDomains:[] failed open

## Terms
- Externalize state (ngoại hóa trạng thái) — move state out of process to Redis/Postgres so any instance can serve any request
- SSE (Server-Sent Events) — one-directional server→client streaming over HTTP/1.1, auto-reconnect via Last-Event-ID
- Eval gate (cổng đánh giá) — AI-specific CI/CD step: deploy only if agent scores above threshold on a golden set
- Idempotency key (khóa bất biến) — client-supplied UUID ensuring server deduplicates retries, returning cached result instead of repeating side effect
- Lethal Trifecta (bộ ba chết người) — private data + untrusted content + outbound channel; all three needed for data leak
- Shadow deployment — mirror traffic to new version, always discard output; zero user risk but doubles token cost
- Canary deployment — route real % of traffic to new version with real output; bounded risk
- Admission control (kiểm soát đầu vào) — pre-call check: estimate cost, reject if exceeds budget before calling LLM
- SBOM (Software Bill of Materials) — list of all packages in image; legally required by US EO 14028, EU CRA

## Covered / To revisit
- [x] Three properties (long-running, stateful, costly)
- [x] 12-factor vs agent memory — externalize state
- [x] Containerization (base image, multi-stage, security)
- [x] Timeout problem, SSE streaming, async job
- [x] 4-tier deployment model
- [ ] API Gateway flow — student initially missed budget check step
- [x] Cost protection / admission control
- [x] CI/CD with eval gate
- [ ] Eval gate threshold — student confused quality drift with overfit
- [x] Idempotency & safe retry
- [x] Shadow vs Canary deployment
- [x] Rollback as 3 independent artifacts
- [ ] Lethal Trifecta application — student tried to redact data instead of blocking outbound channel when both data and untrusted content are required by the use case
- [ ] MCP transports & OAuth 2.1 traps (not covered)
- [ ] Durable execution vs LangGraph checkpointing (not covered)
- [ ] Saga pattern & irreversible actions (not covered)
- [ ] Concurrency & cold start details (not covered)
- [ ] GPU serving techniques (not covered)
- [ ] Agent identity & SPIFFE (not covered)
- [ ] Tier 0 billing & session management (not covered)

## Misconceptions / Examiner findings
- API Gateway flow — omitted budget check; understood rate limit but not cost-based admission control as a separate concern. Gap: didn't distinguish "number of requests" limiting from "dollar amount" limiting
- Quality drift vs overfit — when asked what happens with a fixed eval threshold over time, answered "overfit." Actual issue is quality drift (score drops incrementally but never crosses threshold). Gap: conflating ML training concepts with deployment pipeline concepts
- Lethal Trifecta application — when asked which element to remove given fixed requirements for private data + untrusted content, answered "redact private data" instead of "block outbound channel." Gap: didn't fully internalize that the trifecta is about which element is *removable given constraints*, not which is most important
