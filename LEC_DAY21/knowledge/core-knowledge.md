# Core Knowledge

**Source:** `stream (1).pdf` — "CI/CD for AI Systems" (Tự động hoá vòng đời model: từ thí nghiệm đến production), AICB-P2T2 · Ngày 21 · Chương 5: Vận Hành. 40 slides (numbered 2/38–40/38 on-slide; PDF has 40 pages total incl. title slide). Language: Vietnamese with English technical terms/code.

Opening framing (p.2, explicit): case study — team deploying manually weekly had 3 undetected regressions (only found via user feedback); after adopting a CI/CD pipeline, zero regressions reached production over 6 months. This framing anchors the lecture's core thesis: AI CI/CD's main value is regression prevention via automated gates.

---

## 1. Core Concepts

- **MLflow** (p.7) — open-source platform for managing the full ML lifecycle, from experimentation to production. 4 core components:
  - **Tracking**: log parameters, metrics, artifacts, source code; compare runs via web UI/API. (p.7–9)
  - **Projects**: package code for reproducibility; rerun any run on any platform. (p.7)
  - **Models**: standardized packaging format (mlflow.sklearn, .pytorch, .pyfunc); deploy across platforms. (p.7)
  - **Registry**: lifecycle management — None → Staging → Production → Archived; review/approve workflow before promotion. (p.7, p.10)
  - Why it matters: gives systematic experiment comparison and a governed path for promoting models to production.
  - Related: DVC (data half of reproducibility), eval gate (uses Registry to fetch baseline).

- **MLflow Tracking — loggable entities** (p.8, explicit):
  - **Params**: hyperparameters, data version, model config — immutable within a run.
  - **Metrics**: loss, accuracy — logged per step/epoch for comparison plots.
  - **Artifacts**: plots, confusion matrices, config files stored with the run for traceability.
  - **Signature**: input/output schema, auto-inferred (e.g. `infer_signature`) — required for serving.

- **MLflow LLM Autolog** (p.9, explicit) — MLflow 2.8+ feature; `mlflow.openai.autolog()`, `mlflow.langchain.autolog()` auto-log prompt templates, input/output content, token usage, per-call latency, model name/version, and RAG retrieval context without code changes. Supports OpenAI, LangChain, LlamaIndex, Anthropic Claude.

- **MLflow Model Registry lifecycle** (p.10, explicit) — stage machine: `None → (register) → Staging → (approve) → Production → (retire) → Archived`, with a `reject` path looping back. Key mechanisms:
  - **Version**: each model version has its own stage; multiple versions coexist for champion-vs-challenger A/B testing.
  - **Alias**: `champion` = current Production version, `challenger` = A/B candidate; load models by alias name.
  - **Annotations**: record promote/reject reasons, link to eval report, approver — full audit trail.
  - **Webhook**: triggers CI/CD on stage transition — e.g. auto-deploy staging when Staging is approved.

- **MLflow Registry best practices** (p.11, explicit): naming convention `{task}_{arch}_{version}` (e.g. `sentiment_bert_base`); tag strategy (`data_version`, `git_commit`, `eval_accuracy`) for incident traceback; approval workflow requiring ≥1 reviewer before Staging→Production, notified via Slack webhook; rollback plan — always keep the previous Production version Archived so rollback is one API call.

- **DVC (Data Version Control)** (p.13–17) — "Git for data": versioning, pipelines, reproducibility for datasets too large for Git.
  - **Problem it solves** (p.13, explicit, 4 named problems): (1) results not reproducible — unclear if data or code changed; (2) merge conflicts with large data — Git can't track GB files, shared folders get overwritten with no history; (3) experiments not tied to a data version — MLflow logs metrics but not which data version was used, making A/B comparison meaningless; (4) storage waste — everyone copies data locally, no deduplication.
  - **Pointer file (.dvc)** (p.14, explicit): a small file storing a hash of the data; Git tracks the `.dvc` file, so checking out code = checking out the correct data version.
  - **Content-addressable storage** (p.14, explicit): data stored by content hash — no extra space cost even with 100 versions (dedup).
  - **Remote storage**: supports S3, GCS, Azure Blob, SSH, HDFS via unified `dvc push`/`dvc pull` API (p.14, p.15).
  - **Offline-first**: work locally without network; sync to remote when needed, like git push/pull (p.14).
  - **DVC pipeline (`dvc.yaml`)** (p.16, explicit) — declares stages (`prepare`, `train`, `evaluate`) each with `cmd`, `deps`, `outs`, and `metrics`. `dvc repro` reruns only stale stages (smart caching); `dvc dag` visualizes the dependency graph; `dvc metrics show` diffs metrics.json across runs; `dvc params diff HEAD~1` diffs params vs a previous commit.
  - **DVC Experiments** (p.17, explicit): `dvc exp run --set-param lr=...` runs parameterized experiments (parallelizable with `--jobs`); `dvc exp show` tabulates results; `dvc exp apply` / `dvc exp branch` promote a chosen experiment into the workspace or a Git branch.

