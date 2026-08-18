# Core Knowledge

**Source:** `1-AICB_Ngày_1.pdf` — "AI & LLM Foundation" (AICB-P1 · Ngày 1 · Nền tảng), VinUniversity, Giảng viên: Huỳnh Thành Trung. 78 pages processed (67 numbered content slides + title/section/closing slides). Lecture delivered in Vietnamese; terms below preserve original Vietnamese/English mix as used in source.

Lecture roadmap (p.1): 1) Bức tranh AI 2026, 2) LLM — Trái tim của AI hiện đại, 3) Token Economy, 4) Gọi API lần đầu, 5) Vibe Coding, 6) Thực hành.

Stated learning objectives (p.2, explicit): understand how LLMs work (Transformer, token, next-token prediction); estimate API call cost via token economy; use third-party LLMs (OpenAI, Anthropic) or self-hosted open models; adopt the "Vibe Coding" mindset; build a simple chatbot with streaming response. Stated prerequisites: Python 3.10+, VS Code/Cursor, OpenAI API key.

---

## 1. Core Concepts

### AI Taxonomy (p.3) — explicit
Nested hierarchy: **AI** ⊃ **Machine Learning (ML)** ⊃ **Deep Learning (DL)** ⊃ **Generative AI** ⊃ **LLM**.
- AI: "Máy thực hiện tác vụ 'thông minh'" (machines performing "intelligent" tasks).
- ML: learns from data without explicit rule programming.
- DL: multi-layer neural networks.
- Generative AI: advanced branch that creates new content (text, image, video) resembling human output.
- LLM: a Foundation Model specialized for language — the foundation of GenAI and Agentic AI.
- Course focus (explicit, highlighted red on p.3): "Khóa học này tập trung vào LLM → xây dựng Agentic AI" (this course centers on LLM → building Agentic AI).

### Three main AI groups (p.4) — explicit
| Group | Function | Examples | Pattern |
|---|---|---|---|
| Discriminative AI | Classify/predict | Spam filter, image classifier, fraud detection | Input → Label |
| Generative AI | Create new content | ChatGPT, Claude, DALL-E, Midjourney, GitHub Copilot | Prompt → Content |
| Agentic AI | Plan & act autonomously | AI coding agents, auto customer support, research agents | Goal → Plan → Action |

LLM is stated as the "common engine" (engine chung) underlying both Generative AI and Agentic AI. Course journey (explicit, p.4): LLM Foundation → Agent → Multi-Agent → Deploy → Evaluate.

### AI Agent (p.7–9) — explicit
**Definition (formula, p.9):** Agent = Goal + Reasoning + Tools + Memory + Action.
Components: Goal (receives an objective instead of a single prompt), Reasoning (multi-step analysis/planning), Tools (search, API, database, code), Memory (stores state/history), Action (executes actions in a system).

**Why agents are needed (p.8):** a static prompt only answers one question (Prompt → Response, one step; no access to new data; cannot act). An AI Agent pursues a complete goal (Goal → Plan → Action; connects to APIs/DB/tools; handles multi-step workflows; produces real-world value/ROI).

**Maturity levels, "Từ LLM đến AI Agents" (p.7)** — explicit:
- Level 0 — Core reasoning engine: LLM reasons from its own internal knowledge, no external tools.
- Level 1 — Connected Solver: LLM becomes an agent, connects to external tools, retrieves data, searches, calls APIs.
- Level 2 — Strategic Problem-Solver: agent plans multi-step, uses multiple tools and chains of reasoning for complex problems.
- Level 3 — Collaborative AI Agents: multiple specialized LLM agents coordinate to solve complex problems.

### Historical timeline (p.5) — explicit
Perceptron (1957) → Deep Learning bùng nổ/boom (2012) → Transformer (2017) → ChatGPT (2022) → AI Agents (2024–26).

