# Core Knowledge

Source: `Day10 data pipeline observability E402.pdf` (39 PDF pages / 28 numbered slides — PDF contains progressive-reveal duplicate pages per slide; all distinct content captured). Language: Vietnamese slides with English technical terms (kept as-is below). Reading method: text extraction (pdftotext -layout); no garbled/empty pages encountered, no vision fallback needed.

## 1. Core Concepts

- **Data Pipeline** — Chuỗi các bước tự động hóa việc thu thập, xử lý, và phân phối data từ nguồn đến đích. Typical AI data stack: `Sources → Pipeline (ingest+transform) → Storage (warehouse, vector store) → Serving (API, cache) → Agent (LLM+tools+RAG)`. Matters because 60–80% of real AI project time is data work, not modeling (slide 2, citing project reality: 20% model/agent, 80% data collection/cleaning/pipeline/monitoring). Explicit. (p.3)

- **Garbage in → garbage out** — Core framing concept: output quality is directly proportional to input data quality; a strong RAG agent still hallucinates if the vector store is loaded with dirty data. (p.2)

- **Data Observability** — Cơ chế phát hiện data sai trước khi user phàn nàn (mechanism to detect bad data before users complain). Distinguishes AI pipelines from BI: BI sai → wrong number on a report; Agent sai → wrong action or wrong answer given directly to the user. (p.2, p.4, p.18–21)

- **ETL (Extract → Transform → Load)** — Transform happens before loading into the store. Fits: sensitive data, data requiring masking before storage (e.g., redact PII in support tickets before embedding for an agent). Tools: Talend, Informatica, custom scripts. (p.5)

- **ELT (Extract → Load → Transform)** — Load raw data first, transform later inside the warehouse/lake. Fits: big data, cloud data warehouses. Tools: Spark SQL, BigQuery, custom Python jobs. (p.5)

- **Batch Processing** — Process on a schedule (hourly/daily). Pros: simple, low cost, easy debug. Cons: high latency (data delayed hours). Used for: training data, daily reports, ETL. (p.8)

- **Streaming Processing** — Process realtime as data appears. Pros: low latency (ms–seconds). Cons: more complex, higher cost. Used for: fraud detection, live agent context. (p.8)

- **CDC (Change Data Capture)** — Detects and captures every INSERT/UPDATE/DELETE in a database to sync in realtime instead of full scans. (p.9)

- **Incremental sync** — Only pull the changed portion since the last run (part of good ingestion design). (p.10)

- **Idempotent upsert** — Re-running ingestion must not create duplicate chunks. (p.10)

- **Source versioning** — Knowing which version of a source is newest and when it was synced. (p.10)

- **Rate limiting** — Source API limits requests/min; requires exponential backoff. (p.10)

- **Backpressure** — Consumer processes slower than producer produces → needs buffer or pause signal. (p.10)

- **Dead-letter queue (DLQ)** — Holds failed records (from retry logic) instead of dropping them, so they can be handled later. (p.10, p.25)

- **Chunking** — Splitting a document into pieces that are semantically coherent and fit token budgets; central to AI transform (unlike BI transform). Chunk too large → contains multiple topics, retrieval becomes vague, wastes tokens, reduces room for reasoning. Chunk too small → loses important context, answers miss conditions/exceptions. (p.13–14)

- **Metadata enrichment** — Attaching source, owner, version, effective date to chunks. Good metadata fields: content, chunk_id, source_doc_id, section/title, effective_date, owner/department, version/updated_at. Enables agent to filter by department/date/doc type, show correct citations, trace back to the source document. Common failure: teams embed "pure text" only and forget metadata → correct passage retrieved but agent can't tell which policy version it came from. (p.14)

- **Redaction** — Removing PII/secrets before embedding. (p.13)

- **Canonicalization** — Standardizing product names, order codes, timestamps. (p.13)

- **Schema validation / data contracts** — Enforce data contracts: reject records that don't match schema instead of letting them reach the model. (p.12)

- **6 Dimensions of Data Quality** — Completeness, Accuracy, Consistency, Timeliness, Validity, Uniqueness (full definitions in Relationships & Mechanisms / Examples section below). (p.15)

- **Quality Gates** — Checks that must pass before data reaches the agent: Schema gate, Freshness gate, Content gate, Dedup gate, PII gate. In AI pipelines, data quality protects not just the warehouse but also retrieval, tool use, and the final answer. (p.16)

- **5 Pillars of Data Observability** — Freshness, Distribution, Volume, Schema, Lineage. (p.18)

- **Data Lineage** — Tracks data's journey from source origin → pipeline → transform → chunk/index → retrieved context → model output. (p.18)

- **Trace log** — Minimum fields needed to debug: question/session ID, retrieved chunk IDs, source document version, embedding/index version, pipeline run ID (p.18); expanded trace fields: request_id, pipeline_run_id, retrieved_chunk_ids, source_doc_ids, source_version, embedding_model, latency_ms, fallback_used (p.20).

