# LEC_DAY01 — Quiz

## Q1 [priority: Essential] [weak-area]
**Question:** A spam filter learns to classify emails by training on 10,000 labeled examples (spam/not-spam) and adjusting its internal weights. A rule-based email filter uses hand-written rules like `if "free money" in subject → spam`. Which statement is correct?

- A) Both are Machine Learning because both classify emails
- B) Only the trained filter is ML; the rule-based filter is AI but not ML because its rules are hand-coded, not learned from data
- C) Neither is ML — both are just Discriminative AI
- D) The rule-based filter is ML because it produces the same output format (spam/not-spam label) as the trained one

**Answer:** B
**Explanation:** The defining criterion for ML is *learned* parameters from data, not the output format. The rule-based filter is AI (it performs an "intelligent" task) but not ML because a human wrote the rules. Option D is the exact misconception — conflating output format with mechanism. Option A ignores the fundamental difference in how each system derives its decision rules.

## Q2 [priority: Essential]
**Question:** In the lecture's AI taxonomy, what sits between Deep Learning and LLM?

- A) Agentic AI
- B) Machine Learning
- C) Generative AI
- D) Foundation Models

**Answer:** C
**Explanation:** The nested hierarchy is AI ⊃ ML ⊃ DL ⊃ **Generative AI** ⊃ LLM. Generative AI is the branch of DL that creates new content (text, images, video). LLMs are one type of generative model, specialized for language. Agentic AI (A) is a functional category, not a layer in the nesting. Foundation Models (D) is a related concept but not the term used in this taxonomy.

## Q3 [priority: Essential]
**Question:** The self-attention formula divides QK^T by √d_k before applying softmax. If you remove this scaling step entirely, what happens?

- A) The model becomes faster because there's one fewer computation
- B) Softmax produces a nearly uniform distribution, and the model treats all tokens as equally important
- C) Softmax becomes extremely sharp (near one-hot), causing vanishing gradients and the model struggles to learn
- D) The attention mechanism stops working entirely and produces NaN values

**Answer:** C
**Explanation:** Without scaling, large dot-product scores make softmax push nearly all weight to one token (e.g., [0.99, 0.005, 0.005]). This causes vanishing gradients — the model effectively stops learning because parameter updates become tiny. It also loses the ability to blend information from multiple tokens. Option B describes the opposite problem. Option D is too extreme — the computation still produces valid numbers, just poorly distributed ones.

## Q4 [priority: Essential]
**Question:** An LLM generates "Hà Nội là thủ đô của Việt Nam" with high confidence. According to the lecture, this confidence means:

- A) The statement is factually verified by the model
- B) The token sequence has high probability given the training data, but the model does not verify truth
- C) The model cross-referenced multiple sources to confirm accuracy
- D) The model's RLHF training ensures factual correctness

**Answer:** B
**Explanation:** LLMs optimize for next-token probability, not truth. The model predicts "Việt Nam" because it's the most probable continuation — not because it verified the fact. This is why hallucination is a *structural* property, not a bug: the model can state false information with equal confidence if the false sequence is probable. RLHF (D) improves alignment and safety but does not guarantee factual correctness.

## Q5 [priority: Essential]
**Question:** A plain LLM (no plugins, no tools) is classified as Level 0 in the lecture's agent maturity model. What is it missing to become a Level 1 "Connected Solver"?

- A) Better training data and a larger context window
- B) The ability to connect to external tools, retrieve data, and call APIs
- C) Multi-step planning and reasoning capabilities
- D) Coordination with other specialized LLM agents

**Answer:** B
**Explanation:** Level 0 has internal reasoning only. Level 1 adds tool connectivity — search, APIs, databases — allowing the model to access information beyond its training data. Option C describes Level 2 (Strategic Problem-Solver), and D describes Level 3 (Collaborative AI Agents). Option A would improve the base model but doesn't change its maturity level — it would still be a Level 0 engine without tools.

## Q6 [priority: Essential]
**Question:** A chatbot sends this per API call: system prompt (300 tokens), RAG context (800 tokens), chat history (variable), user question (50 tokens), and receives a 200-token response. After 15 turns of conversation (assuming ~100 tokens of history per turn), approximately how many input tokens is the API processing per call?

- A) 250 tokens (just user question + response)
- B) 1,150 tokens (system + RAG + user + base overhead)
- C) 2,650 tokens (system + RAG + 15 turns of history + user question)
- D) 3,450 tokens (system + RAG + history + user + all prior outputs)

**Answer:** C
**Explanation:** Input per call = system prompt (300) + RAG context (800) + chat history (15 × 100 = 1500) + user question (50) = 2,650 tokens. The user's 50-token question is less than 2% of the total input. This illustrates the lecture's key point: "Tối ưu chi phí = tối ưu prompt + context" — system prompt and chat history dominate cost, not the user's message.

## Q7 [priority: Important]
**Question:** Why does a Decoder-only Transformer (like GPT or Claude) need a causal mask during training?

