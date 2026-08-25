# Core Knowledge

Source: `Day 07 Slides C401.pdf` — AICB-P1, Ngày 7: "Embedding & Vector Store" (55 physical PDF pages / 44 numbered slides, footer format `Tuần 1  N / 44`). All slide references below use the `N/44` numbering shown in the deck's footer.

Framing question posed at the start (Slide "HÃY SUY NGHĨ", before Slide 1): "Agent trả lời sai vì model yếu, hay vì nó không có đúng dữ liệu để suy luận?" (example: CS agent using GPT-4 answers refund policy incorrectly because the underlying data is the old 2023 policy). This framing question is explicit and sets up the lecture's central thesis (see 2.1).

## 1. Core Concepts

1. **Data strategy for AI products** (Slide 4-6) — Explicit. When an LLM/VLM already "knows" the answer, a simple question→model→answer loop suffices; this is rare in real products. When it doesn't, data must be supplied via three routes: In-Context (short data), RAG (large corpus), or Finetuning (needs custom style). Day 07 focuses on the RAG branch. Matters because product quality depends on which data-delivery route is chosen, not just model choice.

2. **Three data types an agent needs** (Slide 7) — Explicit.
   - *Knowledge Data*: documents, policy, SOP, FAQ, manuals, contracts, internal articles — fits retrieval.
   - *Operational Data*: databases, order status, tickets, CRM records, logs, transactions — usually needs controlled querying.
   - *Contextual Data*: session history, user profile/preferences, recent actions, channel context — should be injected briefly and at the right time.

3. **Data Quality Pyramid** (Slide 9) — Explicit, 4-tier model bottom-up: Raw → Cleaned → Structured → Enriched.
   - Raw: e.g. skewed PDF scan, OCR errors ("đổ1 trả"), HTML with junk tags.
   - Cleaned: corrected text ("đổi trả"), header/footer removed, Unicode normalized.
   - Structured: chunked by heading, tagged with source (e.g. `refund-v3.pdf`).
   - Enriched: tagged with `category:support`, `access:public`, `quality:verified`.
   Common mistake flagged: indexing raw data directly and expecting retrieval to self-correct.

4. **Data Ownership & Governance** (Slide 10) — Explicit checklist: who owns/updates the data; who can access it (ACL vs public-internal); how often re-indexing is needed for freshness; whether PII/sensitive fields must be masked before embedding. Explicit warning against indexing everything into the vector DB by default.

5. **Short-term vs Long-term Memory** (Slide 11) — Explicit.
   - Short-term: lives inside the context window, holds recent history/current task, cheap but fills up in tokens, fits short session logic.
   - Long-term: lives outside the context window (vector store, DB, profile store), must be actively retrieved, fits accumulated knowledge and selective user history.
   Key point (explicit): "the context window is not a vector store" — an agent only has long-term memory if there is an explicit retrieval/storage mechanism.

6. **Optimal data format for LLM input** (Slide 12) — Explicit comparison table (Markdown, HTML(clean), Plain Text, JSON/YAML) rated on token efficiency and structure preservation. Markdown is the recommended default — saves ~30–50% tokens vs equivalent HTML (no closing tags/attributes/class names) and LLMs are heavily trained on Markdown.

7. **Data processing pipeline (text)** (Slide 13) — Explicit pipeline: Scanned Image/PDF → OCR Engine → Raw Text → Clean & Structure → Markdown; for documents with existing text layers (Word, HTML), an alternate path skips OCR (Extract Text → Parse). Common mistake flagged: skipping the Clean & Structure step and feeding raw text directly into chunking, which pollutes chunks with header/footer junk, OCR errors, special characters, and degrades retrieval quality.

8. **Embedding (concept)** (Slide 14-16) — Explicit. Definition: an embedding model is a function that converts raw data (text, image, audio) into numeric vectors of the same dimensionality, placed in the same vector space so that distance between vectors measures semantic "closeness". Example given: 1536-dimension vectors for text, image, and audio inputs occupying the same comparable space. Intuition: humans instantly recognize "similar meaning" across language/format/pose differences; machines need this numeric representation to do the same. This is the foundation for semantic search, clustering, dedup, and recommendation.

9. **Cosine similarity vs Euclidean distance** (Slide 17-18) — Explicit.
   - Cosine similarity: measures the angle between two vectors; range -1 to 1, closer to 1 = more semantically similar; not affected by vector norm/magnitude; the dominant metric in NLP/retrieval. Formula: dot product of A,B divided by product of their norms.
   - Euclidean distance: straight-line distance between two points; smaller = closer; affected by scale/norm; formula generalizes to n dimensions (e.g. 1536) with the same sum-of-squared-differences form.
   Cosine is the default for text embeddings (compares meaning, ignores length); Euclidean fits cases needing absolute distance (image, geo).

