# Core Knowledge — Day 17: Data Pipeline Engineering (Chương 4: Hạ Tầng)

Source: `stream.pdf` (35 numbered content slides + title/Q&A, 45 pages total, AICB-P2T2 · Ngày 17 · Tuần 4). Language: Vietnamese slides with English technical terms; this extraction preserves original terms/quotes where they carry precise technical meaning. Running example threaded through the whole deck: an AI customer-support platform (Postgres tickets via CDC, S3 transcripts, Kafka click/feedback events → Bronze → Silver → three Gold tables feeding a RAG index, a classifier, and a routing agent).

---

## 1. Core Concepts

### Medallion architecture as commitment levels, not table names (explicit, p.5)
Bronze/Silver/Gold are three **levels of commitment about the data**, each stating what it promises and what it does not:
- **Bronze** — "nguyên bản" (original/raw): append-only, no UPDATE/DELETE, no cleanliness promise (by design, not negligence). Answers "what happened."
- **Silver** — dedup + enforced schema, "1 hàng = 1 thực thể" (1 row = 1 entity): has a key, correct types, PII handled. Answers "what is the current state."
- **Gold** — aggregated/feature tables, chunked for consumers, "đúng hình dạng cho đúng consumer" (right shape for the right consumer): answers "what does this specific consumer need." Storage format (Delta/Iceberg) is explicitly deferred to Day 18.

