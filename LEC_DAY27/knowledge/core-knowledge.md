# Core Knowledge

Source: "stream (1).pdf" — "Data Observability & Lineage cho hệ thống Data/AI" (40 slides). Language: Vietnamese with English technical terms (kept as in source).

## 1. Core Concepts

- **Pipeline Monitoring** (p.4) — Observes infrastructure/process health: job status, duration, retry count, CPU, memory, worker failure, network timeout. Answers "does the machine run?" Explicit: necessary but not sufficient ("Cần thiết nhưng chưa đủ").
- **Data Observability** (p.4) — Observes data health: freshness, volume, distribution, schema, lineage, quality checks, downstream impact. Answers "is the data trustworthy?" Protects trust of dashboards, ML models, RAG, AI agents. Contrasts directly with monitoring.
- **Data Reliability Operating System** (p.5) — A layered operating model, explicitly stated as not depending on a single tool. Chain: Contract → Tests → Anomaly → Lineage → SLO → Incident (see Relationships section for detail).
- **5 Pillars of Data Observability** (p.6, attributed to Monte Carlo per p.38 references) — Freshness, Volume, Distribution, Schema, Lineage; plus a 6th practical addition on this slide: **Actionability** ("quan sát xong làm gì?" — alert, owner, runbook, suppress publish, rerun, rollback).
  - **Freshness** (p.7): is data new enough? Measured as `freshness_minutes = now() - max(updated_at)`. Should be measured at the serving point (serving table / vector index), not just raw ingest. Uses watermarks (max(event_time), max(updated_at), last_publish_time). Alerted against business SLA (e.g., CEO dashboard before 8:00, RAG index within 30 minutes).
  - **Volume** (p.8): is data missing or excessive? Detects partial ingestion, duplicate events, API rate-limits, lost partitions. Metric: row_count by day/partition/source. Baseline: compare same weekday (Monday vs Monday), not mixed with weekends. Alert threshold example: drop > 50%; only alert when business-meaningful. Action: quarantine — don't publish serving table if volume check critically fails.
  - **Distribution** (p.9): does data behave differently than usual today? Checked per data type — Numeric (mean, median, std, p95, min/max, outlier rate), Categorical (top-k ratio, unseen category, entropy), Text (token length, language ratio, empty/duplicate chunks), Embedding (cosine similarity to baseline, cluster drift). Explicit caveat: distribution drift is not always an error — could be campaign, seasonality, or genuine data source change.
  - **Schema** (p.10): breaking changes often come without warning. Includes required columns, type constraints, enum contracts, and backward-compatibility rules (adding a column is usually safe; renaming/removing is usually breaking).
  - **Lineage** (p.11): when incident occurs, what gets dragged along? Helps trace root cause and blast radius. Example chain: raw.orders (source API) → stg_orders (clean+normalize) → fct_revenue (aggregation) → CEO dashboard (business exposure).
