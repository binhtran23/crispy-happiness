# LEC_DAY12 — Quiz

## Q1 [priority: Essential]
**Question:** An agent's reasoning loop takes 45 seconds. AWS API Gateway has a 29-second timeout. What is the standard solution according to the lecture?
- A) Increase the API Gateway timeout to 120 seconds
- B) Use SSE streaming so tokens are sent incrementally, resetting the idle timer
- C) Switch to a faster model that responds in under 29 seconds
- D) Use WebSocket instead of HTTP

**Answer:** B
**Explanation:** SSE streaming is the de facto standard for LLM output — tokens sent one-by-one reset the proxy's idle timer, preventing 504s. Option A only delays the problem and wastes resources (connection held open with no data). Option C changes the model, not the infrastructure. Option D is unnecessary — SSE runs on plain HTTP/1.1 and is one-directional (server→client), which is exactly what token streaming needs.

## Q2 [priority: Essential] [weak-area]
**Question:** What is the correct order of checks in an API Gateway before a request reaches the agent?
- A) Rate Limiter → Auth Check → Agent
- B) Auth Check → Rate Limiter → Input Validation + Budget Check → Agent
- C) Auth Check → Agent → Budget Check
- D) Rate Limiter → Budget Check → Auth Check → Agent

**Answer:** B
**Explanation:** Auth first (reject unauthenticated requests immediately), then rate limit (reject excessive requests), then input validation + budget check (reject requests that would exceed cost limits) — only then does the request reach the agent. The principle is "reject early = save tokens and money." Option A skips budget check entirely. Option C checks budget after the agent has already run (too late). Option D checks auth last, wasting rate-limit and budget-check work on unauthenticated requests.

## Q3 [priority: Essential]
**Question:** The 12-factor app demands stateless processes, but agents carry conversation memory. How does the lecture resolve this?
- A) Abandon the 12-factor principle for agents
- B) Use sticky sessions to pin users to specific instances
- C) Externalize state to Redis/Postgres/checkpointer so any instance can serve any request
- D) Store state in the container's local filesystem

**Answer:** C
**Explanation:** Externalize state resolves the tension without abandoning the principle — state exists, but outside the process. This enables horizontal scaling, MCP stateless protocol, and scale-to-zero. Option A gives up scalability benefits. Option B is explicitly called "best-effort" — Cloud Run breaks sticky sessions whenever an instance is killed. Option D is lost when the container restarts.

## Q4 [priority: Essential]
**Question:** Which base image does the lecture recommend as the default for ML Python agents?
- A) `python:3.12` (full, ~1GB)
- B) `python:3.12-alpine` (~55MB)
- C) `python:3.12-slim` (~150MB)
- D) `distroless` (~66MB)

**Answer:** C
**Explanation:** `python:3.12-slim` uses glibc, so prebuilt manylinux wheels for numpy/pandas/matplotlib install directly without compilation. Alpine uses musl libc — many ML packages lack musllinux wheels (matplotlib still missing aarch64) and must compile from source. Full image has unnecessary attack surface. Distroless is smallest/safest but has no shell for debugging — suitable for hardened production, not the default.

## Q5 [priority: Essential] [weak-area]
**Question:** An eval gate uses a fixed threshold of 80%. Over 4 sprints, scores drop from 92% → 88% → 84% → 81%. All deploys pass. What is this problem called and how should it be fixed?
- A) Overfitting — retrain the eval model
- B) Quality drift — compare against the baseline branch run, not just an absolute threshold
- C) Regression testing failure — add more unit tests
- D) Model degradation — switch to a newer model

**Answer:** B
**Explanation:** Quality drift: each sprint loses ~4 points but never crosses the threshold, so no deploy is blocked. Over time, quality degrades significantly (92→81 = 11 points lost) without detection. The fix is comparing each run against the current production baseline, catching any significant drop immediately. This is not overfitting (an ML training concept), not a unit test issue (agents are non-deterministic), and not model degradation (the model didn't change).

## Q6 [priority: Essential]
**Question:** Agent A calls Agent B to charge a credit card. B succeeds but replies slowly. A times out and retries. What happens, and what is the correct prevention?
- A) Nothing — the retry is safe because HTTP is idempotent
- B) The card is charged twice; prevent by using an idempotency key so B returns the cached result on retry
- C) The card is charged twice; prevent by increasing A's timeout
- D) B rejects the retry automatically