### Large Language Model (LLM) (p.11) — explicit definition
"Mô hình ngôn ngữ lớn dựa trên kiến trúc Transformer, được huấn luyện trên lượng dữ liệu văn bản khổng lồ (hàng nghìn tỷ token). LLM có khả năng sinh văn bản, trả lời câu hỏi, viết code, và thực hiện reasoning phức tạp." (Large language model based on Transformer architecture, trained on massive text data — trillions of tokens; capable of generating text, answering questions, writing code, and complex reasoning.)
Key characteristics: Decoder-only Transformer architecture; Self-supervised pre-training + RLHF fine-tuning; Next-token prediction; Emergent capabilities appear at scale.

### Transformer architecture (p.12–13) — explicit
Revolutionary 2017 architecture. Core components (decoder-only stack, p.12): Input Tokens → Embedding + Position → Self-Attention → Feed-Forward Network → (repeated Nx layers) → Next Token Prediction.
- Self-Attention: each token "looks at" all other tokens in the context.
- Multi-Head: multiple parallel "viewpoints."
- Feed-Forward: nonlinear per-position processing.
- Residual connections: allow gradients to flow easily.

Two main architecture families (p.12–13, explicit): **Encoder-Decoder** (BERT, T5) — understands bidirectional context, used for classification/translation; **Decoder-only** (GPT, Claude, Gemini) — reads left→right to predict the next token and generate text. Note (explicit): "Ngày nay Decoder-only thắng thế nhờ scale tốt hơn" (Decoder-only dominates today due to better scaling).

### Input Embedding (p.14–15) — explicit
Transformers process data in parallel, not sequentially. Input embedding tells the model "what word is this" but not "where does it stand" — if the model only looks at the bag of tokens, it treats "Con mèo ngồi trên bàn" and "Bàn ngồi trên con mèo" (reordered) as identical.
Formula: `E_token = LookupTable(token)`. E is a vector representing word meaning (learned from a dictionary/embedding table). Embedding carries only identity, not position/coordinates — the vector for "mèo" is the same whether it's word #1 or #100 in the sentence.