- **DVC vs MLflow** (p.17, explicit): "DVC: pipeline + data versioning. MLflow: metrics + model registry. → Use both, don't choose one." — they are complementary, not substitutes.

- **CI/CD for AI vs traditional software** (p.19, explicit table) — see Relationships & Mechanisms below for the full comparison.

- **CI Pipeline architecture for AI** (p.20, explicit): sequential stages `git push/PR → Data Validation → Model Training → Eval Gate → Deploy (if pass)`, with an explicit "Block Deploy" branch off Eval Gate when accuracy drop > 2%.

- **Eval Gate** (p.23, explicit) — described as the single most important safety net ("Safety Net Quan Trọng Nhất", also reiterated in Key Takeaways p.39). Mechanism: load both new and production models, evaluate both on a *fixed* held-out test set, compute `delta = new_acc - prod_acc`; if `delta < -threshold` (default threshold 0.02 i.e. 2%), the job exits with failure, blocking deploy. Best practices: use a fixed (non-shuffled) held-out set; track multiple metrics (accuracy, F1, AUC, latency P95) each with its own threshold (e.g. accuracy ±2%, latency ±10%); log eval results to MLflow for history; post results as a PR comment; require manual review for borderline passes instead of auto-deploying.

- **Great Expectations** (p.21–22, p.33, explicit) — data validation/quality tool used as a CI job (`great_expectations checkpoint run data_quality`). Checks: schema validation (columns/dtypes), null rate (≤5% per column), value range, distribution drift (KL-divergence vs baseline / KS test in p.33), duplicate rows (<0.1%), label balance (minority class ≥10%), volume check (≥10,000 rows/batch), freshness (data not older than 7 days).

- **GitHub Actions workflow structure for AI** (p.21, explicit): triggers (`on: push`/`pull_request` on main); `env` block referencing `secrets.*` (MLFLOW_TRACKING_URI, AWS keys) — never hardcoded; dependency caching via `actions/cache` keyed on `hash(requirements.txt)`, claimed to cut setup time 60–80%; job dependencies via `needs:` for sequential control flow (e.g. training only runs if data-validation passes); OIDC federation recommended for production over long-lived AWS keys (expires each run).

- **4 Deployment (CD) strategies** (p.26–28, explicit): **Canary**, **Blue/Green**, **Shadow (Dark Launch)**, **Rolling** — see Examples & Distinctions for full comparison.

- **Canary deployment mechanics** (p.27, explicit): load balancer (Istio/ALB) splits traffic (e.g. 95% v1 stable / 5% v2 canary); monitors P99 latency, accuracy, error rate; staged rollout `5% → 25% → 50% → 100%` traffic with a health check gate at each step; rollback trigger: P99 latency exceeds threshold OR accuracy drop >2% → auto rollback to v1.

- **Blue/Green deployment mechanics** (p.28, explicit): deploy v2 fully independently of running v1; run smoke/integration tests on v2; switch load balancer to route all traffic to v2; keep v1 alive ~30 minutes for instant rollback; after 30 min stable, shut down v1 to free resources. Use case: large deploys needing zero downtime. Trade-off: double infrastructure cost during the switch window.

- **Shadow Mode (Dark Launch) mechanics** (p.28, explicit): requests forwarded to both models; v1's response is returned to the user as normal; v2's (shadow) response is only logged, never returned; outputs compared on latency/predictions/errors. Use case: large model changes needing validation before exposure. Trade-off: zero user-facing risk, but doesn't test real user reaction, and costs double compute.

- **Rolling deployment** (p.26, explicit, briefly): replaces pods one at a time in a Kubernetes deployment. Pros: simple, K8s default, no extra infrastructure. Cons: slow rollback, mixed versions coexist mid-deploy.

- **Multi-Environment CD Pipeline** (p.29, explicit): three environments — Development (auto-deploy every commit, fast smoke test <2min, fake data), Staging (auto-deploy when dev passes, full integration tests, real data subset, eval gate vs baseline), Production (manual approval required, canary deployment, 30-min monitoring, rollback plan ready).