- A) To reduce the computational cost of attention by skipping some token pairs
- B) To prevent the model from seeing future tokens it hasn't generated yet — without it, the model would "cheat" during training
- C) To handle variable-length input sequences by padding shorter ones
- D) To focus attention only on the most recent tokens and ignore distant context

**Answer:** B
**Explanation:** In autoregressive generation, the model must predict each token based only on previous tokens. Without the causal mask, token at position 3 could attend to tokens at positions 4, 5, etc. — seeing the "answers" before predicting them. The mask sets future positions to −∞ before softmax, forcing those attention weights to ≈0. Option D is wrong — self-attention's strength is precisely that it can attend to *any* position, including distant ones.

## Q8 [priority: Important]
**Question:** Output tokens cost 3–5x more than input tokens. A colleague suggests: "Let's switch from Sonnet ($15/M output) to Haiku ($4/M output) for our customer support bot to save money." When could this switch actually *increase* total cost?

- A) When customers write longer messages in Vietnamese
- B) When the cheaper model fails tasks, causing customers to retry repeatedly — each retry is another full API call
- C) When the system prompt is too long
- D) When the context window is too small

**Answer:** B
**Explanation:** A model that fails the task creates a hidden cost loop: bad answer → user retries → another API call → potentially still bad → retries again. Three failed Haiku calls cost more than one successful Sonnet call, and produce no value. This is the quality/latency/cost triangle in action — cutting cost without maintaining quality can backfire. Options A and C increase cost per call but affect both models equally. Option D would cause truncation, not increased cost from retries.

## Q9 [priority: Important] [weak-area]
**Question:** An API response returns `finish_reason: "length"`. What happened, and what is the correct fix?

- A) The model finished generating naturally — no action needed
- B) The model hit the `max_tokens` limit and was cut off mid-generation — increase `max_tokens` or ask the model to be more concise
- C) The model encountered a rate limit — retry with exponential backoff
- D) The model wants to call a tool — handle the tool call and continue

**Answer:** B
**Explanation:** `finish_reason: "length"` means the response was truncated at the `max_tokens` limit — the output is incomplete. The fix depends on the cause: if `max_tokens` was set too low, increase it; if the output is genuinely long, restructure the prompt for conciseness. Simply retrying with the same `max_tokens` will produce the same truncation. Option A describes `finish_reason: "stop"`, option C is a network/rate issue (not a finish_reason value), and option D describes `finish_reason: "tool_calls"`.

## Q10 [priority: Important]
**Question:** Positional Encoding is added to Input Embedding as X = E + P. What would happen if we skipped this step and fed only E (the embedding) into self-attention?

- A) The model would process tokens faster because there's less computation
- B) The model would treat "Con mèo ngồi trên bàn" and "Bàn ngồi trên con mèo" as identical — it has no sense of word order
- C) The model would only attend to adjacent tokens instead of all tokens
- D) The embeddings would be larger vectors, using more memory

**Answer:** B
**Explanation:** Embedding carries only token identity ("what word is this"), not position ("where does it stand"). The vector for "mèo" is identical whether it's word #1 or word #100. Without positional encoding, the model sees the same bag of tokens regardless of order — "The cat sat on the table" and "The table sat on the cat" would be indistinguishable. Self-attention would still attend to all tokens (C is wrong), and there's no computational or memory difference (A and D are wrong).

## Q11 [priority: Important]
**Question:** The lecture presents three principles of effective Vibe Coding. Which combination is correct?

- A) Write fast + Ship fast + Fix later
- B) Intent-driven + Context-first + Human review
- C) AI-generated + No debugging + Trust the output
- D) Describe vaguely + Let AI decide + Accept results

**Answer:** B
**Explanation:** The lecture's three principles are: (1) Intent-driven — state the goal and desired output clearly, (2) Context-first — provide files, examples, constraints, (3) Human review — AI writes fast, humans verify and finalize. Options A, C, and D describe the reckless version of vibe coding that the lecture explicitly argues against. The key distinction is that effective vibe coding requires the developer to remain a "designer + reviewer + AI orchestrator," not a passive recipient.

## Q12 [priority: Supporting]
**Question:** The LLM training pipeline has three stages: Pre-training, SFT, and RLHF/DPO. What happens if you try to apply RLHF directly to a raw pre-trained model, skipping SFT?

- A) It works better because RLHF is more powerful than SFT
- B) It fails because RLHF requires a model that already knows how to follow instructions (from SFT) — each stage depends on the previous one
- C) It works the same because RLHF and SFT are interchangeable
- D) The model becomes unsafe because SFT is the safety layer

**Answer:** B
**Explanation:** The pipeline is sequential and each stage builds on the previous: Pre-training teaches language/knowledge, SFT teaches *how to answer* instructions, and RLHF/DPO aligns to human preferences. A raw pre-trained model knows language but doesn't know how to respond to instructions — applying RLHF to it would be trying to refine a behavior (instruction-following) that hasn't been established yet. SFT is not the safety layer (D) — that's RLHF's role.