### Positional Encoding (p.16) — explicit
Positional Encoding (P) creates an independent position vector, added directly to Input Embedding (E).
Formula: **X = E + P**, where E = content embedding, P = positional encoding, X = final input fed to self-attention.
Note (explicit): "Positional encoding không thay đổi nội dung, nó cộng hợp vị trí vào ngữ nghĩa" (positional encoding doesn't change content — it adds position information into the semantics).

### Self-Attention (p.17–24) — explicit
**Purpose:** human language is inherently ambiguous; a single word alone often lacks full meaning. Attention is the mechanism letting a model "pay attention" to the most relevant parts of the input when processing each token.
Example (p.17): "Con mèo ngồi trên bàn. Nó rất đáng yêu." When processing "Nó" (it), attention assigns high weight to "mèo" (cat), correctly resolving that "Nó" refers to the cat, not the table.

**Q, K, V (Query, Key, Value)** (p.18) — explicit analogy: self-attention is like a "key-matching" system. A token's Query is compared against every other token's Key to retrieve the corresponding Value.
- Q (Query): "what am I looking for / what do I need?"
- K (Key): "what characteristics do I have to help others' queries?"
- V (Value): "if Q and K match, this is the actual content I supply."
Formulas: Q = XW^Q, K = XW^K, V = XW^V, where W are learned linear-layer weight matrices transforming input X into Q, K, V.

**Scaled Dot-Product Attention formula** (p.19–24), explicit — the core equation:
`Attention(Q, K, V) = softmax(QK^T / √d_k) V`
Four-step process (Input → Process → Output):
1. **Chấm điểm (Attention Scoring):** compute QK^T. Each token "asks" every other token (including itself) → produces a raw score matrix. Why it matters: higher dot-product = more related tokens.
2. **Ổn định (Scaling):** divide by √d_k (d_k = dimension count) to dampen overly large scores and prevent instability. Why it matters: without scaling, large dot products make softmax too "sharp" (near one-hot), causing vanishing gradients and the model to stop learning; scaling spreads the distribution more smoothly and keeps gradients stable.
3. **Chuẩn hóa (Softmax):** compress scores into weights between 0 and 1, each row summing to 1 → yields the attention map.
4. **Trộn thông tin (Weighted Sum):** multiply attention weights by V to aggregate information from other tokens according to their computed importance.

**Worked numeric example** (p.25–30, explicit): sequence [Tôi][yêu][AI], Q=K=V = [[1,0],[1,1],[0,1]], d_k=2. Walks through QK^T → scaling by √2 → softmax → output O=AV. Demonstrates per-token attention distribution (e.g., row for [Tôi] attends 0.4011/0.4011/0.1978 to [Tôi]/[yêu]/[AI] respectively; [yêu] attends most to itself while also drawing from subject [Tôi] and object [AI]; [AI] draws heavily from [yêu] and itself, less from [Tôi]).

### Masked Self-Attention (p.31) — explicit
Problem: without a mask, a token (e.g., row 1) could see all columns, including future tokens — not allowed in Decoder-only generation. Mechanism: add a Mask matrix (M) containing −∞ values before softmax; softmax(−∞) forces attention weight to ≈0. Uses a **causal mask** — each token can only attend to itself and prior tokens (e.g., [Tôi]→sees only [Tôi]; [yêu]→sees [Tôi],[yêu]; [AI]→sees all). Why it matters (explicit): "Khi dạy AI tự động sinh văn bản (Decoder-only), nó phải che (mask) các từ chưa được sinh ra để chống gian lận" — masking prevents the model from "cheating" by seeing future tokens during autoregressive generation training.

### Multi-Head Attention (p.32) — explicit
Instead of learning only one type of relationship, Multi-Head Attention splits attention into multiple parallel heads. Each head learns a different perspective on the same sentence (example given: Head 1 = pronoun/coreference resolution expert; Head 2 = spatial/positional-action relations expert; Head 3 = syntax expert, attends to spacing/punctuation). Results from all heads are concatenated then passed through a Linear Projection (W^O) to produce a richer final representation.

### Token & Tokenization (p.33, 37) — explicit
**Definition:** Token = the smallest unit an LLM processes — roughly 0.75 words for English, ~0.5 words for Vietnamese. LLMs do not read "words," they read subword units.
Examples: "Hello world" → 2 tokens; "Xin chào" → 3–4 tokens; "def func():" → 4 tokens; "anthropic" → 3 tokens (subword splitting).
Note (explicit, important/highlighted): Vietnamese consumes more tokens than English due to diacritics/Unicode characters and finer word-splitting → higher API cost for equivalent content ("Tôi yêu Việt Nam" needs more tokens than "I love Vietnam").
Tokens are used to: compute API cost, limit context window, measure prompt length, determine latency.

### Next-Token Prediction (p.34) — explicit
LLM does **not** "understand" language — it predicts the token with the highest probability, one at a time (example: "Hà Nội là thủ đô của" → predicts "Việt Nam" with p=0.94).
- **Temperature:** controls "creativity" (0 = deterministic, 1 = more random).
- **Autoregressive:** the output token becomes the input for the next step.
Important caveat (explicit, highlighted): LLM can confidently state incorrect information (**hallucination**) because it optimizes for probability, not truth.

### How LLMs are built — 3-stage pipeline (p.35) — explicit
1. **Pre-training** — "reads the Internet," learns language and knowledge.
2. **SFT (Supervised Fine-Tuning)** — learns from examples "how to answer correctly."
3. **RLHF / DPO** — aligned to human preferences, made safer.
Memory analogy given (explicit): Pre-training = "read a lot," SFT = "taught how to answer," RLHF/DPO = "shaped to behave better."

### Inherent LLM limitations (p.36) — explicit
- **Knowledge cutoff:** the model doesn't know events after its training cutoff unless given additional data/tools.
- **Hallucination:** model can answer very confidently but wrongly, because it optimizes token probability, not truth.
- **Context window:** model can only "see" a limited number of tokens per call; too-long context increases cost, and information in the middle can be forgotten.
- Analogy (explicit): LLM is like "a scholar who has read a great deal" but lives in a time bubble and can only see a few pages in front of them at once.
Presenter note on slide (explicit): these limitations explain why good prompting, context management, RAG, and tools are needed (RAG mentioned only as a name here — not explained further in this deck; flagged as a gap, see Section 4).

### Token Economy / API pricing (p.37–41, 45–46) — explicit
**Cost formula:** Input tokens + Output tokens = Cost. Priced per 1 million tokens (1M tokens). Output tokens cost more than input tokens (3–5x). Prices drop ~10x per year (example given: GPT-4-level pricing $20/M → $2/M over 2 years).
**Why some content costs more tokens** (p.38): Vietnamese (Unicode, finer word-splitting); Code (special characters, whitespace); structured text (JSON, URL, long IDs). Rule of thumb (explicit): Unicode + special characters + complex structure → more tokens.
**Prompt length drives cost** (p.40): input tokens dominate cost when system prompt repeats every call, RAG context is long, or chat history grows. Example calculation: user question 50 + system prompt 300 + RAG context 800 + output 200 = 1350 tokens/call. Conclusion (explicit): "Tối ưu chi phí = tối ưu prompt + context."
**Latency vs Cost trade-off** (p.41): longer context, longer output, and larger models all increase latency; more input tokens, more output tokens, and pricier models increase cost. Key insight (explicit): "Nhiều tokens hơn → vừa chậm hơn vừa đắt hơn" (more tokens → both slower and more expensive).

### Model comparison table (p.42, dated 03/2026) — explicit
| Model | Input $/M | Output $/M | Context | Type | Best for |
|---|---|---|---|---|---|
| Claude Opus 4.6 | $5.0 | $25 | 1M | Closed | Reasoning, hard code |
| Claude Sonnet 4 | $3.0 | $15 | 1M | Closed | Balanced choice |
| Claude Haiku 4.5 | $0.8 | $4 | 200K | Closed | Fast, cheap, routing |
| GPT-4o | $5.0 | $20 | 128K | Closed | Multimodal, ecosystem |
| Gemini 2.5 Pro | $1.25 | $10 | 1M | Closed | Long-context tasks |
| Llama 4 Scout | Free | Free | 1M | Open | Self-host, private data |
Note: Closed = API hosted; Open = self-host / more control.

### Framework for quick model choice (p.43) — explicit
- Prioritize cost/latency → FAQ, simple classification/extraction, large batch jobs, short low-reasoning answers → suggested: Haiku, Gemini Flash, small/open-source models.
- Prioritize quality/reasoning → multi-step analysis, code, planning, long documents/complex context, high-reliability tasks → suggested: Sonnet, Opus, GPT-4o, Gemini Pro.
Rule of thumb (explicit): "bắt đầu từ model đủ tốt và đủ rẻ. Chỉ nâng model khi chất lượng thực sự chặn use case" (start with a model that's good enough and cheap enough; only upgrade when quality truly blocks the use case).

### Context Window (p.45) — explicit
Definition: the max number of tokens an LLM can process in one API call (input + output combined).
- 128K tokens ≈ one 300-page book; 1M tokens ≈ 4–5 books.
- Longer context → higher cost.
- Information in the middle of context can be "forgotten" (**Lost in the Middle** — named phenomenon, not further explained; flagged as gap).
Context window sizes (p.45, explicit): Gemini 2.5 = 1M, Claude Sonnet 4 = 1M, Claude Opus 4.6 = 1M, GPT-4o = 128K, Llama 4 = 1M.

### API Call anatomy (p.47, 50) — explicit
**Flow (p.47):** Prompt → API Call → Token Stream → Response.
- Prompt: system + user input + context.
- API Call: sends request to model provider.
- Token Stream: model generates output chunk by chunk.
- Response: complete text + usage + stop reason.
Note (explicit, framed as the right mindset for PM/engineer): every API call always has three things to control simultaneously: **quality, latency, cost**.

**Request fields (OpenAI GPT-4o example, p.50):** `model` (model used), `messages` (conversation input, list of role:system/user/assistant), `max_tokens` (output length limit), `temperature` (optional creativity control).
**Response fields:** `choices[0].message.content` (answer content), `model` (actual model used), `usage.prompt_tokens` (input tokens), `usage.completion_tokens` (output tokens), `usage.total_tokens` (total tokens), `finish_reason` (stop | length | tool_calls).

### Output-control parameters (p.51–52) — explicit
| Parameter | Range | Meaning |
|---|---|---|
| `temperature` | 0–1 | "Creativity" degree; 0 = deterministic; 1 = diverse. Use low for code/analysis, higher for creative tasks. |
| `top_p` | 0–1 | Nucleus sampling — only choose from tokens comprising top p% probability mass. Typically 0.9–0.95. |
| `stop_sequences` | — | Stops generation at a specified string; useful for fixed-structure output or cutting at a desired point. |
Recommendation (explicit): start with `temperature=0` for tasks needing stability; only increase sampling when diversity is genuinely needed.

### Vibe Coding (p.59–64) — explicit
**Definition (p.59):** "Viết phần mềm bằng cách mô tả ý tưởng AI sẽ generate code" (writing software by describing an idea; AI generates the code). Characteristics: not writing code from scratch; describing requirements in natural language; AI generates code; developer reviews and edits.
**Why needed (p.60):** manual coding is slow, boilerplate repeats. AI writes code faster; focus shifts to logic instead of syntax. Key line (explicit): "Viết phần mềm nhanh hơn không phải viết code nhiều hơn" (building software faster, not writing more code).
**Workflow (p.61):** Idea → Prompt → Code → Test → Refine (describe the problem, AI generates code, quick test run, refine prompt, iterate multiple times, finalize result).
**Mindset shift (p.62):** Traditional programming (think algorithm first, write code step by step, debug manually, optimize performance, lots of boilerplate) vs Vibe Coding mindset (describe clear goals, AI generates code, edit via prompting, review logic carefully, iterate fast repeatedly). New mindset (explicit): developer shifts from "code writer" to "designer + reviewer + AI orchestrator."
**3 principles of Vibe Coding (p.63):** 1) Intent-driven — state the goal and desired output clearly; 2) Context-first — provide context (files, examples, constraints); 3) Human review — AI writes fast, humans verify and finalize. Explicit conclusion: "Vibecoding hiệu quả khi ý định rõ ràng, ngữ cảnh đầy đủ và luôn có bước rà soát cuối" (effective Vibe Coding requires clear intent, sufficient context, and always a final review step).
**Prompt quality contrast (p.64):** poor prompt ("Write a chatbot using OpenAI") → vague goal, no specific requirements, no output criteria, AI produces generic code. Good prompt ("Build a CLI chatbot using OpenAI" + explicit requirements: conversation memory, streaming response, exit with "quit", show token usage) → clearer output, easier to use, less rework needed.

