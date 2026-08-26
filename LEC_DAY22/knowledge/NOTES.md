# LEC_DAY22 — LLMOps & Prompt Versioning

## Keypoints
- **LLMOps extends MLOps** with LLM-specific concerns: non-determinism, prompt-sensitivity,
  subjective quality (no ground truth), token-based cost, LLM safety (injection/jailbreak/PII),
  hallucination. Two structural shifts: tracking moves train-time → run-time (every inference is
  the interesting event); cost moves fixed → variable-per-request (a quality change is silently
  also a cost change).
- **7-dimension MLOps↔LLMOps table** (Tracking / Output / Versioning / Evaluation / Cost / Safety /
  Tools). Prompts + system instructions become first-class *versioned* artifacts.
- **temperature=0 ≠ determinism** in practice (GPU float non-associativity, batch effects, MoE
  routing, silent provider updates) — and determinism was never the real problem; *untraceable
  change* was.
- **Trace = tree** (ROOT run → child runs: retriever/chain/LLM), each with latency/tokens/cost.
  Purpose = *attribution*: turn optimization from guessing into arithmetic. Enabled by env vars
  (auto-instrument LangChain) or `@traceable` / `@weave.op()` for custom code.
- **LLM latency model**: `prefill(input_tokens) + output_tokens × time_per_token`. Decode is
  sequential (dominates when output is long); prefill is parallel-per-token but still scales with
  input size (dominates at large context → "context bloat"). Same variable drives latency AND cost.
- **Streaming reduces *perceived* latency (time-to-first-token), not total latency.**
- **Averages hide tails** → use P95, not mean, for user-facing latency.
- **Accuracy is undefined for open-ended generation** — it requires a single ground-truth to
  compare against; free-form text has none. Misses hallucination, off-topic-fluent, context
  mismatch, format errors.
- **RAGAS 2×2**: columns = retriever (Context Precision / Context Recall) vs generator
  (Faithfulness / Answer Relevance). The *pattern* of which metrics drop localizes the bug.
  Faithfulness only checks grounding-in-retrieved-context, NOT whether that context was right →
  can be high on a bad answer grounded in wrong docs.
- **LLM-as-Judge**: stronger LLM scores weaker model's output; scalable/cheap/consistent/
  reference-free but has positional/verbosity/self-preference bias → never judge with the same
  model that generated.
- **Hybrid calibration loop**: judge daily → human review weekly (sample 100–200) → Cohen's κ
  monthly; κ<0.7 ⇒ fix rubric or swap judge. Human anchor is permanent — it's the only instrument
  that can *measure* judge drift; remove it and you go blind.
- **Cohen's κ** = inter-rater agreement corrected for chance (κ=1 perfect, 0 random, >0.7 healthy).
- **Prompt Hub** = "GitHub for prompts": pull latest vs pin `:hash` (production pins for
  reproducible deploys), push with conventional commit message, typed variables, 1-click rollback,
  team review. **Prompt Hub vs Git YAML** decided by one axis: should prompt ship *independently*
  of code? Yes (non-eng editors, live A/B) → Hub; No (prompt+code change atomically, technical team)
  → YAML.
- **Guardrails = Defense in Depth**: input guards (PII, injection, jailbreak, toxicity-in) protect
  system/LLM; output guards (JSON, toxicity-out, grounding, length) protect user/downstream. Place
  by "what am I protecting & does the risk exist yet?"
- **`on_fail` policy**: `reask` (stochastic failure the model might fix — bad JSON), `fix`
  (deterministic mandatory correction — PII redaction), `noop` (observe > intervene — mild toxicity).
  `num_reasks` bounds retries. Alert rules escalate by severity (log → alert human → block+audit).
- **3 Key Takeaways**: prompt=code (version it); LLM-judge needs calibration; defense in depth.
- **Meta-theme (student-derived)**: every cheap scalable shortcut needs a smaller, trusted,
  expensive *anchor* to stay honest — remove it and the system rots while dashboards stay green.

## Terms
- LLMOps (vận hành LLM) — ops discipline for LLM systems, extends MLOps.
- Non-determinism (tính bất định) — same input → different output; makes bugs non-reproducible.
- Prompt-sensitivity (độ nhạy prompt) — one word can swing behavior → version prompts like code.
- Prefill / decode — parallel input processing vs sequential token generation.
- Context bloat (phình ngữ cảnh) — oversized input inflating latency + cost; risks context-recall loss.
- Perceived latency (độ trễ cảm nhận) — time-to-first-token; what streaming improves.
- Faithfulness / Answer Relevance / Context Precision / Context Recall — the 4 RAGAS metrics.
- LLM-as-Judge (LLM làm giám khảo) — stronger LLM evaluates weaker model's output.
- Cohen's kappa (κ) — chance-corrected agreement between judge and human.
- Reproducible deploys (triển khai tái lập được) — same code + pinned prompt hash = same behavior.
- Defense in Depth (phòng thủ nhiều lớp) — stacked independent input+output safety layers.
- on_fail (reask / fix / noop) — Guardrails AI per-validator failure policy.

## Covered / To revisit
- [x] LLMOps vs MLOps, 7-dim table, structural shifts — solid, strong transfer answers.
- [x] Trace anatomy & bottleneck attribution — solid.
- [x] LLM latency model (prefill vs decode) — got there after correction; now solid. **Watch:**
  initially over-generalized "the LLM step is the bottleneck → change provider" and missed the
  `input_tokens=8400` prefill signal. Reinforced.
- [x] Streaming = perceived not real latency — solid.
- [x] RAGAS 2×2 and reading the pattern to localize bugs — excellent (nailed retriever-bug +
  faithful-on-wrong-context).
- [x] Prompt Hub pinning + case-study forensics + Hub-vs-YAML decision — solid.
- [x] LLM-as-Judge bias + calibration loop + why human anchor is permanent — solid.
- [x] Guardrails placement (input vs output) — 4/4 correct.
- [x] on_fail policies + alert rules — solid after sharpening the reask-vs-fix reasoning.

## Misconceptions / Examiner findings
- **Hyperparameters** (S01) — first instinct was "can't track hyperparameters anymore." Corrected:
  they don't disappear, they change shape (temp/top_p still tracked); the real break is
  config-as-prose (prompt). Points to: tendency to say a thing "disappears" when it actually
  "moves." Now understood.
- **Latency attribution** (S02) — when output_tokens was small, attributed a 900ms LLM call to
  "per-step latency" and reached for "change provider," overlooking `input_tokens=8400`. Gap:
  didn't use the numbers on the slide; treated prefill as always-negligible. Fixed — now knows
  prefill dominates at large context. Worth a re-probe next session.
- **Accuracy collapse** (S04) — explained why accuracy is meaningless for RAG via
  "retriever vs model" (wrong axis) instead of "accuracy requires ground truth, which open-ended
  generation lacks." Redirected; understood after. Recall-level gap on the *definitional* reason —
  quiz targets this.
- **Self-preference bias** (S04) — named the bias correctly but gave the definition instead of the
  requested *concrete A/B scenario*; also slightly conflated "judges its own prompt" with the real
  same-model-generates-and-judges setup. Built the poisoned-A/B example together. Weak-area:
  constructing scenarios rather than restating; correlated-bias-vs-noise distinction is new.
- **reask vs fix on PII** (S06) — right conclusion (reask wrong for PII) but reasoning leaned on
  "PII is deterministic" rather than the core "safety corrections must be *guaranteed*, and you
  don't retry your way to a guarantee." Reinforced.