### Bronze layer (explicit, p.6–7)
Commitment: whoever reads Bronze can trust the data is byte-identical to what left the source system. Only INSERT, never UPDATE/DELETE. Examples of "how raw": a `priority` value `'HIGH '` with a trailing space is kept as-is; a ticket sent 3 times is stored 3 times; unknown fields are kept. Table carries lineage metadata columns: `_payload` (raw JSON), `_source`, `_op` (c/u/d), `_ingested_at`, `_batch_id`, `_kafka_offset` (for replay).
Why it matters — three real scenarios motivating immutability: (1) a Silver transform bug found 2 months later — if Bronze is intact, just delete Silver and rerun (~20 min); if Bronze was already "cleaned," you must re-fetch from the source, which may no longer hold historical state (Postgres only has today's state). (2) Retraining a model needs data *as it was* at that point in time — retraining on Bronze that's been "normalized" under this month's rules trains on a rewritten past. (3) Audit — 6 months later, "what data did the model use to reject this customer's request?" is only answerable from an unmodified Bronze. Analogy (explicit): "Bronze là cuốn sổ ghi chép, không phải bản báo cáo" (Bronze is a ledger, not a report — an erasable ledger has no evidentiary value).

### Silver layer (explicit, p.8–9)
Commitment: "một hàng = một thực thể" (one row = one entity). Requires deduplication across all change events for the same entity (e.g., a ticket updated 3 times in Bronze becomes 1 row in Silver reflecting final state) — without this step, downstream counts are wrong (e.g., "how many tickets this month" counts 3 instead of 1), and a model reading Bronze directly might wrongly infer "this ticket type matters 3x more" when it was just edited more often.
Key requirements: **has a key** (unique identifying column, not null, not duplicated) enabling MERGE/upsert (idempotent regardless of retry count); without a key, only INSERT is possible, so an Airflow retry duplicates rows silently (no error raised). **Correct types** — never silently coerce (e.g., upstream changes `priority` from number to string — pipeline runs fine, dashboard shows wrong numbers 3 weeks later before anyone notices). **PII handled** — Silver is the last boundary before data fans out irreversibly to Gold, feature store, vector DB, exported CSVs. Explicit anti-pattern: a "Silver" that's just `SELECT * FROM bronze WHERE user_id IS NOT NULL` commits to nothing extra — it's a redundant layer; removing it makes the pipeline faster and cheaper.

### Gold layer (explicit, p.10)
Commitment: correct shape for *one specific consumer*, usable immediately without further processing — never a single universal table. Three example Gold tables in the running scenario, each with its own definition of "correct" and its own failure mode:
| Table / Consumer | 1 row is | Critical commitment | If wrong |
|---|---|---|---|
| `gold_doc_chunks` / RAG index | one text chunk | stable chunking + pinned embedding model version | swapping embedding model without pinning mixes two vector coordinate systems → noisy results nobody can diagnose |
| `gold_training_set` / Classifier | one labeled ticket | immutable, versioned snapshot | a month later the model's score drops and you can't answer "what training set did it learn from" |
| `gold_feature_daily` / Routing agent | one user × one day | point-in-time correctness, no future leakage | a 12-08 feature accidentally uses a label generated on 15-08 → looks great in test, fails in production |

### CDC (Change Data Capture), log-based via Debezium (explicit, p.11)
Debezium reads the database's write-ahead log (WAL) directly (Postgres: logical replication on WAL; MySQL: binlog) rather than polling. Debezium impersonates a replica, so it adds almost no query load, captures INSERT/UPDATE/DELETE in correct order, and doesn't miss deleted records or intermediate states — unlike polling (`SELECT ... WHERE updated_at > ?` every 5 min), which misses deleted rows, misses intermediate states between polls, and adds load to the production DB. Cost of CDC: must enable a replication slot; an unread slot fills the source DB's disk.

### Data formats: JSON / Avro / Parquet / Arrow, each for a different stage (explicit, p.13)
| Format | Structure | Strength | Use at |
|---|---|---|---|
| JSON | text, no schema | human-readable, universally parseable | only at system edges; don't store long-term |
| Avro | row-oriented, has schema | fast writes, schema evolution is a first-class feature | on the wire — Kafka messages |
| Parquet | columnar, heavily compressed | reads only needed columns, pushes filter predicates to file level | at rest — every table in the lake |
| Arrow | columnar, in-memory | zero-copy exchange between engines | in RAM — DuckDB ↔ Polars ↔ Spark |
Why Arrow matters: reading Parquet in DuckDB then handing off to Polars costs zero conversion because both speak Arrow; before Arrow, every tool switch meant a re-serialization. Explicit warning: don't store Bronze as gzip-compressed JSON "for convenience" — six months later a single-column filter query must decompress and parse the entire history, and the bill reflects it.

### dbt: model layers, materialization, ref(), contract (explicit, p.14)
Three model layers: **staging** (1:1 with source, light cleaning) → **intermediate** (joins, business logic) → **marts** (final Gold tables). Materialization progression: `view` during development → `table` in production → `incremental` when the table is large. `ref()` auto-generates the dependency graph — dbt infers run order without manual declaration. **Contract** (`contract: {enforced: true}` in schema.yml) fails the build if a column's type is wrong, catching errors at build time rather than "someone notices the dashboard is wrong."

### dbt incremental models — three config lines (explicit, p.15)
- `unique_key` — missing it means data duplicates on every rerun.
- `incremental_strategy` — `merge` = upsert; `delete+insert` = overwrite whole partitions.
- `is_incremental()` macro — missing it means every run rescans the entire history.
Explicit warning: `--full-refresh` is only safe if Bronze still holds enough history; if Bronze has been purged by a TTL, a full-refresh silently rebuilds a table with missing data, with no warning.

### SQLMesh as dbt's alternative (explicit, p.16)
| | dbt | SQLMesh |
|---|---|---|
| SQL understanding | treats SQL as text + Jinja | parses into a tree → column-level lineage |
| On model edit | you guess the blast radius | auto-classifies breaking vs non-breaking, backfills only what's needed |
| Dev environment | usually rebuild the whole schema | virtual environment — creates views, near-zero compute |
| Ecosystem | very large, easy hiring | smaller, less documentation |
Choose dbt for a new team needing community/hiring ease; choose SQLMesh if burned by wrong backfills or expensive dev environments. Explicit note: what both share matters more than the difference — both force you to treat transforms as versioned, tested, reviewed code rather than a folder of shared SQL scripts.

### Idempotency — four techniques (explicit, p.17)
Definition (explicit): running once or N times produces the same final state. Needed because every real system is at-least-once (Airflow retries, someone clicking "Clear Task," overlapping backfills, consumer restarts).
| Technique | How | Fits when | Cost |
|---|---|---|---|
| Overwrite partition | DELETE day X then INSERT | clear time column | cheapest, simplest |
| MERGE/upsert | match by key, update if exists | records get revised later (CDC) | needs a reliable key |
| Dedup on read | `row_number()`, keep latest | can't fix the write layer | cost paid on every read |
| Content hash | `md5(payload)` as key | source has no stable key | hash changes if format changes |
Default guidance (explicit): overwrite-partition for date-based tables, MERGE for entity tables — covers most real cases.

### Late-arriving data and lookback window (explicit, p.18)
Data can arrive at the warehouse days after the event actually happened (app offline then syncs, producer retries, slow upstream batch, users editing old records). Mitigation: a **lookback window** — every incremental run recomputes the last N days, not just the newest partition. Sizing rule (explicit): set the lookback to the P99 of `(_ingested_at - event_time)`, **measured from Bronze, not guessed**. Explicit warning: when a label for the past changes (e.g., a user's satisfaction click arrives 3 days late), don't edit the old training snapshot — create a new version.

### Slowly Changing Dimension Type 2 / SCD2 vs overwrite (explicit, p.19)
Overwrite loses history: if T-91's priority changes low→high on 08-14, an overwritten table only shows "high," so "what was the priority when the ticket was created?" is unanswerable, and a training set labeled by today's state teaches the model information it wouldn't have had at prediction time. SCD Type 2 keeps history: each change creates a new row with `valid_from` / `valid_to` / `is_current`; joins use point-in-time logic to fetch the state at the moment the ticket was created. Getting this wrong is explicitly named as causing **training-serving skew**, deferred in depth to Day 19.

### Data testing tools — four options (explicit, p.20)
| Tool | Tests where | Strength | Fits |
|---|---|---|---|
| dbt tests | SQL run in the warehouse | attaches to models, no extra infra | Silver/Gold tables |
| Pandera | in-process Python | validates DataFrame schema (pandas, Polars, PySpark) | Python processing steps |
| Pydantic | per-record, at runtime | catches type errors right at the system boundary | API payloads, Kafka messages |
| Great Expectations / Soda | separate framework, with reporting | shared expectation suites across teams, self-documenting | large organizations |
Explicit rule for where to place checks: right after extract (catch source errors) → after transform (catch logic errors) → before training (final gate). Failing early is cheaper. Advanced test suites/auto-alerting deferred to Day 27.

### Five compute engines (explicit, p.21)
| Engine | Execution model | Strongest at | Cost |
|---|---|---|---|
| DuckDB | in-process, single machine, vectorized | analytics up to a few hundred GB; reads Parquet on S3 directly | doesn't scale out |
| Polars | Rust DataFrame, lazy execution | pandas replacement; much faster Python ETL | younger ecosystem |
| Spark | distributed, JVM | TB–PB scale; joining many large tables; huge ecosystem | cluster ops, slow startup |
| Trino | MPP, federated query | querying across S3 + Postgres + Kafka without moving data | not built for scheduled transform jobs |
| Ray Data | distributed, pure Python | batch inference/embedding on GPU | weaker for SQL analytics |
Running-example mapping: DuckDB builds Silver/Gold; Ray Data embeds 10M chunks for RAG; Trino used for ad hoc cross-checks between the lake and production Postgres.

### Engine decision tree and the "distributed by default" trap (explicit, p.22)
Decision flow: does each run's data fit in one machine's RAM? → yes & <10GB → DuckDB/Polars (single process, no cluster). → no, 10–500GB → DuckDB out-of-core on a large-RAM machine. → over 1TB or joining many large tables → Spark (cluster, shared across teams). Off this axis: Trino when the question spans multiple stores; Ray Data when the heavy work is calling a model, not SQL. Explicit warning: a 10-node Spark cluster processing 8GB is often slower than DuckDB on one machine — most time goes to scheduling/shuffle, not computation. Rule: start on one machine, only move to a cluster once you've measured and found it's genuinely tight.

### Shuffle, skew, and cost of distributed joins/aggregations (explicit, p.23)
Local operations (SELECT, WHERE) stay on one node; **shuffle** operations (JOIN, GROUP BY, DISTINCT, ORDER BY) write to disk → cross network → read back — expensive. Avoiding shuffle: for small tables (<100MB), use a **broadcast join** — send the full copy to every node, join locally. **Data skew**: e.g., one enterprise customer is 40% of tickets → partitioning by `customer_id` piles everything onto one task (example given: 200 tasks finish in 30s, 1 task takes 40 min). Second most common culprit: `user_id IS NULL`. Detect skew via the distribution of task durations, not the average. Fix via **salting**: group by `(customer_id, random 0–15)` then aggregate a second time.

### Small-file problem and partitioning strategy (explicit, p.24)
Problem: e.g., 50,000 files × 2MB each — every file requires a separate list/open/read-footer operation; the engine spends most time planning, not reading data; pipelines slow down gradually with no code change to blame. Fixes: target file size 128MB–1GB; run periodic compaction; reduce the number of write tasks at the end of the pipeline. Partitioning guidance: partition by a column you frequently filter on — almost always `event_date`; don't partition by `user_id` (high cardinality → recreates the small-file problem); aim for a few hundred to a few thousand partitions, not millions. **Predicate pushdown**: filtering on the partition column lets the engine skip whole directories without opening any file; filtering on other columns requires reading everything then discarding.

### Measure before optimizing; embedding cost trap (explicit, p.25)
Rule: read `rows scanned` before elapsed time — time varies with machine load, but rows-scanned doesn't lie. Example given: filtering on the wrong column scans 412M rows in 38.2s; filtering on the partition column scans 1.34M rows in 0.4s. Warning specific to the running example: embedding 10M chunks via Ray Data — if `gold_doc_chunks` is not idempotent, every rerun re-embeds everything, costing real money each time. Correct key: `hash(text) + embedding_model_version` — changing the model deliberately re-embeds everything, but never mixes two versions in the same index.

### Kafka topic as "Bronze with an expiration date" (explicit, p.26)
Direct analogy between batch Medallion and streaming: Kafka topic (immutable log + offsets) ≈ Bronze (append-only table); Flink/consumer transform ≈ Silver (scheduled transform); Feature table continuously updated ≈ Gold. Topic is immutable, append-only — matches Bronze's rule. Offset is each consumer group's private read position; replaying = reading from an old offset = "rerunning the pipeline." Explicit warning: **retention determines how far back you can replay** — 24h retention with a 3-day lookback means you cannot backfill, and you discover this exactly when you need it most.

### Broker choice — five options (explicit, p.27)
| Broker | Trait | Choose when / cost |
|---|---|---|
| Kafka | de-facto standard, largest ecosystem | safe default · heavy JVM ops tuning |
| Redpanda | C++, single binary, Kafka-API compatible | want low latency + less ops · smaller community |
| Pulsar | separates broker/storage, multi-tenant, tiered storage | one cluster serves many teams · more components to understand |
| Kinesis | fully AWS-managed | already on AWS, want turnkey · AWS lock-in, shard-based limits |
| WarpStream | uses object storage as the log, no local disk | cost matters more than latency · hundreds-of-ms latency |
Explicit note: all five expose nearly the same API — what actually locks you in is not the broker but **schema, partition key, and existing consumers**; be more careful with those three than with the broker choice.

### Stream processors — three schools (explicit, p.28)
| | Flink | Spark Structured Streaming | Kafka Streams |
|---|---|---|---|
| Model | event-at-a-time, true streaming semantics | micro-batch (small batches) | library embedded in a Java app |
| Latency | milliseconds | seconds | milliseconds |
| State | rich, proper checkpointing | present but simpler | local RocksDB |
| Choose when | event-time and state are central | team already uses Spark, wants one API for batch+stream | only transforming Kafka → Kafka |
| Cost | steepest operational learning curve | can't hit millisecond latency | not an independent cluster, hard to scale separately |
All three have a SQL API (Flink SQL, Spark SQL, ksqlDB) — Java is not required to get started.

### Event time vs processing time (explicit, p.29)
Event time = when the event actually occurred; processing time = when the system received/processed it. Example: a user acts on a subway (device offline), then reconnects and 5 events arrive within the same second. If windowing uses processing time, 2 hours of activity collapse into one minute — an "events per minute" feature spikes artificially, making a routing agent falsely believe the customer is having a severe incident. **Windows must be computed on event time.** **Watermark** = the point after which the system considers "all events before T have been seen," closing the window. **Allowed lateness** = the grace margin after the watermark; events later than that are explicitly dropped or routed to a side output — never silently discarded without a decision.

### Schema Registry as the streaming data contract (explicit, p.30)
| Mode | Allowed changes | Who upgrades first | Use when |
|---|---|---|---|
| BACKWARD | remove a field; add a field with a default | consumer | default — many consumers, few producers |
| FORWARD | add a field; remove a field that has a default | producer | producer upgrades frequently |
| FULL | only changes safe in both directions | either side | important topic, many dependents |
Explicit tie-back to Part 1: this is the Medallion's data contract, but enforced by the server at the moment the producer writes, rather than discovered late when the pipeline runs. Explicit warning: changing an event's schema changes a feature, meaning a running model receives different input without anyone redeploying it — for topics feeding online features, choose FULL and accept slower change velocity.

### When NOT to use streaming (explicit, p.31)
| AI-system need | Real latency requirement | Cheapest solution |
|---|---|---|
| Update RAG index from internal docs | hours | hourly batch |
| Retrain ticket classifier | days | nightly batch |
| Routing-agent feature (7-day ticket count) | minutes | 5-min micro-batch |
| Block a fraudulent transaction | milliseconds | real streaming |
Real cost of streaming (explicit): managing a state store, checkpoint/recovery, backpressure when a consumer lags, 24/7 on-call, harder debugging than batch (no "just rerun it for me"). Explicit principle: micro-batch (5 min) solves most "real-time" needs at a fraction of the operational cost — choose streaming because of a genuine latency requirement, not because it sounds more modern in an architecture review.

### Orchestration — four schools (explicit, p.32)
| Tool | Central unit | Strength | Cost |
|---|---|---|---|
| Airflow | task — DAG of steps | largest ecosystem of providers; v3 adds event-based scheduling + DAG versioning | heavy configuration, DAGs tend to bloat |
| Dagster | asset — table, model, file | declares "data assets" directly; built-in lineage/observability | requires rethinking in asset terms |
| Prefect | Python flow | dynamic flows, low ceremony, writes like normal code | fewer data-specific defaults |
| Temporal | durable execution | multi-day workflows, multi-step agents; retry lives in the runtime | not a data scheduler |
Explicit key distinction: the biggest difference is the unit of thought — Airflow asks "which steps run," Dagster asks "which tables must exist and stay fresh." For multi-step AI agents, Temporal is increasingly chosen, despite not belonging to the traditional ETL world.

### Safe reruns & backfill — four rules (explicit, p.33)
1. `catchup=False` — deploying a new DAG with the default `True` runs every missed schedule at once (example given: a training DAG deployed at 5pm with `start_date` 30 days back and default catchup instantly queues 30 runs, and a GPU cluster receives 30 jobs simultaneously).
2. `max_active_runs=1` — backfill must not overlap the regular daily run.
3. Backfill must use the exact same code path as the daily run — a separate script produces separate (inconsistent) results.
4. Partitioned writes — backfilling day 08-12 must not touch day 08-13.
Explicit final test / grading criterion for Lab 17: rerun the same old day three times in a row, checksum the Gold table after each run — all three checksums must match exactly.

---

## 2. Relationships & Mechanisms

### Medallion promotion pipeline (Input → Process → Output)
- **Input**: raw source events (Postgres row changes via CDC, S3 JSON dumped hourly, Kafka click/feedback events).
- **Process (Bronze)**: append-only capture with full lineage metadata, no cleaning — preserves ability to replay/audit.
- **Process (Silver)**: dedup, enforce keys/types, handle PII — produces one authoritative row per entity.
- **Process (Gold)**: aggregate/shape per specific consumer (RAG chunks, training snapshots, daily features).
- **Output**: three independent Gold tables feeding three different AI consumers (RAG index, classifier, routing agent) — each with its own, sometimes mutually incompatible, definition of "clean" (p.4, p.10).
- Why each step matters: skipping Silver's dedup breaks counts and silently biases model training toward frequently-edited entities (p.8); skipping Bronze immutability removes the ability to recover from bugs, retrain faithfully, or audit (p.7).

### CDC pipeline (Input → Process → Output)
Postgres WAL (Input) → Debezium reads WAL as a pseudo-replica, in order, including deletes (Process) → Kafka topic (Output, becomes Bronze-equivalent for streaming). Depends on: replication slot being enabled and continuously consumed (unread slot fills source disk — a boundary condition, p.11).

### dbt incremental run (Input → Process → Output)
Prior incremental table state + new Silver rows within the lookback window (Input) → `is_incremental()` branch filters to `event_date >= max(event_date) - lookback` (Process, guards against rescanning full history) → MERGE upsert by `unique_key` (Process, guards against duplication) → updated incremental table (Output). Depends on: `unique_key` correctness, `incremental_strategy` choice, and Bronze retaining enough history for `--full-refresh` to be safe (p.15).

### Data testing placement mechanism
Checks placed at three gates in sequence: right after extract (catches source-side errors cheaply) → after transform (catches logic errors) → before training (final gate, most expensive to fail late) (p.20). This is a general "fail cheap, fail early" mechanism, not tool-specific.

### Engine choice depends on data-volume and shuffle cost
Chain of reasoning: data fits in one machine's RAM → single-process engine (DuckDB/Polars) avoids network/scheduling overhead entirely → distributed engines (Spark) only pay off once data volume forces shuffling that no single machine could hold, and even then shuffle/skew must be actively managed (broadcast join, salting) or a distributed engine can be slower than a laptop (p.22–23).

### Streaming windowing depends on event time, which depends on watermark/lateness policy
Correct feature computation (e.g., "events per minute") depends on windowing by event time rather than processing time; the watermark mechanism determines when a window is considered complete; allowed lateness determines the grace period before late events are dropped or diverted (p.29). This chain is a prerequisite for any correct real-time feature used by a routing agent.

### Schema Registry enables (rather than replaces) the Medallion contract
The registry's compatibility mode (BACKWARD/FORWARD/FULL) enforces the same "data contract" idea as Silver's type/key enforcement, but earlier in the pipeline (at write time, server-side) rather than discovered later when a downstream job breaks (p.30) — an explicit connection drawn back to Part 1's Medallion.

### Orchestration retry/backfill mechanism depends on idempotent transforms
Airflow's `catchup`/`max_active_runs`/partitioned-write rules (p.33) only produce correct results if the underlying transform is idempotent (p.17) — orchestration safety and transform idempotency are explicitly presented as two halves of the same "chạy lại được" (rerunnable) requirement, tested together via the Lab 17 checksum criterion.

### Trade-offs named explicitly
- Bronze: no cleanliness ↔ full auditability/replayability (deliberate trade, p.6–7).
- Silver key/type enforcement: extra upfront rigor ↔ downstream trust ("đọc Silver là được quyền tin," p.9).
- dbt vs SQLMesh: ecosystem/hiring ease ↔ column-level lineage/cheaper dev environments (p.16).
- Dedup-on-read vs fixing at write: flexibility ↔ per-query cost (p.17).
- Streaming vs micro-batch: true low latency ↔ operational cost of state/checkpointing/24-7 on-call (p.31).
- FULL schema-registry compatibility: safety for many dependents ↔ slower schema change velocity (p.30).

---

## 3. Examples & Distinctions

### Bronze vs Silver vs Gold
Similarity: all three are stages of the same Medallion pipeline for the same underlying data. Difference: each answers a different question and makes a different promise (Bronze: "what happened," no cleanliness commitment; Silver: "what is the current state," one row per entity with enforced key/type; Gold: "what does this consumer need," shaped per specific downstream use, never a single universal table). Distinguishing criterion: what does a reader of this table get to assume is guaranteed true?

### CDC log-based vs polling
Similarity: both are mechanisms to capture changes from a source database into a pipeline. Difference: polling (`SELECT ... WHERE updated_at > ?` on a schedule) misses deletes and intermediate states and adds query load; log-based CDC (Debezium reading WAL/binlog) captures every INSERT/UPDATE/DELETE in order with minimal added load, at the cost of needing a managed replication slot. Distinguishing criterion: does the source's write-ahead log get read directly, or does the pipeline re-query current state?

### Avro vs Parquet vs Arrow
Similarity: all three are structured (schema-aware) binary formats, contrasted with human-readable but unstructured JSON. Difference: Avro is row-oriented, optimized for fast writes and schema evolution on the wire; Parquet is columnar, compressed, optimized for selective reads at rest; Arrow is columnar and in-memory, optimized for zero-copy exchange between engines. Distinguishing criterion: is the data in motion (Avro), at rest (Parquet), or in RAM being handed between compute engines (Arrow)?

### dbt vs SQLMesh
Similarity: both treat SQL transforms as version-controlled, tested, reviewed software rather than ad hoc scripts. Difference: dbt treats SQL as text+Jinja (developer must judge blast radius of a change, dev environment often requires rebuilding schemas); SQLMesh parses SQL into a lineage tree (auto-classifies breaking changes, backfills only what's necessary, near-zero-cost virtual dev environments) but has a smaller ecosystem. Distinguishing criterion: has the team been burned by wrong backfills / expensive dev environments (→ SQLMesh), or does it need community size and easy hiring (→ dbt)?

### Overwrite-partition vs MERGE/upsert vs Dedup-on-read vs Content-hash (idempotency techniques)
Similarity: all four guarantee the same end-state regardless of rerun count. Difference: overwrite-partition needs a clear time column and is cheapest; MERGE needs a reliable key and handles records revised later; dedup-on-read pays a cost on every query when the write layer can't be fixed; content-hash is used when the source has no stable key but the hash can break if the payload format changes. Distinguishing criterion: is there a reliable key, and does the source data get revised after first arrival?

### Overwrite vs SCD Type 2 (handling record revisions over time)
Similarity: both are strategies for storing an entity whose attributes change over time. Difference: overwrite keeps only the latest state, losing the ability to answer "what was true at time X" (causing training-serving skew if training labels are joined against today's state); SCD2 stores every version with `valid_from`/`valid_to`/`is_current`, enabling correct point-in-time joins. Distinguishing criterion: does any downstream consumer (e.g., a training pipeline) need to know the state of the entity as of a past point in time?

### Batch vs Streaming, mapped onto Medallion (p.26)
Similarity: identical three-stage structure (Bronze/Silver/Gold ≈ Kafka topic/Flink transform/Feature table). Difference: batch Bronze is an append-only table read on a schedule; streaming Bronze is a Kafka topic with retention-bounded replayability and per-consumer-group offsets. Distinguishing criterion: is "replay" bounded by a retention window (streaming) or by however long the batch table is retained?

### Streaming vs micro-batch vs true real-time need
Similarity: all can serve "seemingly real-time" AI features. Difference: hourly/nightly/5-min batch/micro-batch solve the vast majority of stated latency needs (RAG refresh: hours; retrain: days; routing feature: minutes) at much lower operational cost; true streaming is justified only when the requirement is milliseconds (e.g., fraud blocking). Distinguishing criterion: what is the actual, stated latency requirement — not how "modern" the architecture sounds.

### Flink vs Spark Structured Streaming vs Kafka Streams
Similarity: all three process streaming data and expose a SQL interface. Difference: Flink is event-at-a-time with rich state/checkpointing (ms latency, steepest ops curve); Spark Structured Streaming is micro-batch (seconds latency, simpler state, good if team already runs Spark for batch+stream unification); Kafka Streams is an embedded Java library for Kafka→Kafka transforms only (ms latency but not an independently scalable cluster). Distinguishing criterion: is event-time/rich-state central to the use case, is the team already on Spark, or is the job merely reshaping one Kafka topic into another?

### Event time vs processing time
Similarity: both are timestamps associated with an event as it flows through a streaming system. Difference: event time is when the event actually occurred (on the source device); processing time is when the system ingested/processed it. Distinguishing criterion: is the timestamp taken from the event's own payload/origin, or from the pipeline's clock at ingestion — and windowing/feature computation must use the former to avoid artificial spikes from network delays or offline gaps (p.29 subway example).

### Airflow vs Dagster vs Prefect vs Temporal
Similarity: all four orchestrate multi-step data/AI workflows with retry semantics. Difference: Airflow centers on tasks/DAG steps (largest ecosystem, DAGs can bloat); Dagster centers on assets (tables/models/files, built-in lineage); Prefect centers on Python flows (low ceremony, code-like); Temporal centers on durable execution for long-running, multi-day, multi-step (often agentic) workflows and is explicitly "not a scheduler for data." Distinguishing criterion: does the team think in terms of "steps to run" (Airflow), "assets that must exist/stay fresh" (Dagster), "Python functions" (Prefect), or "long-running durable agent workflows" (Temporal)?

---

## 4. Assumptions, Boundaries & Gaps

### Prerequisites / assumed background
- Explicitly assumes Day 10 content as already known and does NOT repeat it: what a data pipeline is and its stages, ETL vs ELT choice for AI, batch vs streaming at a conceptual level, the 6 dimensions of data quality, and "Airflow is the orchestrator, dbt is the transform" (p.2). Day 17 builds on top of these with "how it runs internally, and why choose X over Y."
- Table storage formats enabling ACID/time-travel/schema-evolution on object storage (Delta Lake, Iceberg) are explicitly deferred to Day 18 — not covered here even though Bronze/Silver/Gold storage is discussed.
- Feature store and Vector DB ("Phục vụ AI" layer of the 2026 tech map) are named but deferred to Day 19.
- Advanced data-quality test suites and automated alerting deferred to Day 27.
- Training-serving skew (raised as a consequence of SCD handling mistakes) is named but its full treatment deferred to Day 19.

### Constraints / limitations stated in the source
- Debezium's replication slot: if not continuously consumed, fills the source database's disk — an operational constraint requiring active monitoring, not fully resolved in the deck (p.11).
- `--full-refresh` in dbt incremental models is only safe if Bronze retains sufficient history; if a TTL has purged Bronze, a full-refresh silently produces an incomplete table with no warning mechanism described (p.15).
- Kafka retention directly bounds replay/backfill capability for streaming pipelines — a hard boundary condition, not something the deck offers a technical workaround for beyond "size retention to your lookback need" (p.26).
- LLM-judge/self-consistency-style cost trade-offs are not part of this deck (that's Day 13/14 territory) — irrelevant here, but the deck's own repeated pattern is: idempotency/dedup/hash techniques each have an explicit, unresolved cost trade-off (e.g., content-hash breaks if source format changes, p.17).
- Broadcast join threshold (<100MB) and partition-count guidance ("a few hundred to a few thousand") are given as rules of thumb without a formula for deriving them from a specific workload (p.23–24).

### Edge / failure cases explicitly called out
- Airflow retry at 3am on a keyless table silently duplicates rows with no error raised (p.9).
- A "Silver" table that adds no real guarantee (mere passthrough filter) is flagged as an anti-pattern layer that should be deleted (p.8).
- Late-arriving data changing a past label — explicitly warned not to overwrite the old training snapshot in place; must version instead (p.18).
- Deploying a new Airflow DAG with default `catchup=True` and a `start_date` 30 days back instantly triggers 30 concurrent runs, overwhelming a GPU cluster (p.33, explicit real scenario).
- Data skew from a single dominant customer (40% of tickets) or from `user_id IS NULL` collapsing a distributed job onto one task (p.23).
- Small-file problem (50,000 × 2MB files) causing gradual, silent pipeline slowdown with no code change to blame (p.24).
- Non-idempotent `gold_doc_chunks` causing full, costly re-embedding of 10M chunks on every rerun (p.25).
- Windowing by processing time instead of event time causing an artificial feature spike that misleads a routing agent into thinking a customer has a severe incident (p.29).
- Schema change on a Kafka topic feeding online features silently changes model input without redeploying the model (p.30).

### Concepts mentioned but not sufficiently explained in this deck (flagged, not filled from external knowledge)
- **Delta Lake / Iceberg** — named as the actual storage format answer for Bronze/Silver/Gold but explicitly deferred to Day 18; not explained here beyond the name-drop.
- **Feature store / Vector DB** internal mechanics — named as the "Phục vụ AI" tech-map layer, deferred to Day 19.
- **Training-serving skew** — named as the consequence of SCD-handling mistakes, but the deck doesn't define/walk through the mechanism in depth; deferred to Day 19.
- **RocksDB** (Kafka Streams' local state store) is named in the stream-processor comparison table but not explained.
- **Tiered storage** (Pulsar) and **WarpStream's object-storage-as-log** design are named as differentiators but not explained mechanically.
- **Salting** for skew mitigation is named with the general idea (group by `(key, random 0–15)` then aggregate again) but no worked numeric example is given.
- **Predicate pushdown** is described conceptually (skip whole directories when filtering on partition column) but no engine-specific implementation detail is given.
- Exact criteria for Airflow's DAG-versioning / event-based scheduling (mentioned as an "Airflow 3" strength) are named but not detailed (p.32).

---

## 5. Learning Priorities

### Essential (required to understand the lecture)
- Medallion as three levels of *commitment* (not just table names): what each of Bronze/Silver/Gold promises and does not promise, and why each answers a different question.
- Bronze immutability and why it matters (rerun after bug, faithful retraining, audit) — the three motivating scenarios.
- Silver's two requirements (key for MERGE/idempotent writes; correct types/PII boundary) and the consequence of skipping dedup.
- Gold as consumer-specific, never a single table — the three-consumer example (RAG chunks / training snapshot / daily features) and each one's failure mode.
- CDC log-based (Debezium/WAL) vs polling — mechanism and trade-offs.
- The four data formats (JSON/Avro/Parquet/Arrow) and which stage each belongs to.
- dbt incremental models: the three critical config lines (`unique_key`, `incremental_strategy`, `is_incremental()`) and the `--full-refresh` boundary condition.
- Idempotency: definition, why every real system needs it (at-least-once), and the four techniques with their fit/cost.
- Late-arriving data and the lookback-window sizing rule (P99 of ingestion lag, measured not guessed).
- SCD Type 2 vs overwrite and its link to training-serving skew.
- The engine decision tree (single machine vs cluster) and the "distributed by default" trap.
- Shuffle vs local operations, data skew, and small-file problem — mechanism and fixes (broadcast join, salting, compaction, partition choice).
- Kafka topic as "Bronze with an expiration date," and retention as a hard bound on replay/backfill.
- Event time vs processing time, watermark, allowed lateness — and why windows must use event time.
- When NOT to use streaming (latency-requirement-driven choice, not fashion-driven) and the true operational cost of streaming.
- Safe rerun/backfill rules (`catchup=False`, `max_active_runs=1`, same code path, partitioned writes) and their dependency on idempotent transforms — tied to the Lab 17 checksum test.

### Important (substantially improves understanding)
- dbt vs SQLMesh comparison and the shared underlying philosophy (transforms as versioned/tested code).
- Data testing tool landscape (dbt tests / Pandera / Pydantic / Great Expectations-Soda) and the three-gate placement rule.
- Ingestion tool comparison (Debezium / Airbyte / Fivetran / Kafka Connect / custom script) and their trade-offs.
- Five compute engines table and their fit by data volume/workload type.
- Measure-before-optimizing discipline (`EXPLAIN ANALYZE`, rows scanned over elapsed time) and predicate pushdown.
- Broker comparison (Kafka/Redpanda/Pulsar/Kinesis/WarpStream) and the explicit point that schema/partition-key/consumers lock you in more than the broker itself.
- Stream processor comparison (Flink/Spark Structured Streaming/Kafka Streams).
- Schema Registry compatibility modes (BACKWARD/FORWARD/FULL) as the streaming data contract, tied back to Medallion's contract idea.
- Orchestration tool comparison (Airflow/Dagster/Prefect/Temporal) and their differing "unit of thought."

### Supporting (useful, not central)
- Specific example SQL/YAML snippets (bronze table DDL, dbt schema.yml contract, incremental model Jinja) — illustrative of the concepts above, not independently essential.
- Specific numeric examples (412M vs 1.34M rows scanned; 38.2s vs 0.4s; 50,000×2MB files) — illustrative, not the mechanism itself.
- The 2026 tech-map table naming specific product names per layer (Ingestion/Vận chuyển/Lưu trữ/Tính toán/Biến đổi/Điều phối/Chất lượng/Phục vụ AI) — orientation aid, not itself a mechanism.
- Lab #17 logistics (2.5 hours, deliverable format, specific bugs to fix) — administrative, not conceptual.
- Next-lecture preview content (Day 18 Delta Lake/Iceberg teaser) — forward pointer only.