---

## 2. Relationships & Mechanisms

### Model / Framework: Transformer forward pass (Input → Process → Output)
Input Tokens → **Embedding + Position** (X = E + P) → **Self-Attention** (Q,K,V → scaled dot-product → softmax → weighted sum) → **Multi-Head** concat + linear projection → **Feed-Forward Network** → (repeat N layers via residual connections) → **Next Token Prediction**.
Why each stage matters: embedding gives meaning but not order; positional encoding restores order information; self-attention lets each token gather relevant context from all others; scaling stabilizes gradients; masking (in decoder-only) enforces causality for generation; multi-head lets different relationship types be captured in parallel; residual connections keep gradients flowing through deep stacks.

### Model / Framework: LLM training pipeline
Pre-training (learn language/knowledge from massive raw text) → SFT (learn correct answering style from labeled examples) → RLHF/DPO (align to human preference/safety). Dependency relationship: each stage builds on the prior — SFT presupposes a pre-trained base model; RLHF/DPO presupposes an SFT model that already knows how to follow instructions.

### Model / Framework: API call lifecycle
Prompt (system+user+context) → API Call (request with model, messages, max_tokens, temperature) → Token Stream (chunked generation, esp. with `stream=True`) → Response (content + usage + finish_reason). Trade-off triangle to manage simultaneously: quality, latency, cost — these are in tension (e.g., more tokens/bigger model raises quality but increases both latency and cost; see Token Economy section).