- **Data Contract** (p.12) — "Shift-left data quality": detect AND block bad data before it goes downstream. Contract = schema + semantics + quality + owner + SLA. Producer knows what to supply; consumer knows what to expect; pipeline can fail early instead of publishing bad data. Example YAML includes dataset name, owner, freshness (max_delay_minutes), column types/required/unique, checks (e.g., min:0), accepted_values.
- **Rules (deterministic checks)** vs **Anomaly Detection (statistical checks)** (p.13) — Two layers of defense in production. Rules: known problems, deterministic, hard fail, business contract (e.g., not null, no duplicate, valid status, amount >= 0, required columns present). Anomaly: unknown unknowns, statistical, soft alert, human review (e.g., row count drops 70%, null rate gradually rising, embedding drift, token length skew).
- **Great Expectations (GX) workflow** (p.14) — Modern workflow chain: Data Asset (batch to validate) → Expectation Suite (meaningful rules) → Validation Definition (asset + suite) → Checkpoint (run + actions) → Result (pass/fail/report/alert).
- **Expectation Suite design categories** (p.15) — Completeness (critical columns not null), Uniqueness (primary key not duplicated), Validity (valid value ranges/enums), Consistency (cross-field/cross-table, e.g. sum(detail)=total, FK exists), Freshness-like (data not too old), Reject/Quarantine (don't publish bad data; critical fail stops downstream, warning goes to review).
- **Severity tiers: critical / warning / info** (p.16) — Not every error should block the pipeline. Critical: data causes direct harm downstream → fail checkpoint, stop publish, page owner (e.g., missing required column, duplicate PK). Warning: needs investigation but can continue → Slack alert, ticket, monitor trend (e.g., null rate slightly up, low OCR confidence). Info: metadata/signal to track → log, Data Docs, dashboard (e.g., new optional column, new category).
- **dbt (transformation layer protection)** (p.17) — Business logic lives in SQL; a wrong join can still "succeed" (silent transform bug, e.g., revenue inflated by wrong JOIN key with no crash/no null). dbt models sit closest to SQL transformation. Data tests check data after model materializes; unit tests check SQL logic against small mock inputs; exposures/lineage show which model feeds which dashboard/model.
- **dbt test taxonomy** (p.18) — Four distinct test types (see Examples & Distinctions for the comparison table).
- **Z-score anomaly detection** (p.20) — `z = abs(current - mean(history)) / std(history); if z > 3: alert()`. Explicit: "a starting point, not the destination." Suitable for: stable metrics, near-normal distribution, sufficiently long history. Not suitable for: strong seasonality, many historical outliers, campaigns/flash sales. Output requires human review — anomaly alone doesn't prove incident. Good metrics: row_count, null_rate, freshness_lag, embedding_drift.
- **Baseline strategies** (p.21) — Rolling mean (stable, simple, use 14-30 day window if no strong seasonality), Same-weekday (accounts for weekly seasonality), Median + MAD (robust to outliers, less skewed by one flash-sale/outage day), Segment baseline (split by region/plan/source/product line for multiple behavior groups), Human review (alerts need context + runbook, not just numbers), Suppress policy (annotate known events like campaigns to avoid alert spam).
- **Observability for AI/unstructured data** (p.22) — Don't monitor raw text/image directly; extract measurable features. Text corpus (token length, language ratio → detect source format change, parser errors, empty content). Image corpus (blur score, resolution, OCR confidence → detect bad scans, wrong upload size). Embedding (cosine drift, norm distribution → detect model embedding change or corpus behavior shift). RAG retrieval (hit rate, chunk count, answer length → detect stale index or chunker breaking context). Prompt/tool logs (tool success rate, citation rate → detect agent misusing tools or lacking grounding). Eval samples (golden questions → detect quality drop even when technical metrics look fine).
- **RAG/AI Lineage** (p.23) — Traces from document to answer so when an answer is wrong, you know which data/model version produced it. Chain: Document (source_uri + version) → Parser (parser_version) → Chunker (chunk_config) → Embedding (model + dims) → Index (index_version) → Answer (citations + run_id).
- **Lineage levels** (p.24) — Table/asset, Column, Run, Ownership (see table in Relationships section).
- **OpenLineage** (p.25) — A standard for pipeline operational metadata so multiple tools can "speak the same language" (event-based lineage). Core entities: Run, Job, Dataset. Input/output datasets describe what's read/written. Facets extend metadata (schema, columnLineage, dataQualityMetrics). Collectors (Airflow, Spark, dbt, custom jobs) emit events. Example event fields: eventType (e.g. COMPLETE), job namespace/name, run.runId, inputs, outputs, facets.
- **SLI / SLO / Error Budget** (p.26) — Reliability needs numbers and policy, not just a feeling of "seems fine." SLI examples: freshness_minutes, null_rate, p99_latency. SLO example: freshness < 60 min in 99.5% of checks. Error Budget = 1 - SLO (portion of allowed failure). Policy: when budget is burned, stop releases and fix reliability.
- **Burn Rate** (p.27) — Normalized speed of consuming the error budget. `allowed_bad_rate = 1 - SLO`; `actual_bad_rate = bad_events/total_events`; `burn_rate = actual_bad_rate / allowed_bad_rate`. burn_rate=1 means spending budget at exactly the allowed pace; burn_rate=4 means budget burns 4x faster than allowed. Fast window detects outages quickly; slow window filters noise and avoids paging for short spikes.
- **Multi-window alerting** (p.28) — Combines short and long windows to reduce false positives (see table in Examples & Distinctions / Relationships).
- **SLO Dashboard components** (p.29) — Current SLI, Target SLO, Error Budget (remaining %, consumed, days left), Burn Rate (short+long windows), Impact (who's affected — dashboard/model/agent/users), Owner/Runbook (owner, escalation, diagnostic links).
- **Data incident lifecycle** (p.30) — 6 stages: Detect → Triage → Mitigate → RCA → Verify → Learn. Explicit: "Detection is only round 1; operations is where value is created."
- **Severity levels for incidents (P0-P3)** (p.31) — See table in Examples & Distinctions. Explicit statement: "bad data can be worse than service downtime because it's silent."
- **Runbook** (p.32) — 7-step template: 1) Confirm (check alert, validation result, run_id, start time), 2) Classify (P0-P3 by downstream impact), 3) Check upstream (source status, freshness, volume, schema changes, retry logs), 4) Read lineage (upstream root cause + downstream blast radius), 5) Mitigate (suppress publish, rerun from checkpoint, rollback version, notify users), 6) Verify (expectations pass, anomaly returns to baseline, SLO stable), 7) Postmortem (timeline, 5 Whys, action items with owner/deadline).
- **Blameless postmortem & Game day** (p.33) — Goal is not to find who's at fault but to fix the system so errors are harder to repeat. Components: Timeline (event sequence: alert, ack, decision, fix, verify), 5 Whys (find systemic cause), Action items (owner + deadline, not vague "improve monitoring"), Game day (inject controlled failure: schema drift, partial data, stale index, wrong join), Learning (update contract/runbook — every incident should strengthen the system), Accountability (blameless does not mean no accountability).
- **Meta-metrics for the observability system itself** (p.34) — MTTD (Mean Time To Detect: from error occurrence to correct alert), MTTA (Mean Time To Acknowledge: from alert to owner taking it), MTTR (Mean Time To Recover: from error to downstream safety restored), Alert precision (real incidents / all alerts — if too low, team gets alert fatigue and ignores alerts), False negative (incidents not alerted — learned from user reports and postmortems), Repeat rate (whether postmortems/action items actually fixed the systemic issue).
- **Tooling map** (p.35) — Organized by layer and maturity: Rules/validation (Great Expectations, Soda Core, custom SQL / SaaS: Soda Cloud, GX Cloud), Transformation tests (dbt Core, dbt-expectations, dbt-utils / dbt platform), Lineage (OpenLineage, Marquez, dbt docs / Monte Carlo, Databand, Atlan), SLO/metrics (Prometheus, Grafana, Streamlit / Datadog, Grafana Cloud, GCP/AWS monitoring). Explicit: "no single tool does everything."
- **Implementation roadmap (Level 0-4)** (p.36) — Level 0 (only find out when user reports) → Level 1 (rules + contracts) → Level 2 (anomaly + dashboards) → Level 3 (lineage + SLO + runbook) → Level 4 (game day + self-healing). Moving from reactive to reliable layer by layer.

