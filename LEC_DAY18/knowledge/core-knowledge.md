# Core Knowledge

Source: `stream.pdf` — "Data Lakehouse Architecture" (AICB-P2T2 · Ngày 18 · Chương 4: Hạ Tầng), 57 PDF pages / slides numbered 1–45 (some pages are section dividers/title/Q&A not in the x/45 count). Language: Vietnamese slides; extraction preserves original terms where precision matters. Track: AI Infrastructure specialization (Day 16+).

## 1. Core Concepts

- **Data Warehouse** (2000s) — Integrated system where storage, query engine, and governance live in one product; data loaded and accessed only via SQL (e.g., Teradata, Oracle Exadata, Redshift, BigQuery, Snowflake). Principle: **schema-on-write** — structure must be declared before write; non-conforming data rejected at write time. Gains: full ACID, fast queries (index/statistics/optimizer), fine-grained governance. Limits: high cost, inflexible (structured data only), closed/proprietary format. [p5]

- **Data Lake** (2010s) — Object storage (S3/GCS/ADLS) + files, no governance layer in between; apps write files directly, readers open files directly. Storage and compute decoupled. Principle: **schema-on-read** — write first, interpret at read time. Advantages: cheap (~$23/TB/month on S3), accepts any format (JSON, image, audio, PDF, transcript), open (Parquet on S3 readable by Spark/DuckDB/Trino/pandas), independently scalable. Explicit: absence of a governance layer means no schema constraint, no transactions, no history. [p6]

- **Data Swamp (four limitations of Data Lake)** — (1) Incomplete writes: job stops mid-write, no "incomplete" marker, readers see mixed state; (2) Concurrent writes: two jobs overwrite the same partition, last writer wins silently, no error; (3) Schema drift: upstream type change (e.g., int→string) goes undetected at write time; (4) No history: cannot query table state at a past point in time; common workaround (daily full copy) increases storage ~30x. Explicit root cause (shared by all four): on object storage nothing defines which files, at which point in time, belong to "the table." [p7]

- **Lakehouse** — Not a separate product; a data lake plus a metadata layer ("table format") that can define exactly which files constitute a table. Four-layer stack: Object storage (S3/GCS/ADLS) → File format (Parquet/ORC/Avro, from Day 17) → **Table format** (Delta Lake/Apache Iceberg/Apache Hudi — new layer; stores file list, version number, schema, statistics; implements ACID and time travel) → Compute engine (Spark/Trino/DuckDB/Flink/Snowflake — all read the same table). Explicit: the added layer is only small JSON metadata files; the other three layers already exist in a data lake. [p8]

- **ACID** — Four transaction properties. **Atomicity**: a write is one unit — fully in effect or not at all (lakehouse example: job stops at file 600/1000 → readers still see pre-job state). **Consistency**: after each write the table still satisfies declared constraints (schema, types, required columns) — e.g., writing string into a BIGINT column is rejected at write time. **Isolation**: concurrent reads/writes produce a result equivalent to sequential execution; lakehouse implements this as **snapshot isolation** (query fixed to the version at query start, unaffected by concurrent commits). **Durability**: committed transactions survive power loss/crash/network failure; object storage provides this (S3 designed for 99.999999999% / 11 nines durability) — explicit: this is the *only* ACID property a plain data lake already has. [p13, p14]

- **Transaction log (commit)** — Because S3 only guarantees atomic single-object PUT, table-defining information is concentrated into one object: a JSON commit file. Three-step process: (1) write new Parquet files (not yet "in" the table), (2) commit — write one JSON file describing version delta (atomic PUT), (3) read — reader doesn't LIST the directory but reads the log to get the file list for the desired version. [p16]

- **Delta Lake** — Open table format = Parquet files + `_delta_log/` transaction-log directory + a read/write protocol (Delta protocol). Serverless, no background process. Converting Parquet→Delta requires no data movement, only adding the log; if Delta is abandoned, the Parquet files remain readable. [p17]. Directory structure: numbered JSON commit files (`00000000000000000000.json`, zero-padded to 20 digits so lexical order = chronological order); table version = highest commit number; every 10 commits a `.checkpoint.parquet` consolidates state so readers don't replay every JSON; `_last_checkpoint` points to the latest checkpoint. Explicit: more Parquet files exist in the directory than are logically "in" the table — removed files are retained for time travel until `VACUUM`. [p18]. Log is append-only; record types: `add` (file belongs to table, with row count and per-column min/max), `remove` (file no longer belongs), `metaData` (schema/config). No `update` record type — row edits = remove old file + add new file (basis of time travel). Commit cost independent of table size (only the delta is logged). [p19]