### Model / Framework: Vibe Coding loop
Idea → Prompt → Code → Test → Refine (iterative cycle). Depends on: Intent-driven prompting + Context-first inputs; gated by Human review at the end of each iteration.

### Relationship: AI Taxonomy → Agent maturity
LLM (innermost taxonomy layer) is the "engine" that, when given Tools/Memory/Reasoning/Action wrapped around it, becomes progressively more capable per the Level 0→3 model (p.7): bare reasoning engine → tool-connected solver → multi-step strategic planner → multi-agent collaboration. This is presented as both a taxonomy (static) and a maturity/capability progression (dynamic) built on the same underlying LLM.

### Relationship: Token count drives cost, latency, and context-window pressure
More tokens (from prompt length, chat history, RAG context, or Vietnamese/code/structured text needing more tokens per unit of content) → higher cost (input+output pricing) AND higher latency AND faster consumption of the fixed context window (risking "Lost in the Middle" forgetting). This is a central trade-off mechanism spanning Sections 2–3 of the lecture (Token Economy).

### Relationship: Temperature/top_p control determinism vs diversity
Both parameters govern next-token sampling behavior on top of the model's predicted probability distribution (from Next-Token Prediction mechanism): temperature scales the distribution's "peakedness"; top_p restricts sampling to a cumulative-probability nucleus. Lower values favor deterministic/reliable output (recommended default: temperature=0); higher values favor creative/diverse output.