## 2. Relationships & Mechanisms

- **Monitoring is prerequisite-but-insufficient for Observability**: monitoring necessary but not enough (p.4) — infra can report "success" while data is wrong (see p.3 case).
- **Data Reliability Operating System pipeline** (p.5, explicit Input→Process→Output chain):
  Contract (schema, owner, SLA, semantics) → Tests (GX, dbt, SQL assertions) → Anomaly (baseline, drift, seasonality) → Lineage (root cause, blast radius) → SLO (budget, burn rate, policy) → Incident (triage, mitigate, postmortem). Each stage feeds the next; contract prevents bad data upstream, tests catch known problems, anomaly catches unknowns, lineage locates impact, SLO turns quality into a measurable commitment, incident response operationalizes all of it.
- **Failure mode described in opening case** (p.3, explicit): Source schema/unit changed → Pipeline runs successfully (no red logs) → bad table / stale index → wrong answer shown to user. Demonstrates that "pipeline success" does not equal "data correct."
- **Rules and Anomaly Detection are complementary, not substitutes** (p.13): rules catch known/deterministic problems; anomaly detection catches unknown/statistical patterns. Both layers recommended together ("Production tốt nhất có hai lớp phòng thủ").
- **GX workflow chain** (p.14, explicit Input→Process→Output): Data Asset → Expectation Suite → Validation Definition → Checkpoint → Result.
- **Severity → Action mapping** (p.16): critical → fail checkpoint/stop publish/page owner; warning → Slack alert/ticket/monitor; info → log/dashboard. Same severity model reused for incident response (P0-P3, p.31) and quarantine logic (p.15).
- **dbt test types map to different layers**: unit tests validate SQL logic (mock data) before data exists in production; data tests (generic/singular) validate actual materialized tables; E2E validation checks the full pipeline output. This is a dependency chain — unit tests can run pre-deployment, data tests run post-materialization, E2E runs across the whole system.
- **Lineage chain example, general data** (p.11): raw.orders (source API) → stg_orders (clean+normalize) → fct_revenue (aggregation) → CEO dashboard (business exposure). This is the mechanism by which lineage supports blast-radius/root-cause analysis (p.11, p.24).
- **RAG/AI lineage chain** (p.23, explicit Input→Process→Output): Document (source_uri+version) → Parser (parser_version) → Chunker (chunk_config) → Embedding (model+dims) → Index (index_version) → Answer (citations+run_id). Purpose: when an answer is wrong, trace which data/model version produced it.
- **Lineage granularity levels increase RCA speed** (p.24): Table/asset (which source feeds this table?) → Column (which column does revenue derive from?) → Run (which run produced the faulty output?) → Ownership (who is responsible?). More detailed levels answer more specific questions and need more metadata (inputs/outputs/job name → inputFields/transformationDescription → run_id/timestamp/code version/parameters → owner/Slack channel/on-call/domain).
- **OpenLineage mechanism** (p.25): Collectors (Airflow, Spark, dbt, custom jobs) emit events describing Job/Run/Dataset with Facets (schema, columnLineage, dataQualityMetrics) — this standardization lets multiple tools interoperate.
- **SLI → SLO → Error Budget → Policy chain** (p.26, explicit): SLI (freshness_minutes, null_rate, p99_latency) → SLO (e.g. freshness < 60min in 99.5% of checks) → Error Budget (1 - SLO = allowed failure fraction) → Policy (when budget burns out: stop release, fix reliability).
- **Burn rate mechanism** feeds multi-window alerting (p.27→p.28): fast window (5m/30m/1d) catches rapid degradation; slow window (1h/6h/7d) filters noise. Combination reduces false positives while still catching real outages quickly.
- **Multi-window alert → action mapping** (p.28): Page (5m short / 1h long) → burning fast and sustained → P0/P1 on-call. Ticket (30m/6h) → clear degradation but not catastrophic → triage in shift. Report (1d/7d) → gradually worsening quality trend → reliability backlog.
- **Incident lifecycle mechanism** (p.30, explicit Input→Process→Output): Detect (alert/check fail) → Triage (severity + owner) → Mitigate (stop publish/rerun) → RCA (root cause + lineage) → Verify (checks green) → Learn (postmortem). Lineage is explicitly used inside the RCA step.
- **Postmortem feeds back into contracts/runbooks** (p.32): "mỗi incident phải làm hệ mạnh hơn" — each incident should update contract/runbook (Learning), closing the loop back to the Data Reliability Operating System's Contract stage.
- **Trade-off: alert sensitivity vs alert fatigue**: implicit across p.8 (only alert on business-meaningful thresholds), p.20-21 (baseline choice affects false positives), p.34 (alert precision — too low causes team to ignore alerts). This is a recurring mechanism/trade-off theme across the deck (explicit in multiple places, connected here as inferred synthesis).
- **Roadmap builds cumulatively** (p.36, inferred from Level 0→4 sequence): each level adds capability on top of the previous (rules+contracts, then anomaly+dashboards, then lineage+SLO+runbook, then game day+self-healing) — not stated as strictly sequential/mandatory but presented as an ordered progression.