10. **Embedding model use cases** (Slide 21) — Explicit table: Semantic search (embed query + docs, take highest cosine), Clustering (group nearby vectors, e.g. 10,000 CS tickets → 15 topics), Dedup (flag pairs above a cosine threshold), Recommendation (embed user intent + item, rank by cosine). Explicit caveat: embeddings don't create "truth" — if source text is wrong/incomplete or chunking is poor, retrieval stays poor ("garbage in, garbage out").

11. **Choosing an embedding model** (Slide 22) — Explicit comparison: `text-embedding-3-small` (1536-dim, MTEB avg 62.3, good for demo/PoC/lab — cheap, fast), `text-embedding-3-large` (3072-dim, MTEB avg 64.6, for production quality needs), open-source models e5/BGE (768-4096 dim, MTEB 61-66, for self-hosting / sensitive data / free). Explicit principle: no single "best" model for every use case — choose by quality, latency, storage, language; a balanced model often beats the strongest-but-slow-and-expensive one in real products.

12. **Vector store record structure** (Slide 24) — Explicit. Each stored record has 4 components: ID, Original chunk (source text), Embedding vector (float array), Metadata (key-value). Search results (top-k + scores) are query *output*, not something stored. Explicit distinction: the vector is used to find (semantic search); the chunk is used to inject into the LLM prompt — two different purposes stored together.

13. **Metadata in vector stores** (Slide 25) — Explicit fields and purposes: Source/filename (traceability, source display), Category (domain filtering before search), Time/freshness (avoid stale content), Access level (restrict access), Section/chunk id (precise debugging/citation). Explicit principle: good retrieval needs semantic similarity combined with metadata filtering.

14. **Query-time parameters: Top-k, Score threshold, Attribute filter** (Slide 26) — Explicit. Top-k = how many nearest chunks to retrieve (e.g. `n_results=3` for simple questions, `10` when synthesizing multiple sources). Score threshold = drop chunks below a cosine cutoff (e.g. 0.7 = only strongly relevant; 0.4 = accept loosely relevant). Attribute filter = filter by metadata before searching (e.g. `where={"category": "support"}`). Recommended flow: filter by category → semantic search → keep top-3 with score > 0.6.

15. **Two-phase retrieval pipeline** (Slide 28) — Explicit, this is the pipeline named as a course objective (Slide 2): Document → Chunk → Embed → Store.
   - Ingestion phase (runs once/offline, "Lab 7 scope"): Document (PDF, docs, HTML) → Chunk (split by section) → Embed (vectorize) → Store (index + metadata into vector store). Runs offline when data changes.
   - Retrieval phase (runs online, per question): Question (user asks) → Embed query (same model) → Search (cosine top-k + filter) → Inject (chunk → prompt → LLM). Runs online every time a user asks.
   Explicit: these two phases are kept separate.

16. **Chunking** (Slide 29) — Explicit. Definition: splitting a long document into smaller pieces to embed and index individually. Why: embedding models have token input limits; embedding an entire long file as one vector prevents finding a relevant passage (only the file, not the passage, becomes findable); smaller chunks give more precise retrieval and less noise injected. Common chunking strategies (explicit list): by heading/section (most common), by fixed token count (200–500 tokens), by sentence/paragraph.

17. **Chunk size trade-off** (Slide 30) — Explicit 3-way comparison: too-big chunks (>1000 tokens) mix multiple topics into one vector — retrieval may hit the right chunk but injects a lot of noise; well-sized chunks (200–500 tokens) hold one complete idea/section with 10–15% overlap between adjacent chunks; too-small chunks (<50 tokens) lose context, retrieve many disjointed fragments, hard to synthesize into a good answer. Overlap technique: carry the last 1-2 sentences of the previous chunk into the start of the next chunk to preserve context at the cut point. Rule of thumb (explicit): start simple with section/heading-based chunking, then optimize using evaluation rather than guessing.

18. **Retrieval vs Memory** (Slide 34) — Explicit distinction (see 3.6 below for full comparison).