---

## 3. Examples & Distinctions

### Encoder-Decoder vs Decoder-only (A vs B)
- **Similarity:** both are Transformer-based architectures using self-attention, feed-forward layers, and positional encoding.
- **Difference:** Encoder-Decoder (e.g., BERT, T5) reads context bidirectionally, suited for classification/translation. Decoder-only (e.g., GPT, Claude, Gemini) reads left-to-right only, suited for predicting the next token and generating text.
- **Distinguishing criterion:** whether the model can see future tokens during processing (bidirectional context vs causal/masked left-to-right). Explicit note: "Decoder-only thắng thế nhờ scale tốt hơn" — decoder-only has become dominant due to better scaling behavior.

### Discriminative AI vs Generative AI vs Agentic AI (three-way distinction, p.4)
- Similarity: all are categories of AI systems, all can be built on LLMs/neural nets.
- Difference: Discriminative = classify/predict existing labels (Input → Label); Generative = create new content from a prompt (Prompt → Content); Agentic = pursue a goal via planning and action (Goal → Plan → Action).
- Distinguishing criterion: the nature of the output — a label, novel content, or an executed action sequence.

### Prompt (static) vs AI Agent (p.8)
- Similarity: both take some form of instruction as input and can use an LLM.
- Difference: a static prompt handles one question in one step, cannot access new data or act; an agent pursues a complete goal, connects to APIs/DB/tools, handles multi-step workflows, and produces real-world value.
- Distinguishing criterion: presence of goal-directed multi-step planning + tool/action capability vs single-turn response.

### Poor prompt vs Good prompt (Vibe Coding, p.64)
- Example given: "Write a chatbot using OpenAI" (vague, no criteria) vs "Build a CLI chatbot using OpenAI" with explicit requirements (conversation memory, streaming response, exit command, token usage display).
- Distinguishing criterion: specificity of goal, requirements, and output criteria — directly affects code quality and rework needed.

### Worked numerical example — Self-Attention on [Tôi][yêu][AI] (p.25–30)
Clarifies the abstract Q/K/V/softmax mechanism with concrete 2-dimensional vectors, showing step-by-step how each token's attention weights are computed and interpreted (e.g., "yêu" attends most strongly to itself while drawing from subject and object; "AI" draws heavily from "yêu" and itself).

### Cost example — customer support chatbot (p.46)
Scenario: 1000 chats/day, 500 input tokens/output... input avg 500 tokens/turn, output avg 200 tokens/turn.
- Claude Sonnet 4: Input 500K×$3/1M=$1.50 + Output 200K×$15/1M=$3.00 = $4.50/day (~$135/month).
- Claude Haiku 4.5: Input 500K×$0.8/1M=$0.40 + Output 200K×$4/1M=$0.80 = $1.20/day (~$36/month).
- Illustrates the Framework for model choice: Haiku for simple tasks, Sonnet/Opus for complex reasoning — a concrete ~3.75x cost difference for the same volume.