## 3. Examples & Distinctions

- **Opening case example** (p.3, explicit): Airflow/cron reports success; dashboard revenue off by 40% because `price` column changed units with no contract; AI agent answers with old policy because vector index is stale. Used to illustrate "pipeline success != data correct" and that silent bad data is the most dangerous failure.
- **Pipeline Monitoring vs Data Observability** (p.4): Similarity — both are forms of "observing" a system. Difference — monitoring observes infrastructure/process (job status, duration, retries, CPU/memory, worker failure, network timeout); observability observes data health (freshness, volume, distribution, schema, lineage, quality, downstream impact). Distinguishing criterion: "does the machine run?" (monitoring) vs "is the data trustworthy?" (observability).
- **Rules vs Anomaly Detection** (p.13): Similarity — both are quality-detection mechanisms in production. Difference — Rules are deterministic/hard-fail/business-contract-based, targeting known problems (null, duplicate, invalid status, negative amount, missing required column). Anomaly is statistical/soft-alert/human-review-based, targeting unknown unknowns (row count -70%, gradually rising null rate, embedding drift, token length skew).
- **dbt test taxonomy — 4-way distinction** (p.18, table):
  | Type | Checks | Data used | Example |
  |---|---|---|---|
  | Unit test | SQL logic of one model | mock input + expected output | 3 orders → revenue correctly 170 |
  | Generic data test | data after model build | actual table | unique, not_null, accepted_values, relationships |
  | Singular data test | custom business assertion | actual table | no inactive user has an active subscription |
  | E2E validation | final output meets expectation | full pipeline | dashboard/model/RAG answer not off |