- **DAG (Directed Acyclic Graph)** — Defines task execution order in orchestration (Airflow core concept). (p.22)

- **Apache Airflow** — DAG-based orchestration. Core concepts: DAG, Operator (execution unit: PythonOperator, BashOperator...), Scheduler (triggers DAGs by cron or event), Executor (runs tasks: Local, Celery, Kubernetes), XCom (passes small data between tasks). Used when: batch pipeline is complex with many dependencies, team has Python skills, need full UI visibility. (p.22)

- **Prefect** — Python-native orchestration alternative, less boilerplate than Airflow; flows = Python functions. Fits teams wanting speed. (p.22–23)

- **Dagster** — Asset-centric orchestration — models data assets, not tasks. Built-in lineage & observability. Fits data-heavy teams. (p.22–23)

- **Idempotency** — Mandatory property: running a pipeline twice must produce the same result as running it once. Lack of idempotency causes duplicate data in the vector store. (p.25)

## 2. Relationships & Mechanisms

- **AI Data Stack flow**: `Sources (DB, API, files, streams) → Pipeline (ingest+transform) → Storage (warehouse, vector store) → Serving (API, cache layer) → Agent (LLM+tools+RAG)`. Each stage is a prerequisite for the next; a failure early propagates to agent output. (p.3)

- **RAG-agent pipeline example (Input → Process → Output)**: `Docs (Notion/PDF) → Ingest (sync/OCR) → Transform (clean/chunk) → Index (embed/store) → Retrieve (top-k) → Agent (answer)`. Why each step matters:
  - Ingestion fails → new documents don't enter the store → agent answers with stale info.
  - Transform is wrong → bad chunks, missing metadata → wrong retrieval.
  - Index has errors → embeddings missing or duplicated → distorted context.
  This differs from a BI pipeline where a failure just produces a wrong number on a report; here it directly produces a wrong action/answer to the user. (p.4)

- **ETL vs ELT (trade-off relationship)**: ETL transforms before load (better for sensitive data needing masking, stable schema, need for very clean data reaching the agent, reducing risk of storing raw sensitive data). ELT loads raw first, transforms later (better for many sources/formats, frequent backfill/replay needs, still-evolving chunking/labeling/feature engineering, need to keep raw for audit/experiment). Hybrid is common in practice: load raw first, but ETL sensitive parts (PII, secrets, legal data) before indexing/serving to the agent. (p.5–7)

- **Ingestion → Agent quality dependency**: if ingestion misses a source (e.g., missing CRM, policy docs, ticket history, or escalation notes), the agent can answer with outdated policy, miss business exceptions, or propose actions inconsistent with actual order status. Ingestion checklist for AI: (1) correct source pulled? (2) latest version pulled? (3) do you know which records failed? (4) is run ID and sync time logged? (p.11)

- **Data quality issue → Agent symptom mapping** (mechanism connecting root cause to observable failure):
  - Missing documents → agent can't find relevant evidence.
  - Outdated version → agent answers based on old policy.
  - Duplicate chunks → agent repeats the same point multiple times.
  - Wrong metadata → agent cites wrong department/wrong effective date.
  - Secret leakage → agent exposes sensitive data to the user.
  Teaching point: many errors that look like "model hallucination" actually have a data pipeline bug as root cause. (p.17)

- **Debug process (5-layer trace, Output → Source)**: Output layer (what did the agent answer, cite, confidence?) → Retrieval layer (which top-k chunks retrieved? zero-hit?) → Index layer (which embedding model/version produced this chunk?) → Pipeline layer (which run produced this chunk? which quality gates passed/failed?) → Source layer (is the original document correct, current, complete?). Warning: if you can't trace from the final answer back to the chunk and source document, "you are debugging in the dark." (p.20)