### Same prompt, three models, three styles (p.44)
Prompt: "Tóm tắt báo cáo tài chính Q1 trong 3 bullet và nêu 1 rủi ro chính." Claude: coherent, structure-oriented, "consulting style." GPT-4o: concise, natural, flexible, suited to app/chat, versatile. Gemini: strong with long context/many documents, suited to multi-file workflows. Distinguishing point (explicit): model selection is not just about price but also style + task fit.

### Anthropic vs OpenAI API syntax (p.53)
Code-level distinction shown side by side: OpenAI uses `client.chat.completions.create(...)` and reads `resp.choices[0].message.content`; Anthropic uses `client.messages.create(..., max_tokens=1024)` and reads `resp.content[0].text`. Distinguishing criterion: different client libraries/response object shapes despite similar conceptual API call flow.

---

## 4. Assumptions, Boundaries & Gaps

### Prerequisites (explicit, p.2, 48)
- Python 3.10+ installed.
- VS Code or Cursor IDE.
- OpenAI API account with credit; `OPENAI_API_KEY` environment variable.
- Google Colab account.
- Course-level prerequisite: this session assumes no prior deep-learning background is required for the taxonomy overview, but the Transformer/attention math (Sections on Self-Attention) assumes comfort with basic linear algebra (matrix multiplication, dot product) — inferred from the level of formula detail presented without derivation from first principles.

### Constraints / limitations stated explicitly
- LLM knowledge cutoff — no awareness of events after training data cutoff without external tools/data.
- Hallucination — confident but potentially false output, because the model optimizes token probability, not truth.
- Context window — hard limit on tokens per call; cost rises with context length; risk of "Lost in the Middle" (information in the middle of a long context can be forgotten).
- Vietnamese and structured/code text cost more tokens than plain English for equivalent content.
- Output tokens cost 3–5x more than input tokens.

### Concepts named but not explained in this deck (flagged gaps — explicit mention, no elaboration provided)
- **RAG (Retrieval-Augmented Generation)** — mentioned only as a term on p.36 ("nhấn mạnh các giới hạn này giải thích vì sao cần prompt tốt, context management, RAG và tools") as a mitigation for LLM limitations; not defined or explained anywhere in this deck.
- **"Lost in the Middle"** — named phenomenon on p.45 (info in the middle of long context can be forgotten) but not explained mechanistically.
- **Emergent capabilities** — mentioned as a key LLM characteristic (p.11: "Emergent capabilities xuất hiện khi scale lên") but not elaborated on what specific capabilities emerge or why.
- **Self-supervised pre-training** — named as a characteristic (p.11) but the mechanism itself is not detailed beyond the 3-stage pipeline overview (p.35).
- **Multi-Agent systems / Adaptive Multi-Agent Systems** — named in the "Future of AI Agents" list (p.10) and as Level 3 of the agent maturity model (p.7), but mechanisms of coordination are not explained — flagged as forward-looking/future-course content (per p.66, "Multi-Agent" is stated as a later course topic, not covered here).
- **Deploy, Evaluate** stages of the course journey (p.4) — named as future roadmap items, not covered in this lecture.
- **Prompt Engineering** — explicitly deferred to "Ngày 2" (Day 2) per the closing slide (p.66): "Prompt tốt tạo ra output tốt. Nhưng 'tốt' nghĩa là gì?" — this lecture does not teach prompt engineering techniques in depth (only briefly touches good vs poor prompt in Vibe Coding context, p.64).
- **Self-hosting open models** — one code example given (Qwen3-0.6B-Base via `transformers` library, p.57) but no explanation of hardware requirements, trade-offs, or when self-hosting is preferable beyond the one-line note "Self-host, private data" in the model comparison table (p.42).