- **GitOps for AI (ArgoCD/Flux)** (p.29, explicit, brief mention): declarative deployment — every infra change goes through a git commit, auto-synced from repo; rollback via `git revert`. Flagged only briefly — see Gaps.

- **AI Testing Pyramid** (p.31, explicit) — 5 layers bottom-to-top: Unit Tests (pytest, fast — preprocessing/tokenization/feature engineering) → Integration Tests (pytest — end-to-end inference pipeline with sample inputs) → Model Tests (pytest + custom scripts — behavioral, performance regression) → Data Tests (Great Expectations — schema, distribution, quality) → Load Tests (k6/Locust — P95 <500ms at 50 RPS).

- **Model Tests — categories** (p.33, explicit): Behavioral (model must reject harmful content), Invariance (rotate image 90° → prediction unchanged), Directional (add "not" to a sentence → sentiment flips), Regression (accuracy on golden test set ≥ v_prev − 0.5%), Fairness (accuracy gap between subgroups <3%).

- **Load Tests — types** (p.33, explicit): Baseline (50 RPS, P95 <500ms, error rate <0.1%), Stress test (ramp to 500 RPS to find breaking point), Soak test (50 RPS for 1 hour, check memory leaks), Spike test (0→500 RPS suddenly, check auto-scaling); pipeline fails if any SLA is violated.

- **MLflow Model Serving deployment options** (p.35, explicit): Local (`mlflow models serve`, dev/test, fast startup), Docker (`mlflow models build-docker`, portable, staging), Cloud (AWS SageMaker, Azure ML, Databricks — managed infra, auto-scaling), Kubernetes (Seldon Core, KServe — production-grade, custom scaling policy).

- **A/B Testing for AI Models** (p.36, explicit): deterministic traffic routing via hashing `user_id` (MD5) so the same user always sees the same variant (consistency); minimum sample size 1,000 samples/variant before concluding, ideally sized via power analysis; statistical significance via p-value <0.05 (95% confidence), with Bonferroni correction mentioned for multiple comparisons; metric selection distinguishes Primary metrics (business KPI: CTR, revenue) from Guardrail metrics (latency P95, error rate must not increase).

## 2. Relationships & Mechanisms

- **CI/CD for AI vs traditional software** (p.19, explicit comparison table):
  | Aspect | Traditional Software CI/CD | AI CI/CD |
  |---|---|---|
  | Artifact | Binary / Docker image | Model weights + metadata |
  | Test | Unit test, integration test | + Model eval, data validation |
  | Versioning | Git for code | Git + DVC for code + data |
  | Deploy | Deterministic — same code | Non-deterministic — needs eval gate |
  | Rollback trigger | Error rate increase | + Accuracy drop, bias metrics |
  | Pipeline input | Code changes only | Code OR data changes |
  | Build time | A few minutes | Minutes → hours (training) |

- **CI Pipeline flow (Input → Process → Output)** (p.20–24, explicit):
  `git push/PR → [Data Validation] → [Model Training] → [Eval Gate] → [Deploy if pass]`
  - Data Validation (Great Expectations): why it matters — fail-fast; if data is bad, the pipeline stops before wasting GPU time on training.
  - Model Training: gated by `needs: [data-validation]` and path filters (only runs when `data/` or `src/` changed) to avoid retraining on doc-only changes.
  - Eval Gate: compares new model vs production baseline (fixed held-out set); why it matters — blocks deploy if accuracy drops >2%, acting as the final safety net before production exposure.
  - Deploy: only proceeds if Eval Gate passes.

- **DVC ↔ Git ↔ MLflow relationship** (p.14, p.17, explicit): `git commit` = code version; `dvc push` = data version; the `.dvc` pointer file links the two layers together. Anyone who does `git clone` + `dvc pull` reproduces the exact same result. MLflow (metrics/registry) and DVC (pipeline/data) are complementary — "use both, don't choose one" (p.17).

- **Model Registry stage transitions enable automation** (p.10, explicit): a Staging→Production webhook can auto-trigger a CD deploy — i.e., Registry lifecycle state is a mechanism that *enables* CD automation, not just bookkeeping.

- **Trade-off: deployment strategy risk vs cost** (inferred from p.26 table structure): all 4 strategies (Canary, Blue/Green, Shadow, Rolling) trade risk-reduction/rollback-speed against infrastructure cost or operational complexity — e.g. Canary needs sophisticated traffic-split monitoring; Blue/Green and Shadow both double resource/compute cost; Rolling is cheap/simple but has slow rollback and mixed-version risk.