- **Optimistic concurrency control (Delta write path)** — (1) read current version (e.g., 6), (2) write new Parquet files (invisible to others yet), (3) attempt commit: create next JSON file conditioned on it not already existing, (4) success = commit complete; if file already exists = conflict. Non-overlapping writes (different partitions) auto-retry and commit as next version without rerunning the job; overlapping writes fail explicitly with `ConcurrentModificationException` (explicit fail preferred over silent overwrite). Underlying mechanism: S3 lacked native "create-if-not-exists," so Delta historically used a DynamoDB table (LogStore) for coordination; simplified since S3 added conditional writes (`If-None-Match`). [p21]

- **Schema Enforcement & Evolution** — Enforcement = the Consistency property; wrong-type write is rejected (`AnalysisException`), job fails, table stays intact. Evolution = deliberate schema change (e.g., add column) via explicit `mergeSchema=true` option (new column added, old rows get NULL); changing a column type needs `columnMapping`. Explicit caution: enabling `mergeSchema` on every job is equivalent to disabling enforcement. [p22]

- **MERGE (upsert/delete by row)** — `DeltaTable.merge(...).whenMatchedUpdateAll(condition=...).whenNotMatchedInsertAll()`; `tbl.delete(...)` for row-level delete (e.g., GDPR). Without MERGE: read whole table → union new data → dedupe → overwrite whole table (100K-row update rewrites full 40TB). Default mode is **copy-on-write** — editing one row rewrites the entire file containing it, so large files are costly. [p23]

- **OPTIMIZE / ZORDER / Liquid Clustering / VACUUM** — `OPTIMIZE` compacts many small files (e.g., 500K × 1MB files/year from per-minute streaming writes) into 128MB–1GB files; cost driver is number of files (each = one object-storage request), not total bytes. `ZORDER BY (col)` co-locates rows with similar values, narrowing min/max ranges in the log for better data skipping. **Liquid clustering** (Delta 3.x) achieves similar goals incrementally — clusters only new data, no full rewrite. `VACUUM RETAIN <n> HOURS` deletes removed files older than the retention window (e.g., 168h/7 days default reference; up to 30+ days for compliance-sensitive domains); trade-off: time travel before that point is lost. [p24]

- **Time Travel** — Query a table at a specific version or past timestamp using ordinary SQL/DataFrame syntax. Mechanism: log records exactly which files belong to each version; edits are remove-old/add-new so old files persist on object storage; reading version N = replay log to N, get the file list, open those files. No data copy is made; cost = storage of old files during the retention window. [p26]

- **Data Versioning** — Broader concept: every table state has an identifier (version number) referenceable in docs/tickets/model-training metadata. "Trained on silver_tickets version 412" is verifiable; "trained on August data" is not. Comparison to Git: similarity = append-only history, each state has an ID, can revert; difference = Git diffs text file contents, Delta stores file lists — enabling Delta to handle 40TB tables. [p26]

- **RESTORE** — Rollback mechanism: does NOT delete the newer version (e.g., 413); instead writes a NEW version (e.g., 414) whose content = "table = state of version 412." The rollback itself is recorded in the audit trail and can itself be rolled back. [p27]

- **Apache Iceberg** — Table format developed at Netflix (2017–2018), donated to Apache, top-level project 2020; designed engine-agnostically (open spec, not tied to one engine). Addresses four limitations of Hive-style directory-defined tables: (1) determining table contents requires LIST on S3 (slow, rate-limited at scale), (2) query users must know the partitioning scheme or risk full scans, (3) changing partitioning requires rewriting all old data, (4) no atomic commit. [p31]