### Edge/failure cases mentioned
- Hallucination as a systematic failure mode of next-token prediction (not a rare bug but an inherent property of probability-based generation).
- Vanishing gradient risk if scaling (division by √d_k) is omitted from attention — explicitly explained as the reason scaling is a required step, not optional.
- Masking omission risk: without causal masking, a decoder-only model would "cheat" by seeing future tokens during training — explicitly framed as something that "must" be prevented.

### Assigned homework / follow-up (explicit, p.65–66)
- Lab #1: call the real OpenAI API — compare GPT-4o vs GPT-4o-mini on latency, cost, quality; deliverable is a complete Python script with streaming chatbot and comparison table; time budget 90 minutes.
- Reading assignment: Vaswani et al. (2017) "Attention Is All You Need" (arXiv:1706.03762) — before Day 2.
- Try calling the API with 3 different prompts.
- Other cited references (not required reading, listed as general references, p.67): Ouyang et al. (2022) "InstructGPT/RLHF" (arXiv:2203.02155); Rafailov et al. (2023) "DPO" (arXiv:2305.18290); Karpathy (2023) "State of GPT" talk; Anthropic/OpenAI/Google API quickstarts.

---

## 5. Learning Priorities

### Essential
- LLM definition and the Decoder-only/Transformer foundation (p.11–13).
- Self-Attention mechanism end-to-end: Q/K/V, scaled dot-product formula, 4-step process (scoring → scaling → softmax → weighted sum) (p.17–24).
- Next-Token Prediction, autoregressive generation, temperature, and hallucination as a structural property (p.34).
- Token definition and the cost formula (Input tokens + Output tokens = Cost) (p.33, 37, 39).
- API call anatomy: Prompt → API Call → Token Stream → Response; request/response fields; the quality/latency/cost triangle (p.47, 50).
- Agent = Goal + Reasoning + Tools + Memory + Action, and why agents differ from static prompts (p.8–9).
- Key Takeaways as stated by the lecturer (p.65, explicit summary): (1) LLM = Transformer predicting next token from context; (2) usable LLM requires Pre-training → SFT → alignment; (3) model choice trades off quality/latency/cost; (4) every API call has prompt, response, usage, and stop reason; (5) effective Vibe Coding = clear intent + sufficient context + careful review.

### Important
- Positional Encoding and why X = E + P is needed (p.14–16).
- Masked Self-Attention / causal masking mechanism (p.31).
- Multi-Head Attention concept and rationale (p.32).
- Three-stage LLM training pipeline: Pre-training → SFT → RLHF/DPO (p.35).
- Inherent LLM limitations: knowledge cutoff, hallucination, context window (p.36).
- Token economy mechanics: why Vietnamese/code/structured text cost more; prompt length driving cost; latency-cost trade-off (p.38, 40–41).
- Model comparison table and the cost/latency vs quality/reasoning selection framework (p.42–43).
- Context window concept and size comparisons across models (p.45).
- Vibe Coding definition, workflow (Idea→Prompt→Code→Test→Refine), 3 principles, and mindset shift (p.59–64).
- AI Taxonomy nested hierarchy (AI⊃ML⊃DL⊃GenAI⊃LLM) and the three main AI groups (Discriminative/Generative/Agentic) (p.3–4).

### Supporting
- Historical timeline (Perceptron 1957 → ... → AI Agents 2024-26) (p.5).
- "Why 2024-2026 is a turning point" statistics (78% adoption, $15.7T GDP, 3.7x ROI) (p.6).
- Future of AI Agents list (Generalist AI, Deep Personalization, Embodied AI, Agent-driven Economy, Adaptive Multi-Agent Systems) (p.10) — forward-looking, not core mechanism content.
- Same-prompt-three-models style comparison demo (p.44).
- Code examples for calling OpenAI API, wrapper functions, reading token usage, chatbot loop, streaming, and self-hosting a local model (p.49, 54–58) — practical/lab material rather than conceptual content.
- Output-control parameter details (`top_p`, `stop_sequences`) beyond `temperature` (p.51–52).
- Reference list (Vaswani et al., Ouyang et al., Rafailov et al., Karpathy talk) (p.67) — supporting citations, not lecture content itself.