19. **Advanced chunking methods** (Slide 36) — Explicit, named but not explained in depth (assigned as a group-research exercise, not lectured content): Recursive Character Splitting (LangChain-style separator hierarchy), Semantic Chunking (uses embeddings to find natural cut points), Document-structure Chunking (heading/table/list-aware, Markdown/HTML), Agentic Chunking (LLM decides chunk boundaries), Parent-Child / Small-to-Big (retrieve small chunk, inject larger parent chunk), Sliding Window + Overlap (fixed-size with stride, trades off coverage vs cost). See Section 4 — flagged as a gap since the deck only names/labels these, students research and present them.

20. **Hosted vs Self-managed vector store** (Slide 39) — Explicit. Hosted/managed (e.g. File Search): faster setup, less infra code, good for demos/PoC. Self-managed (Chroma/Faiss/vector DB): needed when you want control over chunking, metadata, pipeline, or cost details. Lab 7 explicitly uses Chroma (self-managed) to understand the end-to-end pipeline.

21. **Key takeaways** (Slide 42, explicit, verbatim 4 points): (1) Data quality usually matters more than switching to a more expensive model. (2) Embedding is the layer that translates language into a comparable meaning-space. (3) Vector store is long-term memory searchable by semantics and metadata. (4) The retrieval pipeline is the bridge from private data to the agent's grounded answers.

## 2. Relationships & Mechanisms

1. **Data quality/relevance vs model selection** (explicit, recurring theme, Slides 4-6, 21, 38, 42): the deck repeatedly asserts that data quality and retrieval quality determine real product experience more than switching to a stronger/pricier model. Framed via the Stanford AI Index 2025 stat citation (Slide 6) that AI/GenAI adoption rose sharply while the operative question shifted from "which model" to "what is the agent allowed to know".

2. **Core pipeline** (explicit, stated as a Day-07 objective on Slide 2 and detailed on Slide 28): `Document → Chunk → Embed → Store → Query → Inject`. This is the single mechanism the whole lecture builds toward; Ingestion (Document→Chunk→Embed→Store) is offline/one-time, Retrieval (Question→Embed query→Search→Inject) is online/per-query.

3. **Chunk quality → retrieval accuracy → LLM answer accuracy** (explicit, demonstrated Slide 33 and Slide 35 "Failure Demo"): same model, same query, but chunking strategy alone changes correctness. Bad/raw chunk retrieved cosine 0.61 → noisy/incomplete answer; well-structured chunk with metadata retrieved cosine 0.89 → accurate, sourced answer. Mechanism: chunk boundary choice directly determines what context is available to the LLM at answer time.

4. **Semantic similarity + metadata filtering = retrieval quality** (explicit, Slide 25-26): similarity search alone is not sufficient; filtering by metadata (category, freshness, access) before/alongside cosine search is presented as necessary for correct retrieval, not just a nice-to-have.

5. **Embedding model as prerequisite for semantic operations** (explicit/inferred): semantic search, clustering, dedup, and recommendation (Slide 21) all depend on embeddings existing in a shared vector space (Slide 15) — embedding is a prerequisite mechanism for all four use cases.

6. **Data type determines delivery mechanism** (explicit, Slide 7-8): Knowledge data → fits retrieval (RAG); Operational data → needs controlled querying (not raw retrieval); Contextual data → should be injected briefly and at the right time (not stored as a large corpus). This mapping is reinforced by the group discussion exercise on Slide 8.

7. **Memory requires an explicit retrieval/storage mechanism** (explicit, Slide 11): long-term memory is not automatic — the context window (short-term memory) is contrasted explicitly with vector store/DB/profile store (long-term memory), and the deck states an agent only "remembers long-term" when there is a retrieval or external storage mechanism in place.

8. **Data Quality Pyramid as a processing pipeline** (explicit/inferred, Slide 9 combined with Slide 13): the 4-tier pyramid (Raw → Cleaned → Structured → Enriched) can be read as a maturity/processing sequence a document moves through, paralleling the explicit OCR/Clean-and-Structure pipeline on Slide 13 (Scanned Image/PDF → OCR → Raw Text → Clean & Structure → Markdown). The deck does not explicitly merge these two diagrams into one pipeline — treating them as directly equivalent stages is inferred, not stated verbatim.

## 3. Examples & Distinctions

1. **In-Context vs RAG vs Finetuning** (Slide 5): three routes to supply data the LLM doesn't already know — In-Context (short data), RAG (large corpus), Finetuning (needs custom style/behavior). Distinguishing criterion: amount and nature of data needed. Day 07 scope = RAG branch only.

