# Core Knowledge

**Deck:** AICB-P2T2 · Chương 5: Vận Hành · Tuần 5 — "LLMOps & Prompt Versioning" (Ngày 22 / Day 22, VinUniversity, AI Infrastructure track). 37 slides/pages, all text-extracted successfully (pdftotext -layout), no garbled/vision-fallback pages.

Opening framing question (explicit, p.1-2): "Prompt thay đổi = behavior thay đổi" ("prompt change = behavior change"), illustrated by a case study where a 1-line system-prompt edit ("output chi tiết hơn một chút" / "slightly more detailed output") caused 3x latency increase, 200% token cost increase, and no one could trace why because prompt versions weren't tracked. This framing motivates the whole lecture: LLMOps = trace · version · eval every change.

Course structure (p.3): 8 sections — (01) LLMOps vs MLOps Truyền Thống [20 min], (02) LangSmith: Trace, Debug & Monitor [40 min], (03) Prompt Hub: Version Control [30 min], (04) LLM Evaluation: Vượt Ra Ngoài Accuracy [20 min], (05) W&B Weave [15 min], (06) Guardrails & Safety Monitoring [15 min], (07) Live Demo & Labs, (08) Key Takeaways & Preview Day 23.

Lecture objectives (p.4, explicit): master LangSmith tracing; implement prompt versioning via Prompt Hub; set up systematic LLM evaluation (W&B Weave + RAGAS) for faithfulness/relevance/hallucination; apply safety guardrails (PII blocking, prompt-injection detection, reask on bad format).

Lab #22 deliverables (p.5, explicit): (1) LangSmith project with >100 traces from a real RAG pipeline, analyzed for latency/cost/error rate; (2) Prompt Hub with 2 versions + clear commit messages + 50/50 A/B routing implemented; (3) RAGAS evaluation report on 50 QA pairs, faithfulness score > 0.8, comparing 2 prompt versions; (4) Guardrails AI validator blocking PII (email, phone), auto-reformatting non-JSON outputs, logging every incident.

---

## 1. Core Concepts

- **LLMOps** (p.7-11) — Operations discipline for LLM-based systems, extending MLOps with challenges specific to LLMs. Encompasses tracking, versioning (including prompts), evaluation (beyond accuracy), and safety. Matters because LLM behavior is non-deterministic and prompt-sensitive, so without LLMOps discipline teams can't diagnose cost/latency/quality regressions (per the opening case study). Related: MLOps, Prompt Versioning, LLM Evaluation, Guardrails. [p.7-11]

- **MLOps (traditional)** (p.7, explicit) — Operations process for traditional ML systems, covering: Tracking (hyperparameters, train/eval metrics, model artifacts/checkpoints, data versions), Versioning (model weights .pkl/.pt, training datasets, feature pipelines, code/Git), Evaluation (Accuracy, F1, AUC-ROC, Precision/Recall, confusion matrix, benchmark datasets), Safety (model fairness metrics, bias detection, data privacy, adversarial robustness). Serves as the baseline LLMOps is compared against. [p.7]

- **Non-determinism** (p.8, explicit) — Same input can produce different LLM outputs each run, making bugs hard to reproduce — a defining difference from traditional ML/software. [p.8]

- **Prompt-sensitivity** (p.8, explicit) — Changing even one word in a prompt can completely change model behavior; this is why prompts must be version-controlled like code. [p.8]

- **Subjective quality / no ground truth** (p.8, explicit) — LLM output quality often lacks a clear ground truth, requiring LLM-as-judge or human evaluation instead of simple accuracy metrics. [p.8]

- **Token cost model** (p.8, p.11, explicit) — LLM cost is billed per token, not per compute-time; longer prompts/outputs directly increase spend. Tracked metric example: ~$0.002 avg per query. [p.8, p.11]

- **Safety concerns specific to LLMs** (p.8, explicit) — Prompt injection, jailbreak, PII leak — attack surfaces that don't exist in traditional MLOps. [p.8]

