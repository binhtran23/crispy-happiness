# Core Knowledge

Source: `day08-rag-pipeline-v3.pdf` (AICB-P1 · Ngày 8 · "RAG Pipeline: Retrieval — Augmentation — Generation") — 57 physical PDF pages / 46 numbered slides (title, section-divider, and intro pages carry no slide number). Slide refs below use the footer numbering (e.g. "p.6/46") as printed on the deck.

## 1. Core Concepts

- **RAG (Retrieval-Augmented Generation)** — technique combining information retrieval with language generation so the LLM answers from real external data instead of relying only on training memory. Formal definition given explicitly. (p.6/46)
- **R — Retrieval** — retrieving information from an external data store (vector store, DB, search engine); goal = find the right evidence, filter noise, rank by relevance. (p.6/46)
- **A — Augmentation** — enriching the prompt: packaging retrieved context with structure, attaching metadata, separating evidence from the question; goal = reduce noise and counter "lost in the middle". (p.6/46)
- **G — Generation** — producing an answer that stays close to the augmented evidence, with citations; goal = grounded answer, citation, self-check. (p.6/46)
- **Ingestion vs Retrieval pipeline** — Ingestion (offline, batch/near-real-time) builds the index from documents; Retrieval (runtime, per query) uses that index to answer. Explicit: "ingestion quality determines the ceiling of retrieval quality." (p.3/46)
- **Dense (semantic) search** — query/doc embedded as dense vectors (768–1536 dims, all dims ≈ nonzero), compared via cosine similarity/dot product; strong at meaning/paraphrase/cross-lingual, weak at exact keywords (error codes, proper nouns). (p.12–13/46)
- **Sparse (lexical) search, BM25/TF-IDF** — vector length = dictionary size, mostly zeros; counts term frequency; strong at exact term match, fast, no GPU needed; weak at synonyms/paraphrase. (p.12, 14/46)
- **Hybrid Search** — runs dense (vector) and sparse (keyword) search in parallel on the same query, merges results via a fusion algorithm; compensates for each method's blind spots. (p.15/46)
- **RRF (Reciprocal Rank Fusion)** — fusion method using rank position only (ignores raw scores); default constant k=60; simple, no tuning needed — recommended as a starting point. (p.16/46)
- **Alpha Weighting (score fusion)** — `score(d) = α·s_dense + (1−α)·s_sparse`; uses normalized raw scores; requires domain tuning (e.g., chatbot FAQ α=0.7–0.9; code/log/law α=0.2–0.4). (p.16/46)
- **Reranking (Cross-encoder)** — second-stage scoring where query+document are concatenated into one input and read jointly by a reranker model; more accurate than bi-encoder search but slower, so applied only to a small candidate list. (p.17/46)
- **MMR (Maximum Marginal Relevance)** — selects the next chunk that is both relevant to the query and maximally different from already-selected chunks, to reduce redundancy. Formula: `MMR = Sim(d, q) − λ·Sim(d, d_selected)`. (p.18/46)
- **Context Injection** — technique for ordering/formatting retrieved chunks inside the LLM's context window; explicitly not just string concatenation — formatting affects whether the LLM "respects" the data. (p.19/46)
- **Prompt structuring (System/Context/Question separation)** — using XML tags/Markdown headers/delimiters to clearly separate system instructions, retrieved context ("objective evidence"), and the user question. (p.20/46)
- **Lost in the Middle & Document Reordering** — LLMs recall information at the start/end of a prompt better than the middle; important chunks should be reordered (e.g., top result placed first/near boundaries) to avoid being "lost". (p.21/46)
- **Grounding & Verification techniques** — Strict Constraints (answer only from provided context, else say "not enough information"), Metadata Integration (attach date/author/department so LLM can prefer newer/authoritative sources), Citation Formatting (require numbered citations like [1],[2] for verifiability). (p.22/46)
- **Token Budget Management** — context window treated as a limited "cake": rule of thumb ≈ system prompt 20% / retrieved context ≤60% / headroom (query+output) 20%. If context still too long after top-K selection, use **Context Compression** (small model condenses chunks to key points; tools named: LLMLingua, Recomp). (p.23/46)
- **LLM Selection** — choosing between large models (GPT-4, Gemini Pro: complex reasoning, multi-source comparison, higher cost/latency) vs small/local models (Llama 3, Mistral: simple lookup, sensitive/on-prem data, lower cost/latency). (p.24/46)
- **Output Control** — Formatting (Markdown/Table/JSON output for system integration) and Safety & Alignment (policy compliance, no unwanted PII leakage, access-level checks before returning). (p.25/46)
- **Self-Correction loop** — LLM evaluates its own answer against the context before showing it to the user; retries or abstains if it detects reasoning beyond the evidence. Flow: Context + Generate Answer → Self-Check vs Context → Final Answer (retry if not grounded). (p.26/46)
- **Grounding & Abstention** — Strict Grounding (forbid using training knowledge, only provided context — even if the context is wrong, the model must answer per that context); Forcing Citations (RAG "loses half its value" if the model doesn't cite sources); Graceful Degradation (refuse rather than guess when the answer isn't in context, and suggest next steps instead of a bare "I don't know"). Called "the two most important production RAG features" (grounding + abstention). (p.27/46)
- **UX for RAG** — users only see output, not internals (MMR, HNSW); Citations, Confidence indicators, and Streaming (progress shown during 3–5s waits) turn the AI from a "black box" into a trustworthy tool. (p.28/46)
- **Chain-of-Thought (CoT) in Generation** — instructing the model to "think on scratch paper" (e.g., in a `<thought_process>` tag): filter relevant sentences → analyze → synthesize final answer. Useful for comparison/logical/multi-source questions; not needed for simple lookups. (p.29/46)
- **Generation Failure Patterns** — Context conflict (contradictory documents → model confused/guesses/merges; fix: prefer most recent doc or list both and flag conflict), Over-inference (model extrapolates an unstated condition → hallucination; fix: instruct "don't infer conditions not explicitly stated"), Ignoring constraints (model forgets to cite despite instruction, common with small models/long context; fix: place the critical rule near the end of the prompt, closest to "Answer:", and lower temperature to 0). Diagnostic rule: log the `formatted_context` — if context contains the answer but output is wrong → Generation bug; if context lacks the answer → Retrieval bug. (p.30/46)
- **Pre-Filtering** — filter the full index (N chunks) by metadata (department, date/validity, doc type, access level) to a smaller subset (n≪N) before running dense/hybrid search; improves both speed and precision. (p.31/46)
- **Query Transformation — Multi-Query** — LLM auto-generates multiple paraphrased variants of the query, each searched separately, results merged; increases recall by catching chunks the original query missed. (p.32/46)
- **Query Transformation — HyDE (Hypothetical Document Embeddings)** — LLM generates a hypothetical/fake answer to the query, embeds that fake answer, and searches using that vector (instead of the query's own embedding). (p.32/46)
- **Query Processing — Query Expansion** — fixes typos, adds synonyms/domain terms to a single search; increases recall. (p.33/46)
- **Query Processing — Query Decomposition** — splits a complex multi-hop question into multiple sub-questions, searches each in parallel, merges context; rationale: one vector can't answer two distinct intents at once. (p.33/46)
- **Query Processing — Step-Back Prompting** — when a question is too detailed/narrow, the LLM generates a more abstract "step-back" question first to retrieve broader context, avoiding getting "lost" in minutiae. (p.33/46)
- **Agentic RAG — Self-Query** — agent decomposes a complex question into smaller questions, searches multiple times, then synthesizes results. (p.34/46)
- **Agentic RAG — Corrective RAG (C-RAG)** — agent evaluates the quality of retrieved documents itself; if documents are judged "garbage," it auto-triggers a web search or reports an error. (p.34/46)
- **Agentic RAG — Adaptive RAG** — agent chooses its strategy based on difficulty: easy questions answered directly, hard questions trigger deeper/more thorough search. (p.34/46)
- **Agentic RAG — Tool Use** — agent can call external APIs (not just read the static vector store), e.g. eDocman (document status), JIRA (latest tickets), calculator, code interpreter, database query; combines results from multiple tools. Described as extending "R" from vector search only to any callable data source. (p.35/46)
- **RAGAS (RAG Assessment) Triad** — evaluation framework asserting RAG cannot be scored with one number; must separate errors by pipeline stage: Context Recall (Retriever), Faithfulness (Generator), Answer Relevance (whether the answer matches user intent — attributed implicitly to Augmentation/Generation quality). (p.37/46)
- **Context Recall** — does the retriever bring back enough necessary evidence to answer the question? Measured as the proportion of ground-truth-needed info found in retrieved context. Low recall → fix retrieval strategy (hybrid/rerank), chunking, query transformation, or embedding model. (p.38/46)
- **Faithfulness** — does the answer fully stick to the retrieved evidence, or invent extra content? Measured by splitting the answer into individual claims/statements and checking each has supporting evidence; unsupported claim = hallucination. Low faithfulness → fix generation prompt (stricter grounding), add self-correction loop, lower temperature to 0, move critical rules to the end of the prompt. (p.39/46)
- **Answer Relevance** — does the answer address the actual question, without going off-topic or being too short/long? Measured (per slide) by generating a reverse question from the answer and comparing to the original question. Low relevance → fix prompt instructions (format/scope), augmentation (context ordering/compression), or model selection (bigger model for complex questions). (p.40/46)
- **RAG Triad diagnosis table** — combinations of (Context Recall, Faithfulness, Answer Relevance) map to a specific pipeline fix: all high → system healthy; Recall low/Faithfulness high/Relevance low → fix Retrieval; Recall high/Faithfulness low/Relevance high → fix Generation; all low → fix Indexing (source data problem); Recall high/Faithfulness high/Relevance low → fix Augmentation (context correct but packaged wrong). (p.41/46)
- **ROI of RAG** — framing technique improvements as a cost/quality tradeoff: e.g. adding a cross-encoder reranker raised Answer Relevance +5% but increased latency 1s→4s and doubled server cost; not every technique is worth adding. (p.43/46)
- **CI/CD for RAG Evaluation** — integrate RAGAS into CI (e.g. GitHub Actions): every config change triggers automated eval; example threshold "Faithfulness < 80% → block deploy." Framed as treating RAG eval like unit tests for AI. (p.44/46)

## 2. Relationships & Mechanisms

- **Ingestion → Retrieval dependency**: Ingestion (offline: Document → Processing [chunk+embed] → Store) creates the index; Retrieval (runtime) depends on and is capped by that index's quality. (p.3, 11/46)
- **RAG pipeline (Input → Process → Output)**: `Question → Retrieval → Augmentation → Generation → Answer`. Explicit end-to-end runtime flow, paired with the offline `Document → Ingestion → Vector Store` flow. (p.11/46)
- **Day 07 → Day 08 upgrade path**: Day 07's retrieval pipeline (dense search only, threshold, top-K, raw context injection, no rerank/no eval) is explicitly upgraded into the Day 08 RAG pipeline: R (search + rerank) → A (structured context packaging) → G (grounded generation + citation), plus Evaluation (faithfulness, relevance, recall). (p.4/46)
- **Hybrid search mechanism**: User Query → [Vector Engine ∥ Keyword Engine] (run in parallel) → Fusion → Top K. Dense and sparse are not competitors but complementary — each catches what the other misses. (p.15/46)
- **Retrieve-and-Rerank two-stage mechanism**: Stage 1 — broad/fast bi-encoder search over the full corpus → Top 50–100 (noisy). Stage 2 — cross-encoder rereads and rescores that shortlist → Top 3–5 passed to the LLM. Trade-off: speed (stage 1) vs accuracy (stage 2), justified because retrievers are "broad but shallow" while cross-encoders are precise but too slow to run over millions of docs. (p.17/46)
- **Token budget constraint**: as retrieved context approaches/exceeds the ~60% share of the context window, less room remains for instructions and output, degrading quality — this motivates context compression when top-K context is still too long. (p.23/46)
- **Self-correction as a retry mechanism**: Generate Answer → Self-Check vs Context; if not grounded, the loop retries generation (or presumably abstains) rather than returning the ungrounded answer. (p.26/46)
- **Generation failure triage mechanism**: log `formatted_context`; presence/absence of the answer in that logged context is the diagnostic branch point between a Generation-layer bug and a Retrieval-layer bug. (p.30/46)
- **Recommended query-processing pipeline order**: Pre-Filter (metadata) → Query Expansion (synonyms) → Decomposition (split multi-hop) → Step-Back (generalize) → Multi-Query (variants) → HyDE (simulate answer) → Search. Presented as a suggested ordering, not a mandatory sequence for every query. (p.33/46)
- **RAGAS triad as a diagnostic framework**: the three scores are read jointly (not independently) to localize which pipeline stage (Indexing / Retrieval / Augmentation / Generation) is failing — see diagnosis table in Core Concepts. (p.41/46)
- **RAG spectrum**: In-Context (data directly in prompt, small data, fast) vs RAG (large corpus, continuously updated) vs Fine-tuning (retrain model, needed for domain-specific style, more expensive) — three approaches positioned as a spectrum by data volume and need for style/domain adaptation, not mutually exclusive alternatives. (p.10/46)

## 3. Examples & Distinctions

- **Dense vs Sparse search**: similarity — both are retrieval methods represented as vectors over a corpus. Difference — dense understands meaning/paraphrase/cross-lingual but misses exact keywords; sparse matches exact terms/codes but not synonyms. Distinguishing criterion: does the query need semantic understanding (dense) or exact term match (sparse)? Failure examples given: dense search on "lỗi ERR-4012" returns generic "system error" docs instead of the exact error code (p.13/46); sparse search on "nghỉ phép" misses docs that only say "annual leave"/"PTO" (p.14/46).
- **RRF vs Alpha Weighting**: similarity — both are hybrid-search fusion methods. Difference — RRF uses only rank position (k=60 default, no tuning); Alpha Weighting uses normalized raw scores and requires domain-specific tuning of α. Distinguishing criterion: use RRF to start simply; use Alpha Weighting when you need explicit control over the dense/sparse balance. (p.16/46)
- **Bi-encoder (semantic search) vs Cross-encoder (reranker)**: similarity — both score query-document relevance. Difference — bi-encoder embeds query and document separately and compares only the final distance (very fast, scans millions); cross-encoder concatenates query+document as one input and reads jointly (more accurate, much slower, so used only on short lists). (p.17/46)
- **In-Context vs RAG vs Fine-tuning**: distinguishing criterion — data volume and need for domain/style adaptation. In-Context = small data, injected directly; RAG = large corpus, retrieved+injected, continuously updatable; Fine-tuning = requires retraining, needed when a custom style/domain is required, more expensive. (p.10/46)
- **Example — Reranking value**: query "Thủ tục xin visa" (visa procedure) — the correct document ("the steps to apply") initially ranks 8th, not 1st, illustrating why top-K alone (without rerank) is insufficient. (p.17/46)
- **Example — MMR redundancy**: three retrieved chunks all restate "refund takes 7 days" in different wording — wastes LLM tokens without adding new information, motivating MMR's diversity selection. (p.18/46)
- **Example — Context conflict**: one doc says 12 days of leave (2024), another says 14 days (2026) — model should prefer the most recent doc or list both and flag the conflict, rather than guessing/merging. (p.30/46)
- **Example — Over-inference/hallucination**: doc states "free shipping for orders ≥500k in Hanoi"; user asks about Ho Chi Minh City; model wrongly infers the same rule applies there. (p.30/46)
- **Example — Case study, Hybrid Search ROI**: V1 (dense only) — Context Recall 60%, misses error codes/proper nouns/ticket numbers, average faithfulness due to incomplete context. V2 (hybrid BM25+vector) — Context Recall 90% (+30 points), catches exact terms via BM25, faithfulness rises accordingly. Framed as "the highest-ROI change for most RAG systems." (p.42/46)
- **Example — ROI trade-off**: adding a cross-encoder reranker: Answer Relevance +5%, latency 1s→4s, server cost roughly doubles — framed as a judgment call for the lead engineer, not an automatic win. (p.43/46)
- **Query Expansion vs Query Decomposition vs Step-Back Prompting**: distinguishing criterion — Expansion adds synonyms to one search (fixes vocabulary mismatch, e.g. "hoàn tiền" → "refund"/"trả hàng"/"chargeback"); Decomposition splits a question with two distinct intents into separate searches (e.g. "Shopee vs Tiki hoàn tiền?" → two sub-queries); Step-Back abstracts an overly narrow/detailed question into a broader one to get context first (e.g. "Lỗi 404 API Momo user 8910" → "Kiến trúc tích hợp Momo?"). (p.33/46)

## 4. Assumptions, Boundaries & Gaps

- **Prerequisite**: the lecture assumes learners already built the Day 07 indexing/retrieval pipeline (chunking, embedding, dense search, threshold, top-K) and treats Day 08 as a direct upgrade of it. (p.1–4/46)
- **Hallucination explanation is explicitly hedged as incomplete**: slide notes "there could be other reasons too" beyond knowledge cutoff and probabilistic/fluency-over-accuracy generation — not presented as an exhaustive causal account. (p.5/46)
- **Reranking constraint**: cross-encoders are explicitly limited to small candidate lists because of speed, not applied to the full corpus — this is a stated architectural boundary, not a general trade-off left to the reader to infer. (p.17/46)
- **Token budget rule is heuristic**: "context ≤ 60% of token budget" is explicitly labeled a "rule of thumb," not a fixed constraint. (p.23/46)
- **Gap — Context Compression mechanics**: tools LLMLingua and Recomp are named as options for compressing chunks when context is still too long, but the deck does not explain how they work internally. (p.23/46)
- **Gap — Self-correction retry logic**: the self-correction loop is described conceptually (retry if not grounded) but retry limits, fallback behavior, or exact re-generation mechanics are not detailed. (p.26/46)
- **Gap — Agentic RAG techniques are only sketched**: Self-Query, Corrective RAG (C-RAG), and Adaptive RAG each get a 1–2 sentence description with no implementation detail (e.g., how "document quality" is evaluated in C-RAG, or the decision rule in Adaptive RAG). Flagged as underspecified in the source. (p.34/46)
- **Gap — Agentic RAG tool selection**: example tools (eDocman, JIRA, calculator, code interpreter, DB query) are listed, but the mechanism by which the agent decides which tool to call, or how results from multiple tools are combined, is not explained. (p.35/46)
- **Gap — RAGAS scoring methodology**: Context Recall, Faithfulness, and Answer Relevance are defined conceptually (what to measure, how to improve if low) but the exact scoring computation/algorithm is not given in the slides beyond the stated method sketches (e.g., "split answer into statements and check each"). (p.37–40/46)
- **Illustrative, not universal thresholds**: the ROI example (reranker: +5% relevance / 4x latency / 2x cost) and the CI/CD example ("Faithfulness < 80% → block deploy") are presented as illustrative decision points for engineers to weigh, not confirmed as universal standards applicable beyond this course's examples. (p.43–44/46)
- **Explicit scope boundary**: Multi-Agent systems, MCP (Model Context Protocol), and A2A (agent-to-agent) communication are named as the topic for the *next* lecture, explicitly out of scope for Day 08's RAG pipeline content. (p.45/46)
- **When RAG is/isn't needed** is stated as explicit criteria (not inferred): needed when internal data changes frequently, answers need verifiable sources, fine-tuning is too costly/data too sensitive, or hallucination must be systematically reduced; not needed for general-knowledge questions, tasks without a private corpus, tasks not requiring citation (creative writing/brainstorming), or when simple prompt engineering already suffices. (p.8/46)

## 5. Learning Priorities

**Essential**
- RAG definition and the R–A–G framework (p.6/46)
- Ingestion vs Retrieval distinction; end-to-end pipeline flow Question→Retrieval→Augmentation→Generation→Answer (p.3, 11/46)
- Dense vs Sparse search, and why Hybrid Search combines them (p.12–15/46)
- Reranking: bi-encoder vs cross-encoder, retrieve-and-rerank mechanism (p.17/46)
- RAGAS Triad (Context Recall, Faithfulness, Answer Relevance) and the diagnosis table mapping score combinations to pipeline fixes (p.37–41/46)
- Grounding & Abstention as core production RAG requirements (p.27/46)
- Context injection / structured prompting (System / Context / Question separation) (p.19–20/46)

**Important**
- RRF vs Alpha Weighting fusion methods (p.16/46)
- MMR for reducing redundancy (p.18/46)
- Lost in the Middle & document reordering (p.21/46)
- Token budget management and context compression (p.23/46)
- Self-correction loop (p.26/46)
- Generation failure patterns and fixes; retrieval-vs-generation diagnostic logging (p.30/46)
- Query transformation techniques: Multi-Query, HyDE, Query Expansion, Decomposition, Step-Back Prompting (p.32–33/46)
- Pre-filtering by metadata (p.31/46)
- Hybrid search case study (Context Recall 60%→90%) as evidence of ROI (p.42/46)
- CI/CD integration for RAG evaluation (p.44/46)

**Supporting**
- LLM selection: large vs small/local models (p.24/46)
- Output control: formatting and safety/PII filtering (p.25/46)
- UX design for RAG (citations, confidence indicators, streaming) (p.28/46)
- Chain-of-Thought in generation (p.29/46)
- Agentic RAG: Self-Query, Corrective RAG, Adaptive RAG, and tool use (p.34–35/46)
- ROI/cost-quality tradeoff framing (p.43/46)
- Business case for RAG (compliance, data freshness, IP/cost of fine-tuning) (p.8–9/46)
- In-Context vs RAG vs Fine-tuning comparison (p.10/46)