**Answer:** B
**Explanation:** A timeout cannot distinguish "actually failed" from "succeeded but replied slowly," so A blindly re-fires the side effect. The fix is an idempotency key: A sends a UUID with the request; B stores the result keyed by that UUID; on retry with the same key, B returns the stored result without re-executing. Option A is wrong — POST with side effects is not idempotent by default. Option C only delays the problem. Option D requires B to have dedup logic, which is exactly what an idempotency key provides.

## Q7 [priority: Essential]
**Question:** For LLM agents specifically, why can't you simply hash the request body to generate an idempotency key?
- A) Hash functions are too slow for real-time use
- B) The LLM generates slightly different parameters for the same intent, so hash values differ
- C) Request bodies are too large to hash
- D) Hashing is not supported by most frameworks

**Answer:** B
**Explanation:** An LLM won't regenerate byte-identical parameters for the same intent (e.g., adding "please" to a charge request). Different bytes → different hash → server treats it as a new request → side effect fires again. The solution is dedup by semantic intent (e.g., using `order_id`) rather than content hash.

## Q8 [priority: Essential] [weak-area]
**Question:** According to the Lethal Trifecta, data leakage requires three elements simultaneously: private data, untrusted content, and an outbound channel. If your agent MUST process private data AND read untrusted internet content, which element should you remove?
- A) Redact all private data before processing
- B) Block the outbound channel via egress allowlist
- C) Stop reading untrusted content
- D) Encrypt all data in transit

**Answer:** B
**Explanation:** The premise fixes elements 1 and 2 as business requirements. The only removable element is #3 — the outbound channel. Block it via egress allowlist (only allow calls to approved domains), block rendering to external URLs, and use container network policies. Option A contradicts the requirement to process private data. Option C contradicts the requirement to read internet content. Option D protects data in transit but doesn't prevent the agent from intentionally sending data to an attacker's URL.

## Q9 [priority: Important]
**Question:** Shadow deployment mirrors traffic to a new agent version but discards its output. Why is this particularly expensive for agents compared to traditional web apps?
- A) Agents require more CPU
- B) Both versions call the LLM on every mirrored request, doubling the token bill
- C) Shadow deployment requires twice the server instances
- D) Network bandwidth doubles

**Answer:** B
**Explanation:** Each agent request costs real LLM tokens. Shadow runs both versions, so token cost doubles. For a traditional web app, the extra CPU cost of shadowing is negligible. The lecture recommends shadowing only 5–10% of traffic for agents, not 100%, and blocking side effects on the new version.

## Q10 [priority: Important]
**Question:** An agent deploy bundles three independently-versioned artifacts. If the prompt is stored inside a `.py` file in the container image, what capability is lost?
- A) The ability to run unit tests
- B) Fast rollback — changing the prompt requires a full image rebuild and deploy
- C) Version control of the prompt
- D) The ability to use multiple models

**Answer:** B
**Explanation:** The converged industry pattern stores prompts as immutable, content-addressed artifacts with a movable label (e.g., `prod`). Rolling back a prompt = moving the label, taking seconds. If the prompt is inside the image, rollback requires rebuilding the image, running CI/CD, and redeploying — minutes instead of seconds. The three artifacts (image, prompt version, model ID) should have independent rollback paths.

## Q11 [priority: Important]
**Question:** Why does the lecture say serverless functions (Vercel/Lambda) are a poor fit for agents? What is the PRIMARY reason?
- A) Cold start latency of 5–15 seconds
- B) Hard timeout caps (5 min default, max ~900s for Lambda)
- C) Lack of GPU support
- D) No support for environment variables

**Answer:** B
**Explanation:** The lecture explicitly corrects the outdated claim that cold start is the main problem — Cloud Run GPU starts in ~5s, Modal snapshots are even faster. The real issue is hard timeout caps: Vercel 5 min (Hobby), Lambda hard cap 900s, and API Gateway's 29s sits in front of Lambda. When the timeout hits, the agent's work is lost. Additional issues: 4.5MB body cap and statelessness between invocations.

## Q12 [priority: Important]
**Question:** What does CVE-2025-66479 (Claude Code) teach about security defaults?
- A) Always use the latest version of any software
- B) Systems must fail-closed, not fail-open — the strictest config (`allowedDomains: []`) must not result in the most permissive behavior
- C) Never use allowlists, use blocklists instead
- D) Claude Code should not be used in production

**Answer:** B
**Explanation:** Setting `allowedDomains: []` (empty array, intended as "block everything") actually opened egress completely because the check used `length > 0` — an empty array failed the condition, bypassing the filter entirely. The lesson: default and edge-case semantics are a security property. A system must be designed to fail-closed (deny by default), not fail-open. This is a design principle, not a product-specific issue.