- **Hallucination** (p.8, p.23-24, explicit) — The LLM fabricates information not grounded in truth or provided context; must be continuously tracked (target: hallucination rate < 5%, p.11). In RAG specifically, hallucination can occur when the model ignores retrieved context and instead uses parametric (training-data) knowledge — termed **context mismatch** (p.23). [p.8, p.23]

- **LLMOps Stack (5 layers)** (p.10, explicit) — A layered toolchain: (1) Safety Layer — Guardrails AI, Llama Guard 2, NeMo Guardrails: block harmful input/output, detect PII, classify harm categories; (2) Evaluation Layer — RAGAS, Promptfoo, DeepEval, LLM-as-Judge: measure faithfulness/relevance/hallucination, A/B test prompts; (3) Tracing Layer — LangSmith, W&B Weave, Phoenix, Arize: trace every call, debug errors, analyze latency & cost per run; (4) Prompt Management — LangSmith Prompt Hub, YAML in Git, Promptsmith: version/store/review/pin prompts in production; (5) Cost & Observability — Helicone, OpenMeter, Prometheus, Grafana: track token cost, request volume, latency P50/P95, budget alerts. [p.10]

- **LangSmith** (p.13-16, explicit) — Observability platform built specifically for LLM applications. Core capabilities: Tracing (full execution tree: root run → child runs — LLM calls, tool calls, retrieval — with per-step latency breakdown), Monitoring (dashboards for latency heatmap, cost/day/user, error rate trend, real-time token usage), Datasets (annotate traces to auto-build evaluation datasets that grow organically from production traffic), Prompt Hub (store/version/pull/push prompts, pin exact version by commit hash, A/B test), Evaluations (run automated evals on datasets, compare across deployments to detect regressions), Human Review (Annotation Queues — human reviewers label outputs, build golden datasets for calibration). [p.13]

- **Trace / Trace Anatomy** (p.14-15, explicit) — A recorded execution: ROOT run (e.g. rag_pipeline) with child runs — e.g. RETRIEVER (vectorstore.similarity_search, 120ms, 3 docs), CHAIN (format_prompt, 5ms), LLM (gpt-4o-mini, 1.4s — flagged as bottleneck), which itself has input_tokens/output_tokens/cost and finish_reason/latency. Purpose: pinpoint exactly which step is the bottleneck (e.g., "bottleneck ở LLM call, retrieval nhanh, format nhẹ → tối ưu bằng giảm output token hoặc streaming"). [p.15]

- **@traceable / auto-instrumentation** (p.14, explicit) — LangSmith is enabled via environment variables (LANGCHAIN_TRACING_V2, LANGCHAIN_API_KEY, LANGCHAIN_PROJECT); LangChain code is auto-instrumented with zero code change; non-LangChain custom code is traced via the `@traceable` decorator. [p.14]

- **Trace filtering** (p.14, explicit) — Traces can be filtered by: error traces only, latency > 2 seconds, cost > $0.01/call, by model/tag/date range. [p.14]

- **Prompt Hub** (p.13, p.18-21, explicit) — LangSmith's prompt version-control system, described as "giống GitHub cho prompts" (like GitHub for prompts). Supports pull (fetch latest or pin exact version via `:commit_hash`), push (with commit message), and A/B testing of prompt versions. [p.13, p.19]

- **Prompt Versioning** (p.18-21, explicit) — The practice of treating prompts as code: every change gets a commit message, production pins an exact version by hash, rollback is 1-click, A/B tests use isolated versions, and there's a full audit trail (who/when/what changed). Contrasted against unversioned prompt editing, which caused the opening case study's untraceable cost/latency regression. [p.18]

- **Git-native prompt alternative (YAML in repo)** (p.21, explicit) — Alternative to Prompt Hub: store prompts as YAML files (with name, version, description, variables, template) in `/prompts/*.yaml`, changed via Pull Request, tested by CI, loaded from YAML instead of hardcoded, diffed in GitHub/GitLab. Guidance: use Prompt Hub for large teams needing UI review + automatic A/B testing; use Git YAML for dev-centric teams, monorepos, tight CI/CD integration. [p.21]