- **dbt example code** (p.19, explicit): schema.yml data_tests (unique, not_null, accepted_values on order_id/status) paired with a unit_tests block (revenue_logic: given mock rows amount=100+70 status=resolved, expect daily_revenue=170). Illustrates unit test vs data test side by side.
- **Baseline choice examples** (p.21): Rolling mean vs Same-weekday vs Median+MAD — similarity: all are ways to define "normal" for anomaly comparison; difference: rolling mean assumes no strong seasonality, same-weekday explicitly handles weekly seasonality, median+MAD is robust to single-day outliers (flash sale, outage).
- **Severity table for Expectation Suite actions** (p.16, table): critical (missing required column, duplicate PK) → fail checkpoint/stop publish/page owner; warning (null rate up slightly, low OCR confidence) → Slack alert/ticket/monitor trend; info (new optional column, new category) → log/Data Docs/dashboard.
- **Incident severity P0-P3** (p.31, table):
  | Level | Description | Response target | Example |
  |---|---|---|---|
  | P0 | no data serving a critical system | 5 min | fraud feature table not updating |
  | P1 | wrong data affecting user/business | 30 min | revenue dashboard wrong, agent gives wrong policy |
  | P2 | SLO breach or quality degradation | 2 hours | freshness slow, null rate rising |
  | P3 | minor issue / documentation | next business day | low-signal alert, missing owner |
- **OpenLineage event example** (p.25, explicit YAML/JSON-like snippet): eventType COMPLETE, job namespace analytics / name build_fct_revenue, run.runId timestamp, inputs [raw.orders], outputs [mart.fct_revenue], facets (schema, dataQualityMetrics).
- **Data contract YAML example** (p.12, explicit): dataset `orders`, owner `commerce-data`, freshness max_delay_minutes:30, columns (order_id: int/required/unique, amount: decimal/checks min:0, currency: string/accepted_values [USD,VND]).
- **Schema contract YAML example** (p.10, explicit): columns order_id/amount/currency/created_at with types and required flags; policy distinguishing missing_required:critical vs new_optional_column:info — illustrates the severity distinction applied to schema specifically.

## 4. Assumptions, Boundaries & Gaps