- **Iceberg four-tier metadata tree** — **Catalog**: stores a pointer to the table's current metadata file; changing the table = atomically changing the pointer (implementations: REST Catalog, Glue, Nessie, Polaris, Hive Metastore). **metadata.json**: schema, partitioning, config, and list of snapshots (each snapshot = one table version). **Manifest list**: per-snapshot, lists manifests with partition value ranges, enabling skip of whole file groups. **Manifest file**: lists individual data files — path, row count, per-column min/max, partition values. **Parquet files**: actual data, same format as Delta/plain data lake. Deeper structure than Delta but enables filtering at every tier; commit = a single pointer swap at the catalog (compare-and-swap), needing no external coordinator. [p32]

- **Hidden Partitioning (Iceberg)** — Table defined with `PARTITIONED BY (days(created_at))` — a transform function, not a raw column. Iceberg stores the relationship between the column and the partition expression, so it can infer needed partitions even when the query filters on the underlying column (`WHERE created_at >= ...`) without the user knowing the partition scheme; on Hive/Delta, an equivalent query without an exact partition-column filter causes a full scan. [p33]

- **Partition Evolution (Iceberg)** — Changing the partition granularity (e.g., `days(created_at)` → `hours(created_at)`) via `ALTER TABLE ... REPLACE PARTITION FIELD ...` without rewriting existing data. Delta addresses the same underlying need differently, via liquid clustering. [p33]

- **Apache Hudi** — Mentioned as a "third option," strong at streaming upserts via key-based indexing (developed at Uber); explicitly stated as less common in AI projects, not covered in depth this lecture. [p35]

- **Point-in-Time Correctness** — Requirement, especially for AI/ML, that a feature/label be computed using only data that existed as of the prediction/labeling time, to avoid data leakage. [p39]

- **RAG table requirements (gold_doc_chunks)** — Required columns: `chunk_id, doc_id, doc_version, chunk_text, token_count, chunk_strategy, valid_from, embedding_model, embedding_dim, source_uri, updated_at`. The lakehouse table is the source of truth to rebuild the vector index; the vector store itself (index) is Day 19 content. [p40]

- **LLM inference log table** — Minimum columns: `request_id, ts, model, prompt_version, input_tokens, output_tokens, latency_ms, cost_usd, tool_calls, trace_id, user_feedback`. [p41]

## 2. Relationships & Mechanisms

- **Evolution chain**: Data Warehouse → Data Lake → Lakehouse — each architecture emerged to fix a specific limitation of its predecessor. Warehouse (ACID+schema, high cost, closed) → Data Lake (low cost, open, no ACID/schema → data swamp risk) → Lakehouse (ACID+schema+history restored, low storage cost, open format). Explicit "essence": Lakehouse introduces no new concept — it restores the guarantees warehouses already had, implemented on object storage. Three enabling conditions: open table format (Delta/Iceberg/Hudi) + low-cost object storage + decoupled compute engine (Spark/DuckDB/Trino). [p9]

- **Depends on / enables**: Lakehouse's ACID, schema enforcement, and time travel all depend on (are "built on top of") the table-format layer's commit mechanism (a single atomic metadata-defining file/pointer). [p16, p32, slide 44 takeaway 1]

- **Object storage capability → table-format design**: S3 guarantees atomic single-object PUT, high durability, strong read-after-write (since 2020); S3 does NOT guarantee: multi-object transactions (writing 1000 files = 1000 separate ops), LIST as a consistent snapshot, atomic rename (rename = copy+delete), locks, or "table version." Because of this gap, both Delta (atomic JSON commit) and Iceberg (atomic catalog pointer swap) each rely on exactly one atomic low-level primitive to build ACID at a higher layer. Prior/rejected workarounds and their flaws: temp-dir-then-rename (rename not atomic → intermediate states), `_SUCCESS` flag files (depends on every reader checking it), per-day directories (avoids overwrite but directory count explodes, no row-level fix), self-built locks via Redis/DynamoDB (hard to guarantee correctness in all cases). [p15]

- **Delta read process (Input → Process → Output)**: Input = `_last_checkpoint` pointer. Process = (1) load nearest checkpoint (e.g., v10), replay subsequent JSON commits (11,12,13); (2) resulting file list + schema + per-column min/max at v13; (3) data skipping — discard files whose min/max stats can't match the query predicate; (4) open only remaining files, no directory LIST at all. Output = correct file set for the target version, opened efficiently. Why it matters: eliminates slow/rate-limited S3 LIST calls; enables skipping data before reading any bytes. This is also the basis of snapshot isolation — the file list is fixed for the whole query even if other jobs commit v14/15/16 concurrently. [p20]

