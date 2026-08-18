# LEC_DAY01 — AI & LLM Foundation

## Keypoints
- AI ⊃ ML ⊃ DL ⊃ Generative AI ⊃ LLM — nested taxonomy, with GenAI as a distinct layer
- Three AI groups: Discriminative (Input→Label), Generative (Prompt→Content), Agentic (Goal→Plan→Action)
- Agent = Goal + Reasoning + Tools + Memory + Action; LLM alone is Level 0 (no tools, no action)
- Transformer pipeline: Tokens → Embedding + Position (X=E+P) → Self-Attention (Q/K/V, scaled dot-product) → Multi-Head → FFN → Next Token
- Scaling by √d_k prevents softmax from going sharp → avoids vanishing gradients
- Masked (causal) attention enforces left-to-right generation in decoder-only models
- Next-token prediction: LLM optimizes probability, not truth → hallucination is structural, not a bug
- Autoregressive chaining: one wrong token shifts the entire distribution for all following tokens
- Token Economy: Cost = input tokens × price + output tokens × price; output costs 3-5x more
- Vietnamese/code/structured text consume more tokens than plain English
- System prompt + RAG context + chat history dominate input cost, not the user's message
- Model selection: start cheap, upgrade only when quality blocks the use case
- Vibe Coding: Intent-driven + Context-first + Human review; developer becomes designer/reviewer/orchestrator

## Terms
- LLM (Mô hình ngôn ngữ lớn) — large language model based on Transformer, trained on trillions of tokens
- Transformer — 2017 architecture using self-attention; foundation of modern LLMs
- Token (Token) — smallest unit an LLM processes; subword unit, ~0.75 words EN, ~0.5 words VI
- Embedding (Nhúng) — vector representation of a token's meaning via learned lookup table
- Positional Encoding (Mã hóa vị trí) — position vector added to embedding so model knows token order
- Self-Attention (Cơ chế tự chú ý) — mechanism letting each token attend to all others via Q/K/V
- Query, Key, Value (Q, K, V) — three projections of input: Q asks, K advertises, V supplies content
- Softmax — converts scores to probability distribution (0-1, rows sum to 1)
- Causal Mask (Che dấu nhân quả) — prevents decoder from seeing future tokens during generation
- Multi-Head Attention (Đa đầu chú ý) — parallel attention heads each learning different relationship types
- Next-Token Prediction (Dự đoán token tiếp theo) — autoregressive generation, one token at a time
- Temperature (Nhiệt độ) — controls sampling randomness; 0=deterministic, 1=diverse
- Hallucination (Ảo giác) — model confidently stating false information; structural property of probability-based generation
- Knowledge Cutoff (Giới hạn kiến thức) — model has no awareness of events after training data cutoff
- Context Window (Cửa sổ ngữ cảnh) — max tokens processable in one API call
- Pre-training — learns language/knowledge from massive raw text
- SFT (Supervised Fine-Tuning) — learns how to answer from labeled examples
- RLHF/DPO — alignment to human preferences and safety
- RAG (Retrieval-Augmented Generation) — mentioned but not explained; covered later in course
- Vibe Coding — writing software by describing intent; AI generates code; human reviews
- finish_reason — API response field: stop (complete), length (truncated), tool_calls (agent action)

## Covered / To revisit
- [x] AI taxonomy (AI⊃ML⊃DL⊃GenAI⊃LLM) — solid after GenAI layer correction
- [x] Three AI groups (Discriminative/Generative/Agentic) — understood
- [x] AI Agent definition and maturity levels — strong, spotted missing observation loop
- [x] Transformer pipeline overview — solid
- [x] Embedding + Positional Encoding — understood the why
- [x] Self-Attention Q/K/V mechanism — solid understanding including scaling rationale
- [x] Masked attention and Multi-Head attention — understood
- [x] Next-Token Prediction and hallucination — strong, identified chain effect
- [x] LLM training pipeline (Pre-training → SFT → RLHF) — covered
- [x] LLM limitations (cutoff, hallucination, context window) — solid
- [x] Token Economy and cost optimization — good after guided discovery of hidden cost drivers
- [x] Model selection framework — understood cost/quality tradeoff including retry costs
- [x] API call anatomy — understood finish_reason implications
- [x] Vibe Coding — reconciled personal practice with lecture's framework

## Misconceptions / Examiner findings
- **ML boundary** — initially claimed an if-else bot could be ML because discriminative AI "is technically if-else based on percentage." Gap: conflated the output format (binary decision) with the mechanism (learned vs hand-coded rules). Corrected: ML requires learning from data by adjusting parameters; hand-coded rules are AI but not ML.
- **Vibe Coding** — initial understanding was the reckless version (don't know/care about logic, just tell AI to fix errors). Through discussion, recognized their own practice (brainstorm, review plans, diagnose root causes) actually aligns with the lecture's disciplined version. Gap was labeling, not practice.

## Session 1 — 2026-08-17
- Full lecture covered in one session
- Student has practical experience with AI-assisted coding (uses AI agents, reviews plans, diagnoses issues)
- Strongest areas: attention mechanism understanding, identifying structural properties (hallucination, cost drivers)
- Homework assigned: Lab #1 (API comparison), read "Attention Is All You Need", try 3 API prompts