2. **Cosine similarity vs Euclidean distance** (Slide 17-19): similarity — both measure vector "closeness". Difference — cosine measures angle (direction, scale-invariant, range -1..1, dominant in NLP/text retrieval); Euclidean measures straight-line distance (scale-sensitive, dominant when absolute distance matters, e.g. image/geo). Worked example (Slide 19, in-class exercise): A=[1,2,3], B=[2,4,6] gives cosine similarity 1.0 even though B=2×A (different magnitude, same direction) — flagged explicitly by the deck as a discussion prompt about what this implies for cosine vs Euclidean, but the deck does not give the answer (see Section 4, gap).

3. **Cross-lingual/cross-modal similarity example** (Slide 20): "Chính sách hoàn tiền" vs "Quy định đổi trả" ≈0.87 (very close in meaning, same language); vs "Cách giao hàng nhanh" ≈0.52 (loosely related); vs "Thời tiết hôm nay" ≈0.31 (unrelated); vs "Refund policy" (English) ≈0.82 (cross-lingual — explicitly highlighted as the embedding model's power over plain keyword search, since "hoàn tiền" and "refund" are close despite different languages).

4. **Chunking granularity example** (Slide 33), same FAQ chunked 3 ways: too-big (all 3 Q&A pairs in one chunk) → query "đổi size" retrieves all 3, LLM gets noise from irrelevant Q1/Q3; well-sized (one Q&A per chunk) → query retrieves exactly Q2, correct answer; too-small (answer text split from its question) → query retrieves only "Trong 30 ngày" without context, LLM doesn't know what the 30 days refers to.

5. **Failure Demo: bad chunk vs good chunk** (Slide 35), same query ("Chính sách đổi trả áp dụng trong bao lâu?"), same model: raw/unsectioned chunk (cosine 0.61) → noisy/incomplete answer mentioning hotline instead of key details; section-structured chunk with metadata (cosine 0.89) → accurate, correctly grounded answer ("30 ngày kể từ ngày nhận, sản phẩm còn nguyên tem").

6. **Retrieval vs Memory** (Slide 34): similarity — both supply an agent with information beyond the immediate prompt. Difference — Retrieval finds context relevant to the *current question*, reads from a knowledge base, focused on relevance/grounding (example: user asks "chính sách đổi trả" → agent searches FAQ → answers from the document). Memory stores state/preferences/selective history from a user profile or interaction history, focused on continuity/personalization (example: same user asks a second time → agent recalls they previously exchanged for size M → proactively offers to change size again). Explicit note: many systems use both — retrieval for domain facts, memory for who the user is and what they've done before.

7. **Short-term vs Long-term Memory** (Slide 11): see Core Concept 5 — full distinguishing table (location relative to context window, cost, persistence mechanism, use case fit).

8. **Hosted vs Self-managed vector store** (Slide 39): distinguishing criterion = need for control. Hosted trades control for speed (demo/PoC); self-managed trades setup effort for control over chunking/metadata/pipeline/cost.

9. **3 data types example prompts** (Slide 8, discussion hints, not fully resolved in the deck itself — used to test the Knowledge/Operational/Contextual classification): "Đơn hàng 12345 đang giao" (operational-leaning), "Bảng size áo" (knowledge-leaning), "User này hay mua size M" (contextual-leaning) — deck poses these as discussion prompts without stating the "correct" classification (see Section 4, gap).

## 4. Assumptions, Boundaries & Gaps

1. **Unresolved in-class question on cosine scale-invariance** (Slide 19) — the deck explicitly asks "Cặp 1 có cosine = 1.0 mặc dù B=2×A. Vì sao? Điều này nói gì về cosine similarity so với Euclidean distance?" as a discussion prompt but does not provide the explanation in the slide text itself. Flagged gap — students/tutor must reason this out (cosine ignores magnitude/scale, only measures direction), not sourced verbatim from the deck.

2. **3-way data type classification exercise unresolved** (Slide 8) — hint questions ("Đơn hàng 12345 đang giao" / "Bảng size áo" / "User này hay mua size M") are posed without the deck stating definitive answers; this is intentionally left as a group discussion, not explicit content.

3. **Advanced chunking methods named but not explained** (Slide 36-37) — Recursive Character Splitting, Semantic Chunking, Document-structure Chunking, Agentic Chunking, Parent-Child/Small-to-Big, Sliding Window+Overlap are only named/labeled with a one-line description each; the deck assigns them as a group-research exercise (students read external docs/blogs/papers) rather than teaching the mechanics. Flagged gap — do not treat these one-liners as full explanations.

4. **MTEB benchmark referenced but not defined** (Slide 22) — "MTEB Avg" scores are given for embedding models, and "MTEB Leaderboard on Hugging Face" is referenced, but the deck never defines what MTEB measures or how the scores are computed. Flagged gap.

5. **Common mistakes explicitly flagged as boundaries/failure modes**:
   - Indexing raw data directly into the vector DB and expecting retrieval to self-correct data quality issues (Slide 9).
   - Skipping the Clean & Structure step before chunking, producing chunks polluted with header/footer junk, OCR errors, special characters (Slide 13).
   - Indexing "everything" into the vector DB by default without ownership/governance review (Slide 10).
   - Embeddings do not manufacture truth — poor source text or poor chunking still yields poor retrieval ("garbage in, garbage out", Slide 21).

6. **Governance prerequisites are stated as checklist items, not fully resolved processes** (Slide 10): data ownership, access control, freshness/re-indexing cadence, and PII masking are named as questions to answer per-deployment; the deck gives one worked example (FAQ owned by CS, public-internal, re-indexed weekly, no PII) but does not generalize a full governance process.

7. **No single "best" embedding model** (Slide 22, explicit assumption/boundary): model choice is presented as a trade-off across quality, latency, storage, and language — not a solved/ranked problem; a "balanced" model is explicitly recommended over the top-scoring model for real products.

8. **Two-phase pipeline separation is an explicit design constraint** (Slide 28): Ingestion (offline) and Retrieval (online) are kept deliberately separate — this is stated as a boundary/architectural rule, not just a description.

9. **Python code example has extraction-quality caveats** (Slide 23) — the source PDF's text layer has broken Vietnamese diacritics in this code/comment block (e.g. "Chính sách hoàn � tin", "Quy định đ� i � tr") due to a font/combining-character extraction artifact, not a content gap; the underlying meaning is recoverable from context and inline comments (`~0.87 hoàn tiền <-> đổi trả`, `~0.31 hoàn tiền <-> thời tiết`). Flagged as an extraction-quality note, not treated as garbled/missing content requiring vision fallback — no core concept is affected.

10. **Discussion-exercise source document (Slide 31) has the same diacritic-extraction artifact** — the sample "raw CS policy document" used for the chunking exercise is legible in structure/intent (contact info, 3 numbered policy sections, footer) despite scattered broken diacritics; not re-verified via vision since it's a discussion prop, not a definitional slide.

## 5. Learning Priorities

**Essential**
- Central thesis: data quality/relevance often matters more than model selection (Slides 4-6, 21, 38, 42).
- Core pipeline: Document → Chunk → Embed → Store → Query → Inject, and its two phases (Ingestion offline / Retrieval online) (Slides 2, 28).
- Embedding definition and mechanism: text/image/audio → vector in shared space, distance = semantic closeness (Slides 14-16).
- Cosine similarity vs Euclidean distance — definitions, formulas, when each is used (Slides 17-19).
- Chunking: definition, why it's needed, size trade-offs (too big / good / too small), overlap technique (Slides 29-30, 33, 35).
- Vector store record structure (ID, chunk, vector, metadata) and the chunk-vs-vector distinction (Slide 24).
- Metadata's role combined with similarity search; Top-k / score threshold / attribute filter (Slides 25-26).
- Retrieval vs Memory distinction (Slide 34).

**Important**
- Three data types (Knowledge/Operational/Contextual) and their delivery mechanism mapping (Slide 7).
- Data Quality Pyramid (Raw → Cleaned → Structured → Enriched) (Slide 9).
- Data processing pipeline for text (OCR → Clean & Structure → Markdown) and why Markdown is preferred (Slides 12-13).
- Short-term vs Long-term memory (Slide 11).
- Embedding model use cases: semantic search, clustering, dedup, recommendation (Slide 21).
- Embedding model selection trade-offs (dimension, MTEB score, cost/latency, self-host option) (Slide 22).
- Data ownership & governance checklist (Slide 10).
- Cross-lingual embedding example and its implication for search vs keyword matching (Slide 20).
- Failure Demo (bad vs good chunk) as a concrete illustration of the central thesis (Slide 35).

**Supporting**
- Chroma code walkthrough specifics (add/query/inject syntax) (Slide 27).
- Hosted vs Self-managed vector store trade-off (Slide 39).
- Lab 7 deliverable/assessment/checklist structure (Slides 38, 40-41).
- Named-but-unexplained advanced chunking methods list (Slide 36) — supporting only until researched further; treat as pointers, not explained content.
- References list (Slide 43) and Day 08 preview (Slide 44).
- Stanford AI Index adoption statistics (Slide 6) — supporting motivational data, not a mechanism.