- **Observability worked example (mechanism)**: User complains agent gave outdated info → check answer trace (which chunk retrieved?) → check Freshness (which policy version/date does that chunk belong to?) → check Volume (did today's embedded document count drop to 0?) → trace Lineage (2am ingestion run failed at policy-sync step) → root cause (API timeout, retry/backoff misconfigured). Without observability: issue discovered 8 hours later, 500 users affected. (p.19)

- **Pipeline/data signals vs Agent/product signals must be connected**: Pipeline signals — freshness of knowledge base, failed sync count, DLQ size, duplicate chunk rate, missing metadata rate, embedding queue lag. Agent/product signals — retrieval hit rate, % answers with valid citation, user correction/escalation rate, tool-call failure rate, abandoned conversations after a wrong answer. Practical view: good observability must connect data issue → retrieval issue → business impact. (p.21)

- **Orchestration example pipeline (Input → Process → Output) for RAG/agent**: `Sync docs/API → Quality gate → Chunk+metadata → Embed → Upsert vector store → Smoke test retrieval → Notify/alert`. Practices: trigger hourly/on new file/on policy change; fail fast (quality gate failure blocks further indexing); smoke test (run standard questions to check retrieval); notify Slack if new index drops hit rate. In AI pipelines, orchestration isn't just "run jobs" — it also controls input quality before the agent uses new data. (p.24)

- **Error handling patterns (mechanism)**: Retry with backoff (attempt 1 → 30s → attempt 2 → 2m → ...) → Dead Letter Queue (failed records preserved, handled later) → Partial failure handled via idempotent tasks (safe to re-run) → Alerting (Slack/email on failure) → SLA breach alerting (pipeline running late vs deadline). (p.25)

- **Scheduling strategies**: Cron-based (e.g. `0 2 * * *` = 2am daily — simple, predictable), Event-driven (trigger on file upload/webhook), Dependency-based (run only after upstream pipeline finishes), Backfill (rerun for historical dates). (p.25)

- **Orchestration tool choice by team maturity (relationship/progression)**: Small RAG/agent systems typically start with cron + Python; as steps, sources, and teams grow, they move up to Airflow / Prefect / Dagster. Tool fit: Airflow → mature batch ETL, scheduled retraining, multi-step jobs, needs full UI. Prefect → Python pipelines, startup teams needing fast local-to-cloud flows. Dagster → asset-heavy pipelines, clear lineage needs, data platform teams. (p.23)

## 3. Examples & Distinctions

- **ETL vs ELT: similarity/difference/criterion** — Similarity: both move data from source to storage with a transform step. Difference: ETL transforms before loading (cleaner, more controlled, but slower to onboard new sources); ELT loads raw then transforms later (flexible, preserves raw for replay/audit, but risks storing unmasked sensitive data). Distinguishing criterion: whether sensitive/PII data needs masking before storage, and whether schema/transform logic is stable vs. still evolving. (p.5–7, explicit)

- **Batch vs Streaming: similarity/difference/criterion** — Similarity: both are data processing modes within a pipeline. Difference: batch runs on a schedule with higher latency and lower cost/complexity; streaming runs continuously with low latency but higher cost/complexity. Criterion: does the use case need realtime response (fraud detection, live agent context) or can it tolerate delay (training data, daily reports)? (p.8, explicit)

- **BI transform vs AI/RAG transform** — Similarity: both clean and restructure raw data. Difference: BI transforms data to produce accurate reports; AI transforms data so the model understands context correctly and retrieves the right evidence (chunking, metadata, embeddings specific to AI). (p.13, explicit)

- **Structured vs Unstructured vs Event-stream sources (example set)** — Structured: PostgreSQL/MySQL (via CDC), Snowflake/BigQuery, REST/GraphQL APIs (rate limits). Unstructured: CSV/JSON/Parquet/PDF/Word files, S3/GCS/Azure Blob, web scraping (HTML→text). Event streams: Kafka/Kinesis, Webhooks, IoT sensors (time-series). (p.9, explicit)

- **Worked ingestion example — internal CSKH (customer support) agent** — User asks "what is the latest refund policy?"; agent needs CRM (order/transaction status), policy docs (monthly refund policy), ticket history (similar past cases), escalation notes (when to hand off to a human). Illustrates multi-source ingestion dependency. (p.11, explicit)

- **Worked transform code example** — Python snippet: `load_pdf → clean_text → chunk(text, size=500, overlap=80) → write_record({chunk_id, content, source_doc, version, department})`. Illustrates chunking + metadata enrichment together in practice. (p.13, explicit)

- **Worked quality-gate code example** — Python `validate(record)` asserting: content non-empty, `updated_at >= cutoff_date`, content length ≥ 80, no secret content, chunk_id not already seen (dedup). Illustrates schema/freshness/content/PII/dedup gates as executable checks. (p.16, explicit)

## 4. Assumptions, Boundaries & Gaps

- **Assumption underlying the whole lecture**: agent quality is bounded by data pipeline quality ("garbage in → garbage out"); this is asserted as the throughline but not derived from a cited study on the slide itself beyond the general framing (the Sculley et al. reference in Tài Liệu Tham Khảo is cited as explaining "why 80% of AI time is data work," suggesting this framing is grounded in that source). (p.2, p.28 — explicit citation, content not summarized on slide)

- **Boundary/limitation — ETL vs ELT choice is not absolute**: slide explicitly notes "many teams use hybrid" (load raw first, but ETL sensitive parts before indexing) — so the ETL/ELT dichotomy is a simplification with real-world blending. (p.7, explicit)

- **Gap — chunk size guidance is qualitative only**: slides state chunk-too-big and chunk-too-small problems but do not give a concrete rule/formula for correct chunk size beyond the code example's `size=500, overlap=80` (which is illustrative, not stated as a general rule). Flagged, not filled from external knowledge. (p.13–14)

- **Gap — "quality-kém → agent-sai" mapping is stated as typical symptom, not exhaustive causal proof**: the table linking data issues to agent symptoms is presented as common patterns ("Điểm dạy học quan trọng: nhiều lỗi nhìn giống hallucination nhưng gốc rễ là data pipeline bug") — it does not claim all hallucination is a data bug, but the slide doesn't specify how to distinguish genuine model hallucination from data-pipeline-caused hallucination beyond the 5-layer trace process. (p.17, p.20)

- **Gap — 5 Pillars of Observability lack agent-specific detail on slide 18 itself**: Freshness/Distribution/Volume/Schema are given one-line definitions; Lineage is elaborated with required log fields, but Distribution and Schema pillars are not explained with AI-specific mechanics (e.g., how "distribution" applies to embeddings specifically) beyond the general definition. Flagged as underexplained. (p.18)

- **Gap — Orchestration tool comparison omits explicit criteria for choosing between Prefect and Dagster beyond team type/asset-focus**: reasons given are brief ("ít boilerplate," "asset model hợp với tables, features, indexes") without deeper trade-off discussion (e.g., scaling limits, cost). (p.22–23)

- **Prerequisite (implicit/inferred)**: understanding this lecture assumes familiarity with basic RAG concepts (embeddings, vector store, retrieval, top-k) since these terms are used without redefinition (e.g., "embed/store", "top-k", "retrieval"). Inferred, not explicitly stated as a prerequisite on any slide.

- **Boundary — Lab scope**: Lab #10 explicitly limited to building pipeline (raw→cleaned→chunked→embedded), quality gates (schema/freshness/duplicates), trace log, and before/after comparison after simulated data corruption — 4 hours total (1.5h Vibe Coding + 2.5h Lab). Explicit. (p.26)

- **Forward pointer / boundary of this lecture's scope**: next lecture is "Guardrails & AI Safety" — explicitly stated that safety/guardrails ("an agent working correctly does not mean it's safe") is out of scope for this lecture and will be covered next, including OWASP Top 10 for LLMs and poisoned-data-attack risk as a thinking prompt. Explicit. (p.28)

## 5. Learning Priorities

**Essential**
- Data Pipeline definition and the AI data stack flow (Sources → Pipeline → Storage → Serving → Agent).
- Garbage in → garbage out framing and why pipeline failures manifest as agent-level failures (not just report errors).
- ETL vs ELT distinction and when AI/ML teams pick each (including the common hybrid pattern).
- The 6 Dimensions of Data Quality (Completeness, Accuracy, Consistency, Timeliness, Validity, Uniqueness).
- Quality Gates before data reaches the agent (Schema, Freshness, Content, Dedup, PII).
- Data issue → Agent symptom mapping, and the insight that apparent "hallucination" is often a data pipeline bug.
- 5 Pillars of Data Observability (Freshness, Distribution, Volume, Schema, Lineage) and the 5-layer debug trace process (Output → Retrieval → Index → Pipeline → Source).
- Chunking and metadata enrichment: why chunk size matters and why metadata (source, version, department, effective date) is required for agent trust/citation/traceability.

**Important**
- Batch vs Streaming trade-offs and when each is used.
- Ingestion mechanisms: CDC, incremental sync, idempotent upsert, rate limiting, backpressure, dead-letter queue, source versioning.
- Trace log required fields (request_id, pipeline_run_id, retrieved_chunk_ids, source_doc_ids, source_version, embedding_model, latency_ms, fallback_used).
- Pipeline/data signals vs Agent/product signals and the need to connect data issue → retrieval issue → business impact.
- Orchestration fundamentals: DAG, Operator, Scheduler, Executor, XCom (Airflow); Prefect and Dagster as alternatives; tool choice by team maturity/data-asset focus.
- Error handling patterns (retry+backoff, DLQ, idempotent partial-failure handling, alerting, SLA breach) and scheduling strategies (cron, event-driven, dependency-based, backfill).
- Idempotency as a mandatory property to avoid duplicate vector-store data.

**Supporting**
- Specific tool names (Talend, Informatica, Spark SQL, BigQuery) as ETL/ELT examples.
- Structured/unstructured/event-stream source taxonomy (specific product examples like Kafka, Kinesis, S3, GCS).
- Text normalization specifics (lowercasing, Unicode NFC/NFD, whitespace collapsing, HTML stripping, language detection).
- Worked code snippets (transform pipeline script, quality-gate validate() function) — useful as illustration, not conceptually new.
- Reference readings (Sculley et al. 2015; Kleppmann's *Designing Data-Intensive Applications*; Chip Huyen's *Designing Machine Learning Systems*).
- Lab #10 logistics (deliverables, time allocation).