- **Problem → Delta mechanism → Result mapping** (explicit summary table, p25):
  - Job stops mid-write → atomic JSON-file commit → table stays at prior state on failure.
  - Two jobs overwrite each other → optimistic concurrency (check at commit time) → non-overlapping writes both succeed; overlapping writes fail explicitly.
  - Read during write → query fixed to one version → snapshot isolation, consistent results throughout the query.
  - Undetected schema drift → `metaData` in log + write-time check → wrong-type writes rejected immediately.
  - No history → old files retained + log records every version → `versionAsOf`, `RESTORE`, `DESCRIBE HISTORY`.
  - Row update/delete rewrites whole table → MERGE/DELETE act per affected file → 100K-row update doesn't rewrite 40TB.
  - Hundreds of thousands of small files → OPTIMIZE, clustering, min/max stats in log → fewer files, irrelevant files skipped.
  Overall cost of all these guarantees: a few-MB JSON log directory plus periodically scheduled OPTIMIZE/VACUUM jobs.

- **Time travel mechanics and choice of version vs. timestamp**: `versionAsOf` (integer, timezone-independent) is for exact reproduction; `timestampAsOf` is for incident investigation when only a rough time is known, not yet a version number. [p27]

- **RESTORE mechanism**: appends a new commit rather than deleting history — this makes rollback itself auditable and reversible. [p27]

- **MLflow + data_version pattern**: logging `data_version` (e.g., 1 vs 3) alongside hyperparameters lets a metric drop (e.g., AUC 0.81→0.62) be traced to which dataset version was used (e.g., a version containing 4M erroneous rows before RESTORE), separating data-caused from code/hyperparameter-caused regressions. Rule: every training run must log `table_path` and `version`. [p29]

- **Time travel's three boundaries**: (1) bounded by retention — `deletedFileRetentionDuration` and `logRetentionDuration` (default 30 days); exceeding either loses that version; explicit configuration to know: `VACUUM RETAIN 168 HOURS` = 7-day window. (2) Not a substitute for backup — if the S3 prefix/bucket itself is deleted, the log and data are both gone; time travel fixes logic errors, not infrastructure loss (still need S3 versioning, cross-bucket replication, strict permissions). (3) Storage cost — retaining 30 days of history for a heavily-upserted table multiplies storage well beyond current data size; principle: retention should exceed worst-case incident-detection time. Reference figures given: 7 days for ingest tables, 30–90 days for model-training tables. Common mistake flagged: `VACUUM RETAIN 0 HOURS` removes rollback capability and can fail in-flight queries. [p30]

- **Delta vs Iceberg — shared core vs. differing mechanism**: Both provide ACID w/ snapshot isolation, time travel + rollback, schema evolution, row-level MERGE/UPDATE/DELETE, metadata-based data skipping, Parquet storage, open-source/Apache-licensed specs, periodic small-file compaction. They differ in metadata organization (Delta: linear JSON log; Iceberg: snapshot-based metadata tree) → which drives differing commit mechanisms (Delta: put-if-absent next-log-file; Iceberg: catalog compare-and-swap of a pointer) and differing scalability characteristics; catalog role (Iceberg mandatory; Delta optional); partitioning approach (Iceberg hidden partitioning + partition evolution; Delta liquid clustering). Explicit: because the core is equivalent, the real selection driver is usually the engine/catalog ecosystem already in use, not "which format is better." [p34, p36]