- **Testing Pyramid as prerequisite chain** (inferred from p.31 layering, consistent with standard testing-pyramid ordering shown bottom-up: Unit → Integration → Model → Data → Load): the pyramid shape and volume ordering (unit tests most numerous/fastest, load tests fewest/most expensive) is explicit in the visual layout, though the lecture doesn't state explicit dependency rules between the layers (see Gaps).

- **A/B testing workflow** (p.36, explicit end-to-end): `route_request(user_id)` deterministically assigns variant A/B via hashed user_id → prediction served from the assigned model → outcome (click, latency) logged as MLflow metrics per variant → chi-squared/statistical significance test → declare winner if p<0.05. Why each step matters: hashing gives consistent user experience; minimum sample size avoids premature conclusions; guardrail metrics prevent optimizing a business KPI at the cost of latency/errors.

## 3. Examples & Distinctions

- **4 Deployment Strategies compared** (p.26, explicit table):
  | Strategy | How it works | Pro | Con |
  |---|---|---|---|
  | Canary | Route 5% traffic to new model, increase gradually if healthy | Detects errors early, step-wise risk control | Needs good monitoring, complex traffic-split logic |
  | Blue/Green | Deploy v2 fully parallel to v1, switch load balancer when ready | Zero downtime, instant rollback via load balancer | Double resource cost while both envs run |
  | Shadow | New model processes traffic but doesn't return response | Tests with real traffic without affecting users | Doesn't test user reaction, doubles compute cost |
  | Rolling | Replace pods one at a time in K8s deployment | Simple, K8s default, no extra infra | Slow rollback, mixed versions during deploy |

- **Canary vs Blue/Green vs Shadow (distinguishing criterion)**: Canary = partial live traffic split with gradual increase; Blue/Green = full traffic switch at once between two complete parallel environments; Shadow = full traffic mirrored to new model but response never shown to user (zero user-facing risk, but no real user-reaction signal). Rolling differs from all three by operating at the pod/instance level within a single environment rather than routing between two full deployments.

- **DVC vs MLflow** (p.17, explicit): similarity — both support ML reproducibility/experiment comparison; difference — DVC handles data versioning + pipeline execution; MLflow handles metrics tracking + model registry/lifecycle. Distinguishing criterion: DVC = "what data, what pipeline stage produced this"; MLflow = "what run, what model version, what stage in production lifecycle."

- **Champion vs Challenger** (p.10, explicit): champion = current Production model version (alias); challenger = A/B test candidate version. Distinguishing criterion: stage/role in the Registry, not a difference in model type.

- **Concrete example — regression prevention case study** (p.2, p.39, explicit): manual weekly deploys → 3 undetected regressions found only via user complaints; after CI/CD adoption → 0 regressions in 6 months. Used both as the lecture's opening hook and its closing key takeaway, indicating it's the central illustrative example tying Eval Gate + Canary together.

- **Concrete example — eval gate code** (p.23, explicit): `compare_models.py` loads new and prod models, evaluates both, computes delta, exits with failure code if `delta < -threshold` — illustrates the eval gate mechanism concretely (not just conceptually).

- **Concrete example — live demo pipeline** (p.38, explicit): `git push → GitHub Actions trigger → DVC pull + dvc repro → Train + log MLflow → Eval Gate (compare vs baseline) → Canary deploy 5%→100%`, plus a bonus step simulating a deliberately degraded model to demonstrate the eval gate auto-blocking deploy — described as taking "8 minutes" end to end (title of slide 38, p.38).

## 4. Assumptions, Boundaries & Gaps