- **RAGAS** (p.24, explicit) — RAG Evaluation metrics framework with 4 core metrics: Faithfulness (is output grounded in context / hallucination detection — % of answer statements verifiable from context; target > 0.8), Answer Relevance (does the answer match the question — similarity via reverse-generation; target > 0.75), Context Precision (are retrieved chunks actually relevant — % retrieved chunks in the ideal answer; target > 0.7), Context Recall (is there enough context to answer correctly — % of ground-truth facts present in context; target > 0.8). [p.24]

- **LLM-as-Judge** (p.25, explicit) — Using a stronger LLM (example given: GPT-5) to evaluate the output of a smaller model (example: gpt-4o-mini), described as ~10x more scalable than human eval. Pipeline: Input+Context → Small LLM → generated Output → Judge LLM. Advantages: scalable (thousands of evals/day), 10-50x cheaper than human eval, consistent (no fatigue), reference-free (no ground truth needed), established frameworks exist (G-Eval, MT-Bench). Limitations: positional/verbosity bias in the judge; needs periodic calibration against human judgments (measured via Cohen's kappa); should not use the same model to judge itself; judge prompt/rubric must be very explicit. [p.25]

- **Human Eval & Hybrid Approach** (p.26, explicit) — Best practice combines LLM-as-judge (daily, full eval suite, cheap/scalable) with Human Review (weekly, sample 100-200 outputs via Label Studio/Argilla) and Calibration (monthly, compare LLM judge vs human via Cohen's kappa; if kappa < 0.7, adjust rubric or change judge model). Tools: Label Studio (open source), Argilla (NLP-focused), Scale AI (enterprise), Prolific (crowdsourcing). Inter-annotator agreement (IAA) > 0.7 considered healthy; gold-standard samples used to calibrate the LLM judge. [p.26]

- **W&B Weave** (p.28-29, explicit) — LLM evaluation platform with auto-tracking: `weave.init()` auto-tracks all decorated (`@weave.op()`) function calls (input/output/latency) with no extra config. Features: Auto-trace, Dataset versioning (new dataset version auto-created when data changes, tracks eval evolution over time), Model comparison (side-by-side comparison of e.g. GPT-4o vs Llama-3 vs Claude on the same eval suite), Cost tracking (total token cost per eval run, budget-aware development). Includes a custom `Scorer` class (e.g. `FaithfulnessScorer`) for defining evaluation metrics, which can itself use LLM-as-judge internally. [p.28]

- **Guardrails (safety architecture)** (p.31-32, explicit) — "Defense in depth" architecture with two layers around the LLM: Input Guardrails (detect PII, prompt injection, jailbreak attempts, toxicity filter) and Output Guardrails (validate JSON format, toxicity check, factual grounding, length compliance). Tools: Guardrails AI (Python library — validate JSON, detect PII, check toxicity, auto-reask on failure), Llama Guard 2 (Meta open-source safety model, classifies input/output into 14 harm categories), NeMo Guardrails (NVIDIA, dialog safety rails — topical & safety — for conversational AI), Azure Content Safety (Microsoft managed service, text+image moderation, customizable thresholds). [p.31]

- **Guardrails AI (library specifics)** (p.32, explicit) — Built via `Guard().use_many(...)` composing validators: `ValidJson(on_fail="reask")` (retry the LLM if output isn't valid JSON), `DetectPII(pii_entities=[...], on_fail="fix")` (auto-redact PII), `ToxicLanguage(threshold=0.7, on_fail="noop")` (log but don't block). Guard wraps the LLM call with `num_reasks` controlling retry count. [p.32]

- **Key LLMOps metrics** (p.11, explicit) — Example targets shown: Token Cost/Task ~$0.002 avg/query, Hallucination Rate threshold < 5%, Faithfulness Score (RAGAS) target > 0.8, Latency P95 SLA target < 3s. Additional metrics mentioned: user satisfaction (thumbs up/down, CSAT), context precision & recall (RAG quality), prompt injection attempt rate, cache hit rate (reduces redundant LLM calls), error rate by error type (timeout, rate limit, invalid output), model drift (output quality comparison over time). [p.11]

---

## 2. Relationships & Mechanisms

- **MLOps → LLMOps (extends)**: LLMOps is explicitly framed as extending MLOps, adding LLM-specific concerns (non-determinism, prompt sensitivity, subjective quality, token cost, safety, hallucination) on top of the traditional ML operational disciplines (tracking, versioning, evaluation, safety). [p.7-9, explicit]

- **MLOps vs LLMOps comparison table** (p.9, explicit) — direct mapping across 7 dimensions:
  | Aspect | MLOps | LLMOps |
  |---|---|---|
  | Tracking | Hyperparams, train/eval metrics | Per-call trace, token cost per request |
  | Output | Deterministic, reproducible | Non-deterministic, subjective quality |
  | Versioning | Model weights, datasets, code | + Prompts, system instructions |
  | Evaluation | Accuracy, F1, AUC-ROC | Faithfulness, Relevance, Hallucination rate |
  | Cost model | Compute time (GPU hours) | Token cost per task |
  | Safety | Model fairness, bias | Prompt injection, jailbreak, PII leak |
  | Key tools | MLflow, W&B, DVC | LangSmith, W&B Weave, Helicone, RAGAS |

- **Trace anatomy mechanism (Input → Process → Output)**: User query → ROOT run (rag_pipeline) → RETRIEVER (fetches docs, scored) → CHAIN (formats prompt) → LLM call (generates answer, tracked for tokens/cost/latency) → final output. Each step's latency is broken down so the bottleneck step is identifiable (why it matters: enables targeted optimization, e.g., reduce output tokens or use streaming when the LLM step dominates latency). [p.15, explicit]

- **Dataset creation pipeline (Input → Process → Output)**, p.16, explicit: (1) Production traces run continuously → (2) Annotate important traces (correct/incorrect, edge cases) → (3) Add to evaluation dataset (1-click) → (4) Dataset grows organically from production → (5) Run regression test on every new deploy. Why it matters: closes the loop from real usage to systematic regression testing, avoiding blind deploys.

- **Prompt A/B testing workflow (Input → Process → Output)**, p.20, explicit: Incoming request → Traffic Router (hash of request_id % 2) → 50% routed to Prompt v1 (concise style) / 50% to Prompt v2 (detailed style) → Evaluation on faithfulness · relevance · cost · latency. Best practice: collect > 100 queries before concluding; tag every trace with its prompt version in LangSmith for accurate comparison. Depends on: Prompt Versioning (isolated pinned versions) + LangSmith tracing (tagging).

- **Prompt push/pull mechanism with best practices** (p.19, explicit): pulling latest (`hub.pull("org/name")`) vs pinning exact version (`hub.pull("org/name:hash")` — production best practice, prevents surprise updates); pushing requires a commit message (feat/fix/refactor prefix, Git-conventional-commit style); typed variables (e.g. `{context}`, `{question}`) enable input validation; team review process (one person pushes, another reviews before deploy).

- **LLM-as-Judge pipeline (Input → Process → Output)**, p.25, explicit: Input + Context → Small LLM (e.g. gpt-4o-mini) generates Output → stronger Judge LLM (e.g. GPT-4o/GPT-5) scores the output. Trade-off: scalable/cheap/consistent vs. subject to bias, requiring periodic calibration.

- **Hybrid eval calibration loop (Input → Process → Output)**, p.26, explicit: LLM-as-Judge runs daily (full suite) → Human Review weekly (sample 100-200 outputs) → Calibration monthly (compute Cohen's kappa between judge and human) → if kappa < 0.7, Update Judge Prompt (adjust rubric or change judge model). This is a feedback loop ensuring the cheap/scalable judge stays trustworthy — depends on human eval as ground truth anchor.

- **Guardrails input/output layering (mechanism)**, p.31, explicit: User Input passes through Input Guardrails (PII detection, prompt injection/jailbreak detection, toxicity filter) before reaching the LLM (GPT/Claude); LLM output passes through Output Guardrails (JSON validation, toxicity check, factual grounding, length compliance) before reaching the user. Rationale (explicit, Key Takeaway #3, p.850-852): "Defense in Depth" — don't rely on a single control point; each layer catches different failure types.

- **Guardrails failure-handling mechanism**, p.32, explicit: `on_fail` policy determines behavior per validator — "reask" (retry the LLM call, e.g. for invalid JSON), "fix" (auto-redact, e.g. for PII), "noop" (log only, don't block, e.g. for moderate toxicity). `num_reasks` bounds retry attempts.

- **Alert rules mechanism** (p.32, explicit): threshold-based automated responses — Toxicity rate > 0.5% → Slack alert + manual review; Hallucination rate > 5% → page on-call + investigation; PII leak detected → immediate block + audit log; reask count > 3/hour → review prompt quality; Guard failure rate > 2% → check guard configuration. Mechanism: metrics from tracing/eval/guardrails layers feed into alerting thresholds that trigger operational responses — ties the Tracing, Evaluation, and Safety layers of the LLMOps Stack together operationally.

- **Trade-off: Accuracy vs LLM-specific eval metrics** — accuracy alone is presented as insufficient (p.23) because it misses hallucination (no ground truth to compare against), off-topic-but-fluent answers, RAG context mismatch, and format non-compliance — motivating the shift to RAGAS/LLM-as-judge metrics (Section 04 as a whole is essentially "why accuracy fails → what replaces it").

- **Prompt Hub vs Git YAML (decision framework)**, p.21, explicit: not a strict either/or but a "when to use which" — Prompt Hub for larger teams needing UI-based review and automated A/B testing; Git YAML for dev-centric teams, monorepos, and tight CI/CD integration.

---

## 3. Examples & Distinctions

- **Case study example** (p.1-2, explicit): 1-line system prompt change ("more detail") → 3x latency increase + 200% token cost increase + untraceable cause, because prompt versions weren't tracked. Used to motivate the entire lecture's premise.

- **RAG hallucination example — context mismatch** (p.23, explicit): in a RAG pipeline, model ignores the retrieved context and instead answers from parametric (training) knowledge, effectively fabricating an answer not grounded in the provided context.

- **Medical-context accuracy failure example** (p.23, explicit): a model with 95% accuracy can still be harmful if the remaining 5% of failures are hallucinations occurring in a medical (y tế) context — illustrating why raw accuracy is an incomplete quality signal.

- **Model comparison example table** (p.29, explicit) — Weave comparing 5 models on the same eval suite (Faithfulness / Relevance / Latency P95 / Cost per 1M tokens / overall star rating): GPT-5-4 (0.91 / 0.88 / 3.2s / $2.5 / 5 stars), GPT-5.4-mini (0.82 / 0.80 / 1.1s / $0.75 / 4 stars), Claude 4.6 Sonnet (0.89 / 0.86 / 2.8s / $3 / 5 stars), Llama-3.1 70B (0.78 / 0.75 / 4.5s / $0.05 / 3 stars), Gemini 3.1 Pro (0.80 / 0.79 / 1.8s / $2 / 4 stars). Illustrates the cost/quality/latency trade-off across model choices. [Note: these specific model names/versions (GPT-5-4, GPT-5.4-mini, Claude 4.6, Gemini 3.1) appear to be illustrative/placeholder figures in the slide rather than necessarily real released models — flagged as uncertain whether these are actual product names or example placeholders.]

- **Guardrails AI code example** (p.32, explicit): `ValidJson(on_fail="reask")`, `DetectPII(pii_entities=["EMAIL","PHONE","SSN"], on_fail="fix")`, `ToxicLanguage(threshold=0.7, on_fail="noop")` — demonstrates three distinct on-fail behaviors in one pipeline.

- **A vs B distinctions:**

  - **MLOps vs LLMOps**: Similarity — both are operational disciplines for tracking, versioning, evaluating, and securing ML-based systems. Difference — MLOps assumes deterministic/reproducible output and cost measured in compute time; LLMOps must handle non-deterministic, subjective-quality output, prompt-based versioning, token-based cost, and LLM-specific safety risks (injection/jailbreak/PII leak vs. fairness/bias). Distinguishing criterion: presence of prompts as a first-class versioned artifact and non-deterministic output. [p.7-9]

  - **Accuracy vs RAGAS/LLM-eval metrics**: Similarity — both attempt to measure "correctness" of output. Difference — Accuracy requires ground truth and exact/near matching, missing hallucination, off-topic fluency, and format errors; RAGAS/LLM-as-judge metrics (faithfulness, relevance, context precision/recall) are designed to catch these LLM-specific failure modes, some without needing ground truth (reference-free). [p.23-25]

  - **LLM-as-Judge vs Human Eval**: Similarity — both assess subjective output quality. Difference — LLM-as-judge is scalable (thousands/day), 10-50x cheaper, consistent, reference-free, but carries bias (positional/verbosity) and needs calibration; Human eval is the calibration ground truth, slower/costlier, sampled (100-200/week) rather than exhaustive. Distinguishing criterion: LLM-as-judge trades some accuracy/trust for scale and cost; human eval anchors trust via periodic Cohen's kappa calibration. [p.25-26]

  - **Prompt Hub vs Git YAML prompt management**: Similarity — both provide versioning, review, and audit trail for prompts. Difference — Prompt Hub offers a dedicated UI, built-in A/B testing, commit-hash pinning within LangSmith; Git YAML is plain-file-based, uses standard PR/CI workflows, fits dev-centric/monorepo teams without adding a new tool. [p.21]

  - **Input Guardrails vs Output Guardrails**: Similarity — both are validation layers around the LLM call. Difference — Input guardrails act on what goes into the LLM (PII, prompt injection, jailbreak, toxicity of user input); Output guardrails act on what comes out (JSON validity, toxicity of output, factual grounding, length compliance). Distinguishing criterion: position relative to the LLM call and the class of risk each catches. [p.31]

---

## 4. Assumptions, Boundaries & Gaps

- **Prerequisite (implicit/inferred)**: the lecture assumes learners already understand traditional MLOps concepts (hyperparameters, model versioning, standard ML evaluation metrics) since Section 01 explicitly frames itself as "Nhắc Lại" (recap) before contrasting with LLMOps — this is inferred as a prerequisite rather than explicitly stated as one. [p.7]

- **Prerequisite (implicit)**: familiarity with RAG (Retrieval-Augmented Generation) pipeline architecture is assumed — the deck uses RAG as the running example throughout (trace anatomy, RAGAS metrics, A/B testing, labs) without defining RAG itself. [multiple pages, e.g. p.15, p.23-24]

- **Gap — LLM-as-Judge mechanics not detailed**: the slides state LLM-as-judge frameworks "G-Eval, MT-Bench" exist but don't explain how they work internally (prompting technique, scoring rubric construction). Flagged as mentioned-but-not-explained. [p.25]

- **Gap — Cohen's kappa not defined**: the metric is used repeatedly (calibration mechanism, p.26) as the standard for measuring judge-human agreement, with a stated healthy threshold (>0.7 IAA, kappa<0.7 triggers rubric change) but the slides never define what Cohen's kappa is or how it's computed. Flagged as a concept the lecture relies on without explaining.

- **Gap — RAGAS metric calculation methods are high-level only**: the "Cách Tính" (how it's calculated) column in the RAGAS table (p.24) gives only brief descriptions (e.g. "% statements in answer verifiable from context") without the underlying algorithm (e.g. how statement extraction or verification is actually performed). Flagged as insufficiently explained for deeper mechanism understanding.

- **Boundary/limitation — LLM-as-judge bias**: explicitly stated limitations include positional bias, verbosity bias, and the rule that a model should not judge its own output. These are stated as known failure modes requiring mitigation (calibration), not fully solved. [p.25]

- **Boundary — target thresholds appear to be illustrative examples, not universal standards**: metrics like "Faithfulness > 0.8", "Hallucination < 5%", "Latency P95 < 3s", "$0.002/query" are presented as example targets (p.11) without stating whether they are industry benchmarks, VinUniversity-lab-specific targets, or illustrative numbers only. Uncertain whether these generalize beyond this lecture/lab context.

- **Boundary — guardrail failure modes not exhaustively covered**: alert rules (p.32) cover toxicity, hallucination, PII leak, reask count, and guard failure rate, but the guardrails architecture doesn't address what happens when Input and Output guardrails disagree or when a request must be blocked mid-generation (streaming) — not addressed in the deck.

- **Explicit boundary/best-practice constraint**: A/B testing requires collecting >100 queries before drawing conclusions (p.20) — stated as a best practice, functioning as a minimum-sample-size constraint, though no statistical justification (confidence interval, power analysis) is given.

- **Uncertain — specific model figures**: the model comparison table (p.29) lists "GPT-5-4," "GPT-5.4-mini," "Claude 4.6 Sonnet," "Gemini 3.1 Pro" — naming/versioning that doesn't clearly correspond to confirmed real product releases as of the lecture's likely writing; flagged as possibly illustrative/placeholder rather than factual, should not be treated as ground truth about actual model performance.

- **Forward pointer (not this lecture's content)**: Day 23 preview (p.36) covers Monitoring & Observability Stack — OpenTelemetry instrumentation, Prometheus metrics for LLM, Grafana dashboards, alert rules & incident response. Explicitly out of scope for Day 22's core content, mentioned only as what's next.

---

## 5. Learning Priorities

**Essential** (required to understand the lecture):
- LLMOps vs MLOps distinction and the 7-dimension comparison table (p.9)
- Why prompts must be version-controlled (prompt-sensitivity + case study) (p.1-2, p.8, p.18)
- LangSmith tracing fundamentals: trace anatomy (root/child runs), bottleneck identification (p.14-15)
- Prompt Hub mechanics: pull/push/pin by commit hash, commit messages (p.19)
- Why accuracy is insufficient for LLMs; RAGAS's 4 core metrics (faithfulness, relevance, context precision/recall) (p.23-24)
- LLM-as-Judge concept: what it is, its advantages/limitations, need for calibration (p.25)
- Guardrails input/output layering ("Defense in Depth") and on_fail behaviors (reask/fix/noop) (p.31-32)
- The 3 Key Takeaways (prompt-as-code; LLM-as-judge needs calibration; defense in depth) (p.36)

**Important** (substantially improves understanding):
- LLMOps 5-layer stack and representative tools per layer (p.10)
- A/B testing workflow for prompts (traffic router, tagging, evaluation dimensions) (p.20)
- Dataset creation pipeline from production traces → regression testing (p.16)
- Human eval + hybrid calibration loop (daily/weekly/monthly cadence, Cohen's kappa) (p.26)
- W&B Weave's auto-tracking, model comparison, and cost-tracking features (p.28)
- Guardrails alert-rule thresholds and what they trigger (p.32)
- Git YAML alternative to Prompt Hub and when to choose each (p.21)

**Supporting** (useful, not central):
- Specific example metric target numbers (p.11) — useful as reference, not conceptually central
- Specific tool lists beyond LangSmith/RAGAS/Guardrails AI (e.g. Phoenix, Arize, Helicone, OpenMeter, Promptsmith) (p.10)
- Model comparison example table's specific numbers (p.29) — illustrative, not conceptually load-bearing
- Live Demo agenda structure and exact timings (p.34) — logistical, not conceptual
- Lab #22 task-by-task instructions (p.35) — execution detail, not core knowledge
- Day 23 preview topics (p.36) — explicitly next lecture's content, not this one's