- **Instructor name placeholder** ("[Tên giảng viên]") appears on every slide — administrative, not content; flagged only for completeness, not extracted as knowledge.
- **Z-score method explicitly scoped** (p.20): only appropriate for stable metrics, near-normal distributions, and sufficiently long history; explicitly NOT appropriate under strong seasonality, many historical outliers, or campaign/flash-sale periods. The deck does not describe what method to use instead beyond suggesting better baselines (rolling/same-weekday/median-MAD, p.21) — no deeper anomaly algorithm (e.g. ML-based) is covered.
- **Distribution drift explicitly ambiguous** (p.9): "Distribution drift không luôn là lỗi" — the deck does not give a concrete decision procedure for distinguishing real drift from legitimate business change (campaign, seasonality) beyond noting the ambiguity and pointing to human review / suppress policy as mitigations.
- **Anomaly output requires human review** (p.20): explicitly stated that anomaly alone doesn't prove an incident — deck does not detail the review process/criteria.
- **Actionability pillar** (p.6) is presented as part of "5 pillars" framing (title says "5 trụ cột") but 6 pillars are actually listed (Freshness, Volume, Distribution, Schema, Lineage, Actionability) — slide 6 title and content count are slightly inconsistent; source doesn't resolve this explicitly (flag: possible naming/count inconsistency, not filled in from external knowledge).
- **"Actionability" pillar itself is thin** (p.6): only bullet points (alert, owner, runbook, suppress publish, rerun, rollback) with no worked example or dedicated slide, unlike the other 5 pillars which get individual deep-dive slides (p.7-11). Flag: mentioned but not sufficiently explained.
- **RAG/AI observability section (p.22-23)** lists many feature categories (text, image, embedding, RAG retrieval, prompt/tool logs, eval samples) at a high level without worked numeric examples or thresholds — less depth than the structured-data pillars section. Flag as lighter-coverage area.
- **Burn rate windows (fast/slow)** (p.27) are described conceptually ("phát hiện outage nhanh" / "lọc noise") without concrete numeric window definitions on that slide; concrete windows only appear later in the multi-window alerting table (p.28) — reader must connect the two slides.
- **Tooling map (p.35)** is presented as illustrative categorization ("Không có tool nào làm hết mọi việc") — the deck does not evaluate or compare the listed tools' capabilities, only names them by layer/maturity tier.
- **Roadmap levels (p.36)** are not explicitly stated as strictly sequential or mandatory — labeled "Level 0-4" implying progression, but the deck doesn't state whether levels can be skipped or must be done in order (inferred progression, not fully explicit).
- **References section (p.38)** lists external sources the lecture is built on (Monte Carlo Data Observability, GX Core docs, dbt Developer Hub, OpenLineage spec, Google SRE Workbook, Soda documentation) — these are cited as background sources, not further explained in-deck; useful for context but content beyond what's on these 40 slides should not be assumed as covered.
- **Closing Q&A prompt (p.39)** is open-ended discussion ("Nếu chỉ được thêm một lớp bảo vệ... bạn chọn contract, test, anomaly, lineage hay SLO?") with no source-provided answer — explicitly a discussion prompt, not resolved in the deck.
- **Lab integration slide (p.37)** proposes specific tools (DuckDB+CSV, GX/custom checks, dbt-duckdb, pandas/numpy, dbt docs/manifest, Markdown runbook) for a follow-up lab, but no actual lab content/dataset is included in this deck — flagged as forward-pointer only, not covered material.

## 5. Learning Priorities

**Essential**
- Monitoring vs Data Observability distinction (p.4) — the central reframing of the whole lecture.
- "Pipeline success != data correct" (opening case, p.3; restated in Key Takeaways, p.37).
- 5(+1) pillars of data observability: Freshness, Volume, Distribution, Schema, Lineage, (Actionability) (p.6-11).
- Data Contracts — concept and shift-left rationale (p.12).
- Rules vs Anomaly Detection as complementary layers (p.13).
- Severity tiers (critical/warning/info) and their action mapping (p.16), reused in incident P0-P3 (p.31).
- Lineage concept and its role in RCA/blast radius (p.11, p.24).
- SLI/SLO/Error Budget/Policy chain (p.26) and Burn Rate (p.27).
- Data incident lifecycle: Detect→Triage→Mitigate→RCA→Verify→Learn (p.30) and the Runbook 7-step template (p.32).
- Key takeaways slide (p.37) — the instructor's own explicit summary of what to remember.

**Important**
- Great Expectations workflow chain (p.14) and Expectation Suite design categories (p.15).
- dbt test taxonomy (unit vs generic data vs singular data vs E2E) (p.18-19).
- Baseline strategies for anomaly detection (rolling, same-weekday, median+MAD, segment) (p.21).
- RAG/AI lineage chain and observability for unstructured data (p.22-23).
- OpenLineage standard (Run/Job/Dataset, facets, collectors) (p.25).
- Multi-window alerting (page/ticket/report tiers) (p.28).
- SLO Dashboard components (p.29).
- Blameless postmortem and game day practice (p.33).
- Meta-metrics: MTTD/MTTA/MTTR, alert precision, false negative, repeat rate (p.34).

**Supporting**
- Z-score formula details and its explicit scope limits (p.20).
- Specific YAML/config examples (contract, schema policy, OpenLineage event, dbt schema.yml/unit test snippets) — useful as illustration but not core concepts themselves (p.10, p.12, p.19, p.25).
- Tooling map by layer/maturity (p.35).
- Implementation roadmap Level 0-4 (p.36).
- Lab integration suggestions (p.37).
- References/sources list (p.38).
- Closing Q&A discussion prompt (p.39) — not source content to teach, but a discussion trigger.