- **Interoperability trend**: Delta UniForm (writes in Delta, also emits Iceberg-compatible metadata so Iceberg engines can read it); Apache XTable (converts metadata Delta↔Iceberg↔Hudi without copying data); market signals cited: Databricks acquired Tabular (Iceberg's creators) in 2024, AWS released S3 Tables. [p36]

- **AI/ML vs BI requirements table** (explicit, p37) — five rows: exact dataset reproduction (BI: rarely needed; AI/ML: mandatory — solved via time travel + version logging); unstructured data (BI: rare; AI/ML: frequent — solved via files on object storage + table metadata pointer); high-throughput full-table scans (BI: rare/narrow filters; AI/ML: every epoch — solved via columnar Parquet, large files, parallel multi-file reads); point-in-time correctness (BI: rarely needed; AI/ML: mandatory — solved by reading table as of the correct timestamp); row-level delete/GDPR (BI: yes; AI/ML: yes, more complex — solved via DELETE/MERGE with audit trail). Explicit note: this table explains why Lakehouse became the default AI infra choice — warehouse satisfies row 5 but not rows 2–3; plain data lake satisfies rows 2–3 but not rows 1 and 4.

- **Fix-version-before-read pattern** (Input → Process → Output): Input = a Delta table being actively written by an ingest job. Process = (1) read current version number FIRST via `DeltaTable(path).version()`, before any data read; (2) read explicitly that version (not "latest"); (3) log the version as an MLflow parameter alongside hyperparameters. Output = a training run whose input dataset is a fixed, reproducible integer reference, reconstructible months later via `DeltaTable(path, version=412)`. Why it matters: reading "latest" twice in one script can silently return two different datasets if ingestion is still committing. `deltalake` (Python/Rust) works without Spark/JVM. [p38]

- **Point-in-time join mechanism / data leakage example**: Predicting ticket escalation at ticket-creation time using "number of tickets customer has opened" as a feature. WRONG approach: join against the current-state table → feature counts tickets that didn't exist yet at prediction time (leakage) → inflated offline AUC (0.94) that collapses in production (0.62). CORRECT approach: read the table `timestampAsOf` the prediction time → feature reflects only information available then → lower but production-matching offline AUC. Explicit boundary: table-level time travel solves this at the table granularity; when each row needs its own as-of timestamp, that requires row-level as-of join — a feature-store capability, deferred to Day 19. [p39]

- **RAG versioning mechanism**: because embedding models change periodically, overwriting the chunk table in place breaks retrieval-quality comparison, prevents rollback to a worse new embedding, and prevents tracing a wrong answer back to its source chunk. Fix: append new rows with a new `embedding_model` value while keeping old rows, and A/B test before full cutover. [p40]

- **LLM inference log ACID rationale**: append-only high-frequency multi-process writes need atomic commits to avoid losing entries; late-arriving `user_feedback` (days later) requires MERGE to attach it to the correct existing row; cost/usage analytics by day and `prompt_version` require SQL queryability. [p41]

- **LLM-generated training data versioning**: pipeline = LLM generates data → filter/review → training set; each loop = one table version, enabling per-loop contribution to model quality to be measured; without versioning, data from different loops mixes and individual contribution can't be measured. [p41]

## 3. Examples & Distinctions

- **Warehouse vs Data Lake vs Lakehouse** (8-criterion comparison table, p10): storage location, storage cost, data type support, schema handling, ACID, history/rollback, engine compatibility, suited workload. Key stated value of Lakehouse: no need to copy data to a separate system to run ML — every copy is a potential source of metric drift between teams.

- **Lakehouse vs Data Mesh vs Table Format** — explicitly distinct concept types, not competitors: **Lakehouse** = technical architecture (centralized storage on object storage, distributed compute, ACID from the table-format layer; scope = where data lives and what guarantees correctness). **Data Mesh** = organizational model (each domain team owns/is responsible for its data product; federated governance; scope = who is responsible for data) — NOT a software product. **Table format** = technical specification (Delta/Iceberg/Hudi — metadata conventions so multiple engines understand the same table; scope = which files belong to a table, at which version). Note: an org can run Data Mesh with every domain using Lakehouse, and every Lakehouse must pick a table format. [p11]

- **Three ACID-absence incidents** (p12): (1) write stops mid-job → dashboard silently reads wrong result (missing: Atomicity); (2) two concurrent writers (backfill + hourly job) overwrite the same partition, no error/warning (missing: Isolation); (3) a training job reading at 2:00 while an ETL job deletes/rewrites the same partition at 2:05 → training job fails or completes with inconsistent data (missing: Isolation). Common thread: the system gives no error signal in any case — problems surface only when downstream numbers look wrong.

- **A/C examples** (p13): Atomicity analogy outside lakehouse = bank transfer, no "debited but not credited" state exists. In lakehouse: job stopping at file 600/1000 leaves readers seeing the pre-job state. Consistency: writing string-typed `amount` into a BIGINT-declared column is rejected at write time; without this property, bad data enters silently and surfaces only at consumption, often weeks later.

- **Version vs Timestamp use** (p27): `versionAsOf` — reproducibility (exact, integer, timezone-independent). `timestampAsOf` — incident investigation (rough time known, version unknown yet).

- **Four time-travel use cases** (p28): (1) reproduce a model — retrain under exact 3-months-ago conditions when production model behaves oddly; without versioning, thousands of upserts later the old dataset no longer exists and cause (data vs. code) can't be isolated; with versioning, `versionAsOf` solves it directly. (2) roll back bad data — bad ingest job wrote 4M invalid rows; without versioning requires a manual identify-and-delete script on a live table (2–4 hours, uncertain completeness); with versioning, `RESTORE` takes seconds. (3) audit/compliance — "what data + who changed it as of June 30" is answerable directly via `DESCRIBE HISTORY` + `timestampAsOf`; without versioning this information isn't retained at all (compliance risk in regulated domains). (4) investigate metric discrepancy — same query gives different results today vs yesterday; running it against yesterday's and today's versions and comparing isolates whether the cause is data change or code change; stated as the most common operational use case.

- **Hive-style table vs Iceberg approach** (p31): Hive defines a table as "every file under a directory, partitioned by subdirectory" — a definition that breaks down at scale (LIST-based, requires knowing partition scheme, partition changes require full rewrite, no atomic commit). Iceberg defines a table via an explicit metadata tree listing individual files with stats and partition values — file paths become an implementation detail.

- **Delta Lake vs Apache Iceberg** (full comparison table, p35): origin (Databricks/2019 open-sourced vs Netflix/Apache top-level 2020); metadata (linear JSON log + checkpoint vs metadata.json→manifest-list→manifest tree); commit mechanism (put-if-absent next log file vs catalog compare-and-swap pointer change); catalog (optional vs mandatory: REST Catalog/Glue/Nessie/Polaris/Hive); hidden partitioning (no — must filter on exact partition column vs yes); partition evolution (no, uses liquid clustering instead vs yes); row-level delete (MERGE copy-on-write, deletion vectors in 3.x vs merge-on-read from spec v2, deletion vector v3); best-supported engines (Spark/Databricks, delta-rs for non-JVM vs Spark/Trino/Flink/Snowflake/BigQuery); common adopters (Databricks, Azure vs Netflix, Apple, AWS, Snowflake, Cloudera).

- **Selection criteria** (p36): Choose Delta when — already on Databricks/Azure (built-in integration), workload mainly Spark with a small team/single table owner, need fast deployment (`pip install deltalake`, write to a directory, no catalog needed). Choose Iceberg when — multiple engines read the same table (Spark, Trino, Flink, Snowflake), need vendor neutrality, table is large with many external query consumers and partitioning may need to change. Explicit: data in both formats is Parquet; the cost of switching later is mainly team process/catalog integration, not the data itself.


## 4. Assumptions, Boundaries & Gaps

- **Time travel is retention-bounded**: cannot query beyond `deletedFileRetentionDuration`/`logRetentionDuration` (default 30 days); `VACUUM` permanently forecloses time travel before its retention cutoff. [p30]
- **Time travel is explicitly not a backup**: does not protect against bucket/prefix deletion; still requires S3 versioning, cross-region/bucket replication, and strict permissions as separate safeguards. [p30]
- **Storage cost trade-off** of longer retention on heavily-upserted tables is explicit but no formula/exact multiplier given beyond "many times current data size." [p30]
- **MERGE default is copy-on-write** — rewrites entire files containing changed rows; the slides note Delta 3.x deletion vectors as a partial mitigation but do not explain the deletion-vector mechanism itself — flagged as mentioned-not-explained. [p23, p35]
- **`mergeSchema=true` used on every job effectively disables schema enforcement** — an explicit caution, not elaborated with guardrails beyond "must be deliberate." [p22]
- **Iceberg catalog is mandatory**; Delta's catalog is optional — the practical operational implications of this difference (e.g., deployment complexity trade-offs) are stated as a comparison-table fact but not elaborated. [p35]
- **Row-level as-of join (feature store capability)** is explicitly named as needed once features require per-row timestamps, but is explicitly deferred to Day 19 — not explained in this deck. [p39]
- **Apache Hudi** is explicitly named as a third table-format option "strong at streaming upsert via key-based indexing" but the mechanism is not explained — explicitly stated as out of scope for this lecture. [p35]
- **Vector store / index mechanics** — the deck states the lakehouse table is the source of truth for rebuilding a vector index, but vector store internals are explicitly deferred to Day 19. [p40]
- **Deletion vector v3 (Iceberg spec v2/v3 merge-on-read)** is named in the comparison table but not explained mechanistically anywhere in the deck — flagged as a term introduced without sufficient in-deck explanation. [p35]
- **Prerequisite assumed**: Day 17 content (Medallion architecture Bronze/Silver/Gold, ingestion patterns CDC/connectors/Kafka, transform tools dbt/SQLMesh, engines DuckDB/Polars/Spark/Trino/Ray, Parquet/Avro/Arrow, small-file problem, partitioning strategy) is explicitly NOT repeated in Day 18 and is treated as already known. [p2]
- **Feature store / as-of join** and **Vector Database 101** are explicitly assigned as pre-reading for Day 19, implying they are gaps relative to this deck's coverage. [p45]

## 5. Learning Priorities

**Essential**
- Why Data Lake lacks ACID and what "data swamp" symptoms result (four limitations) [p6-7]
- Definition of Lakehouse as data lake + table-format metadata layer; the four-layer stack [p8]
- The four ACID properties, specifically which ones object storage does NOT provide natively (A, C, I missing; only D present) and why [p13-15]
- The atomic-commit-file principle (single JSON/pointer object determines table content) as the foundation of ACID-on-object-storage [p16]
- Delta Lake structure: `_delta_log`, checkpoints, add/remove/metaData records, no-update/remove+add semantics [p18-19]
- Delta read process (checkpoint + replay + data skipping) and why it enables snapshot isolation [p20]
- Optimistic concurrency control and conflict handling (non-overlap auto-succeeds; overlap fails explicitly) [p21]
- Schema enforcement vs. evolution trade-off [p22]
- MERGE mechanics vs. full overwrite, and why it matters for CDC/late data [p23]
- Time travel core mechanism (versionAsOf/timestampAsOf/DESCRIBE HISTORY/RESTORE) and its three boundaries (retention, not-a-backup, storage cost) [p26-30]
- Fix-version-before-read pattern for reproducible ML training [p38]
- Point-in-time correctness / data leakage example [p39]
- Delta vs Iceberg: shared core guarantees vs. differing metadata/commit mechanism, and selection criteria (engine/catalog ecosystem, not "which is better") [p34-36]

**Important**
- OPTIMIZE/ZORDER/Liquid Clustering/VACUUM maintenance mechanics and trade-offs [p24]
- Problem→mechanism→result full mapping table (synthesis of Delta's value) [p25]
- MLflow data_version logging pattern for traceable model regressions [p29]
- Iceberg four-tier metadata tree (catalog/metadata.json/manifest list/manifest file) [p32]
- Hidden partitioning and partition evolution (Iceberg-specific capabilities) [p33]
- AI/ML vs BI requirements comparison and why Lakehouse is the default AI infra choice [p37]
- RAG table versioning rationale (embedding_model changes, doc_version) [p40]
- LLM inference log table requirements and ACID rationale [p41]
- Distinction between Lakehouse (architecture), Data Mesh (org model), Table format (spec) [p11]

**Supporting**
- S3-specific technical guarantees/non-guarantees detail and rejected workarounds (temp-dir-rename, `_SUCCESS` flag, per-day dirs, Redis/DynamoDB locks) [p15]
- Interoperability trend items: Delta UniForm, Apache XTable, Databricks/Tabular acquisition, AWS S3 Tables [p36]
- Apache Hudi passing mention [p35]
- Checklist items for "AI-ready lakehouse table" (useful as a practical recap, not new conceptual content) [p42]
- Demo/Lab logistics (Lab #18 deliverables, timing) [p43-44]