- **Assumption**: learners already understand basic Git workflow (branches, commits, PRs) — DVC and GitHub Actions content builds directly on top without re-explaining Git basics.
- **Assumption**: a working MLflow tracking server and DVC remote (S3/GCS/Azure/SSH) are available/setup — the lecture shows client-side commands (`mlflow.start_run`, `dvc push`) but does not explain tracking-server deployment/infrastructure setup itself (deferred to the Lab #21 assignment, p.40, which explicitly tasks students with "Setup MLflow tracking server local (SQLite backend)").
- **Boundary/constraint (explicit, p.23)**: Eval Gate threshold given as a single example default (2% accuracy drop) — the lecture doesn't specify how this threshold should be chosen/tuned for different problem types, only that "threshold khác nhau theo metric" (different thresholds per metric) as a best practice, without concrete guidance on setting them.
- **Gap — GitOps/ArgoCD/Flux** (p.29): mentioned only as a single labeled callout box ("GitOps cho AI với ArgoCD/Flux: Declarative deployment...") with no further explanation of mechanism, setup, or how it integrates with the rest of the pipeline shown. Flagged — concept named but not sufficiently explained in-source.
- **Gap — Testing Pyramid inter-layer dependencies**: the pyramid diagram (p.31) shows five layers and tools per layer, but the lecture does not explicitly state execution order/dependency logic between layers (e.g., must unit tests pass before model tests run) — only the CI pipeline diagram (p.20) shows explicit sequencing for Data Validation → Training → Eval Gate → Deploy. Testing pyramid ordering beyond this is inferred from typical bottom-up structure, not explicitly stated as a rule.
- **Gap — OIDC federation mechanics**: mentioned by name (p.21, p.24) as the production-recommended alternative to long-lived AWS keys, but no explanation of how OIDC federation is configured/works is given.
- **Gap — statistical significance details**: A/B testing slide (p.36) references chi-squared test, p-value <0.05, and "Bonferroni" correction for multiple comparisons by name only, without explaining the underlying statistical reasoning or when Bonferroni correction should be applied.
- **Edge/failure case (explicit, p.23)**: borderline eval gate pass (delta close to but not below threshold) — best practice says require manual review rather than auto-deploy, but no concrete rule for what counts as "borderline" is given.
- **Edge/failure case (explicit, p.27)**: canary rollback trigger — P99 latency exceeds threshold OR accuracy drop >2% → auto rollback to v1. No detail on how quickly rollback executes or how "P99 latency threshold" is set.
- **Scope note**: this lecture is Day 21 of an "AI Infrastructure" specialization track (per course structure); it explicitly previews Day 22 as LLMOps & Prompt Versioning (LangSmith, W&B Weave, prompt versioning in CI/CD, LLM regression testing) — i.e., LLM-specific CI/CD concerns are explicitly deferred to the next lecture, not covered here beyond the brief MLflow LLM Autolog mention (p.9).

## 5. Learning Priorities

**Essential**
- CI/CD for AI vs traditional software — the 7-row comparison table (p.19), especially: non-deterministic deploys require an eval gate; pipeline can be triggered by data changes too.
- Eval Gate mechanism and rationale — fixed held-out test set, delta-vs-threshold logic, blocks deploy on accuracy drop (p.23); explicitly called the most important safety net (p.23, p.39).
- MLflow Tracking core loggable entities (params, metrics, artifacts, signature) and the Model Registry lifecycle (None→Staging→Production→Archived) (p.8, p.10).
- DVC's role and mechanism: pointer files, content-addressable storage, `dvc.yaml` pipeline stages, `dvc repro` (p.14–16) — and that DVC + MLflow are complementary, not interchangeable (p.17).
- The 4 deployment strategies (Canary, Blue/Green, Shadow, Rolling) and their trade-offs (p.26–28), especially Canary's staged rollout + rollback trigger (p.27) since it's used in the live demo (p.38).
- CI pipeline architecture: Data Validation → Model Training → Eval Gate → Deploy, with fail-fast and path-filter mechanisms (p.20).

**Important**
- Great Expectations data quality checks (schema, null rate, distribution drift, volume, freshness) (p.22, p.33).
- GitHub Actions workflow structure: secrets management, dependency caching, job dependencies (`needs:`), OIDC federation (p.21, p.24).
- AI Testing Pyramid layers and tools (Unit/Integration/Model/Data/Load, with pytest/Great Expectations/k6-Locust) (p.31–33).
- Model Registry best practices: naming convention, tag strategy, approval workflow, rollback plan (p.11).
- A/B testing mechanism: deterministic hashing, minimum sample size, significance threshold, primary vs guardrail metrics (p.36).
- Multi-environment CD pipeline (Dev/Staging/Production) and what differs at each stage (p.29).

**Supporting**
- MLflow LLM Autolog specifics (p.9) — likely to be revisited in more depth on Day 22 (LLMOps).
- DVC remote storage configuration details/CLI flags for S3/GCS/Azure/SSH (p.15).
- MLflow Model Serving deployment target options (Local/Docker/Cloud/Kubernetes) (p.35) — operational detail, not conceptual core.
- GitOps/ArgoCD/Flux mention (p.29) — named only, insufficiently explained (see Gaps).
- Specific code syntax examples (Python/YAML snippets) — useful as reference but the underlying mechanism/concept is the priority, not memorizing exact syntax.
