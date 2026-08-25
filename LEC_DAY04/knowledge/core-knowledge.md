# Core Knowledge

Source: `day04-prompt-engineering-tool-calling.pdf` (78 PDF pages; internal slide numbering runs 1/64–64/64, since several pages — title, section dividers, agenda — carry no footer number). Page references below use the internal slide numbers where visible (e.g. "slide 21/64"); pages without a footer number are referenced as "(divider/no number)".

## 1. Core Concepts

- **Prompt as interface** — Explicit. Definition: the prompt is the interface between human intent and model behavior/capability; tool calling is the interface between the model and the external world. Why it matters: framing for the whole lecture. Related: RTCF, system prompt, tool calling. (slide 2/64, restated slide 4/64)

- **Specificity beats cleverness** (golden principle) — Explicit. A short but precise prompt usually outperforms a long, rambling one. Intuition: clarity of task/context/constraint/format drives usable output more than "clever" phrasing. (slide 4/64)

- **RTCF framework** (Role, Task, Context, Format) — Explicit. Four components of a good prompt: Role (vai trò), Task (nhiệm vụ), Context (bối cảnh), Format (định dạng). Guidance: start with Task + Format; add Role or Context only when they demonstrably improve quality/consistency. Why it matters: gives a repeatable structure for prompt construction. (slide 5/64, deep-dive with examples slide 6/64)

- **Prompt iteration** — Explicit. Prompt engineering is an iterative process: write → test → observe → improve; no one writes a perfect prompt on the first try. Demonstrated via a v1 (vague) → v2 (has format) → v3 (full RTCF) progression. (slide 7/64)

- **Instruction vs Conversation vs System prompt** (prompt types) — Explicit. Three categories distinguished by purpose and use case (see Distinctions below). (slide 8/64)

- **Negative prompting & boundary setting** — Explicit. Telling the model only what NOT to do is weak; effective negative prompts pair a "don't" with a positive alternative. (slide 9/64)

- **Token budget awareness** — Explicit. Longer prompt ≠ better prompt; every extra token adds cost, latency, and sometimes noise. Practical rule: if added length doesn't change desired behavior, cut it. (slide 10/64)

- **Temperature & sampling parameters** — Explicit. Temperature controls determinism vs. diversity (0 = deterministic/classification; 0.3–0.5 = chatbot/support; 0.7–1.0 = creative; 1.0–1.5 = brainstorming). top_p (nucleus sampling, typically 0.9–0.95) restricts sampling to tokens whose cumulative probability ≤ p. Important caveat (explicit): temperature does not substitute for a good prompt — lowering it on a vague prompt just makes the model repeat the same mediocre output. Guidance: don't tune temp and top_p simultaneously. (slide 11/64)

- **Zero-shot / One-shot / Few-shot / CoT** — Explicit. Four prompting strategies distinguished by how many examples/reasoning steps are supplied (see table in section 3). Practical order of escalation: zero-shot → few-shot → decomposition/CoT; don't jump straight to complex prompting. (slide 13/64)

- **Few-shot prompting** — Explicit. 2–5 examples; used to show a *pattern* to follow, not to "re-teach" the model everything. Best practice: diverse, correctly formatted, edge-case-covering examples; 2–5 is enough. (slides 14–16/64)

- **Chain-of-Thought (CoT)** — Explicit. Prompting the model to reason step by step; appropriate for multi-step reasoning tasks, when you want visible intermediate logic, or need to debug where the model went wrong. Explicit caveat: CoT is a reasoning-improvement tool, not magic — overkill for simple formatting/extraction tasks. (slide 17/64, code example slide 18/64)

- **Tree-of-Thought (ToT)** — Explicit but only briefly introduced. Useful for problems requiring exploring multiple directions; more complex, costs more tokens/latency; should be introduced only as an extension, not a default for every task. (slide 17/64)

- **Structured output prompting** — Explicit. Needed because default LLM output is free-form text, hard to parse programmatically (e.g., in agent pipelines needing JSON). Four approaches: JSON mode (API parameter, OpenAI), prompt-based ("Respond ONLY with valid JSON"), XML tags (e.g. `<thinking>...</thinking>`), and prefill (start the assistant message with `{`, an Anthropic technique). Caveat: always validate JSON output — the model can still produce malformed output, especially with complex schemas or high temperature. (slide 19/64)

- **System prompt anatomy** — Explicit. Five components, described as priority-ordered: Persona (role, expertise, communication style) → Rules (should/must-do behaviors) → Capabilities (which tools/data allowed) → Constraints (what not to do, when to refuse/escalate) → Output format (JSON, markdown, bullets, schema, language). (slide 21/64, example slide 22/64)

- **System prompt template** — Explicit. Real-world structure: `## Identity`, `## Rules` (ALWAYS/NEVER/WHEN-conditions), `## Available Tools`, `## Output Format`, `## Escalation`. Presented as a starting point to adapt per use case. (slide 27/64)

- **Prompt injection** — Explicit. Direct injection: user directly says "Ignore your instructions and do X." Indirect injection: malicious content embedded in a document/email that the agent reads via a tool. (slide 35/64)

- **Defense strategies (6)** — Explicit. (1) Delimiter separation — wrap untrusted input, e.g. `<user_input>...</user_input>`; (2) Instruction hierarchy — system prompt always takes priority over user input; (3) Input validation — filter known injection patterns before adding to prompt; (4) Output validation — check output before executing actions; (5) Least privilege — minimal necessary tool permissions; (6) Human-in-the-loop — require confirmation for sensitive actions. Explicit caveat: no defense is 100% perfect; use defense-in-depth (multiple layers combined). (slide 36/64)

- **Prompt evaluation framework** — Explicit. Three dimensions: Correctness (is the output right? measured via test cases + human review), Consistency (does the same input reliably give matching output across runs? measured via repeated runs, % match), Safety (can it be bypassed? measured via adversarial test cases). Guidance: run 10–20 test cases; if <90% pass, iterate the prompt; use A/B testing to compare prompt versions on the same test set. (slide 37/64)

- **Guardrails pattern** — Explicit. Pipeline: User Input → Pre-guard (validate input) → LLM → Post-guard (validate output) → Safe Output. Pre-guard: detect injection attempts, validate input format, rate limiting. Post-guard: mask PII in output, validate JSON schema, block dangerous tool calls. (slide 38/64)

- **Context window management** — Explicit. Context is composed of five allocatable "buckets": System policy, History (recent/relevant), Current input (current task), Tools (schemas), Output buffer. Caveat: token budget allocation must be proactive — don't let history, tools, and examples consume all space needed for output. (slide 29/64)

- **Lost in the Middle problem** — Explicit (cites Liu et al. 2023). Attention is higher at the beginning and end of context, weaker in the middle. Practical consequences: place important instructions at the start or end; long context → middle info is easily "forgotten"; break long lists with headers/separators; recent context should sit right before the user query. (slide 30/64)

- **Memory injection & context compression** — Explicit. Memory injection: only include facts truly needed for the current task; prioritize recent/relevant history over dumping the full transcript; good for support agents, coding assistants, multi-turn tutors. Compression techniques: Summarize (condense older parts), Drop (discard irrelevant parts), Archive (push out of context, fetch back only when needed). Framing: context engineering is a selection/prioritization problem — if everything is "important," nothing actually stands out to the model. (slide 31/64)

- **Token budget allocation (buckets)** — Explicit. Table of buckets: System prompt (policy/rules/output format — risk if too large: slower, harder to maintain), History (recent turns/relevant facts — risk: noisy, lost-in-the-middle), Tool schemas (name/description/params — risk: poor tool selection if schema too long/vague), Output buffer (space for the model's answer — risk: truncated output if under-provisioned). (slide 32/64)

- **RAG context pattern** — Explicit. Flow: User Query → Retrieval (search DB) → Relevant Chunks → Inject into Prompt → LLM Response. Best practices: inject with source citation, limit chunk size (500–1000 tokens), rank by relevance and take only top-k. Note: an agent can instead have a `search_kb` tool to retrieve context on-demand rather than pre-stuffing the whole knowledge base into the prompt. (slide 33/64)

- **Tool calling** — Explicit. The mechanism by which an agent moves from "talking" to "interacting with the real world." Core point: the model does not itself run code or call external APIs — your application receives the tool request, executes the tool, and sends the result back to the model. (slide 39/64)

- **Tool calling roles** — Explicit. Four roles across two model calls: Developer (defines tool schema, writes implementation, handles errors) → LLM (decides which tool + what arguments, based on user intent) → Application (receives the tool call, executes it, returns the result) → LLM again (synthesizes the tool result into a natural-language reply). (slide 40/64)

- **Tool schema anatomy** — Explicit. Components: Name (short, clear, correct verb for the action), Description (states when to use this tool), Parameters (input described via JSON Schema), Required fields (tells the model what's missing before it can call the tool). Caveat: the LLM reads the description like documentation — a vague description causes wrong tool selection or wrong arguments. (slide 41/64, code example slide 42/64)

- **tool_choice parameter** — Explicit. Values: `auto` (default — model decides whether to call; most use cases), `required`/`any` (forces at least one tool call — pipeline steps, routing), `none` (forbids tool calls, text only — testing/fallback), `{"name": "X"}` (forces a specific tool — when you know exactly which tool is needed). Caveat: use `required` carefully — the model may call a tool with fabricated arguments if the user hasn't provided enough information. (slide 44/64)

- **Tool error handling** — Explicit. Error types and handling: Timeout → retry with exponential backoff; Error response → pass the error message to the model so it can inform the user; Unexpected format → validation layer + fallback; Tool not found → log + return error JSON. Recommended system-prompt instruction: explain the issue to the user and suggest alternatives; never silently retry more than 2 times. Explicit framing: tool errors are not an edge case — they will happen in production; plan for failure. (slide 46/64)

- **Tool design principles (4)** — Explicit. Single Responsibility (each tool does one clear thing — violation: model struggles to decide which tool to call), Idempotency (same input → same result; side effects controlled — violation: retries easily cause secondary errors), Sensible granularity (not too small, not too large — violation: large call overhead, or an overly rigid tool), Independent testing (unit test each tool before wiring into the agent — violation: hard to separate tool bugs from prompt bugs). (slide 47/64)

- **Tool granularity** — Explicit. Too small (e.g., separate `get_customer_name`/`get_customer_email`/`get_customer_phone`) → too many calls, large overhead, messy flow. Too large (e.g., `handle_all_customer_operations`) → model doesn't understand boundaries, hard to debug/reuse. Guidance: design tools around one clear business action (e.g., `lookup_order`, `get_weather`, `query_sales_data`, `send_email_draft`). (slide 48/64)

- **Parameter design best practices** — Explicit. Required vs Optional: only require what's truly necessary; Enum constraints (e.g., `"status": {"enum": ["pending","shipped","delivered"]}`) reduce argument errors; Default values should be documented in the description; add example values into parameter descriptions (e.g., date format `YYYY-MM-DD, e.g. 2026-04-05`). (slide 49/64)

- **Tool return format best practices** — Explicit. Return structured JSON (success: `{"status":"success","data":{...},"source":"..."}`; error: `{"status":"error","message":"...","code":"..."}`), not raw HTML/XML. Rules: consistent error format, include metadata (source, timestamp), truncate overly long responses. Rationale: models handle structured JSON much better than raw text or HTML. (slide 50/64)

- **Tool description engineering** — Explicit. Same tool, different description → completely different model behavior. A good description should contain: (1) function, (2) when to use, (3) when NOT to use — written like API documentation. (slide 51/64, contrast example with `search_orders` slide 51/64)

- **Sequential vs Parallel tool calls** — Explicit. Sequential: Tool B needs Tool A's output (e.g., find order ID → then check shipping status). Parallel: independent tools can run simultaneously (e.g., weather, exchange rate, and calendar lookups at once). Caveat: only parallelize when there's no data dependency; even when parallel, a clear merge/verify step is still needed at the end. (slide 52/64)

- **3 tool use patterns** — Explicit. (1) Conditional tool use: agent decides itself whether a tool is needed or answers directly. (2) Tool chaining: output of Tool A becomes input to Tool B. (3) Parallel fetch + merge: pull from multiple independent sources, then synthesize. Framing: tool calling is fundamentally a control-flow problem — when to call, what to call, in what order, and what to do on failure. (slides 53–54/64)

- **Agent/tool loop (with error handling)** — Explicit. Minimal loop: send messages+tools to model → for each `function_call` item, run the tool and append the result to messages → send back to model for final response. Robust version adds: `MAX_ROUNDS` cap (default 5 in example) to prevent infinite loops, per-tool try/except (TimeoutError and generic Exception handled), and an explicit "no tool_calls → break" exit condition. (slides 55–56/64)

## 2. Relationships & Mechanisms

- **Tool Calling Flow** (core mechanism, slide 39/64): `LLM decides (emits tool_call JSON) → App executes tool → tool result returned to LLM → LLM produces final response`. Why each step matters: the LLM never executes code itself (safety/control boundary); the application layer is the only thing that actually runs the tool, which is why error handling and validation belong to the app, not the model.

- **RTCF dependency**: Task + Format are the baseline; Role and Context are conditional additions — only include them when they measurably improve output quality or consistency (slide 5/64). This is a "start minimal, add with justification" mechanism, not a rule that all four are always required.

- **Prompting technique escalation ladder** (slide 13/64, decision tree slide 20/64): Zero-shot → Few-shot → CoT/decomposition. Decision logic (explicit, from the flowchart): if task is simple → zero-shot is enough; else if output format is unstable → use few-shot (1–3 examples); else if multi-step reasoning is needed → use CoT; otherwise → decomposition. Mechanism: don't jump to complexity — added technique should be justified by an observed problem with the simpler approach.

- **System prompt anatomy is priority-ordered** (slide 21/64): Persona → Rules → Capabilities → Constraints → Output format, marked explicitly as a priority ordering in the source diagram — i.e., these layers stack, with policy/boundary-setting elements (Rules, Constraints) governing how Capabilities and Output format get used.

- **System prompt anti-patterns contrast with anatomy** (slide 25/64 vs slide 21/64): too-long (2000+ tokens hoping the model "just gets it right"), self-contradictory (e.g., demanding both "be concise" and "explain every step in detail"), vague ("be smart," "be professional" without defining output standards), and untested edge cases (out-of-scope questions, refusals, tool failure) — these are the failure modes the anatomy + testing checklist are designed to prevent.

- **System Prompt Testing Checklist enables iteration** (slide 26/64 → slide 23/64 example): Happy path, Edge case, Out of scope, Adversarial (injection), Tool decision, Format consistency — this is the process that turns a v1 system prompt (missing constraints, causing hallucinated order status, off-scope answers, inconsistent format) into a v2 with explicit boundaries, an output contract, and a refusal pattern.

- **Guardrails pattern as pipeline**: User Input → Pre-guard (validates/filters input) → LLM → Post-guard (validates/filters output) → Safe Output (slide 38/64). This operationalizes the Defense Strategies (slide 36/64): delimiter separation and input validation happen at pre-guard; output validation and least-privilege/dangerous-call blocking happen at post-guard; instruction hierarchy and human-in-the-loop apply across the whole pipeline.

- **Context engineering ties token-budget buckets to placement strategy**: The 5 context buckets (slide 29/64) and 4 token-budget buckets (slide 32/64) define *what* competes for space; the Lost in the Middle finding (slide 30/64) defines *where* to place things within that space — important instructions at start/end, recent context right before the user query.

- **RAG as a context-injection mechanism**: `User Query → Retrieval (search DB) → Relevant Chunks → Inject into Prompt → LLM Response` (slide 33/64) is presented as one concrete way context engineering populates the "History"/"Current input" buckets, with a stated alternative — an on-demand `search_kb` tool — as a tool-calling-based substitute for pre-stuffing context.

- **Tool Use Patterns as control-flow structures** (slides 53–54/64), shown visually:
  - Conditional: `User → LLM → (?) → Tool | Direct`
  - Chaining: `User → Tool A → LLM → Tool B → Reply`
  - Parallel: `User → LLM → {Tool A, Tool B, Tool C} → Merge → Reply`
  These map directly onto the Sequential vs Parallel distinction (slide 52/64): Chaining = sequential (dependency-driven), Parallel = independent-source fetch requiring an explicit merge step.

- **Robust tool loop mechanism** (slides 55–56/64): `Input (user_input) → Process (call model → extract tool_calls → execute each with try/except → append results → repeat up to MAX_ROUNDS) → Output (final model response, or a "max rounds reached" warning)`. Why it matters: this is the concrete implementation pattern tying together tool schema design, tool_choice behavior, and error handling into one control loop; the explicit design rule is "always have a max-rounds cap; always handle errors gracefully."

- **Lab 4 operationalizes the whole lecture** (slides 57–62/64): system prompt (from the anatomy/template) + 2 tool schemas (from tool schema anatomy/description engineering) + agent loop (from the tool loop pattern) + 5 test questions (mapping to the 3 tool-use patterns + conditional/refusal handling) + error-classification notes (prompt vs tool vs control-flow) + self-review checklist (mirrors the System Prompt Testing Checklist and Tool Design Principles).

## 3. Examples & Distinctions

**Illustrative examples (selected, not exhaustive):**
- Bad vs good prompt: "Viết email cho tôi" (unclear recipient/topic/tone/length) vs. a fully specified RTCF apology email (tone, length limit, CTA) — demonstrates specificity-beats-cleverness. (slide 4/64)
- RTCF component-level good/bad table: e.g. Role "Senior Python dev, FastAPI expert" vs "Developer"; Format "Return only the function, no explanation" vs. empty — each paired with a "why it matters" reason (style/scope, ambiguity, wrong-stack guesses, unwanted explanations). (slide 6/64)
- Prompt iteration v1→v2→v3: "Tóm tắt bài báo này" (vague) → adds bullet format constraint → adds audience, focus, and tone — showing progressive RTCF completion. (slide 7/64)
- Few-shot code example: sentiment classification prompt built from two labeled examples plus a new input to classify. (slide 15/64)
- CoT code example: hotel review scoring prompt with 3 explicit reasoning steps (identify aspects → judge sentiment per aspect → synthesize final score); contrasted with a non-CoT version that outputs only "3/5" with no explanation. (slide 18/64)
- System prompt v1 vs v2: v1 lacks constraints and causes hallucinated order status, out-of-scope answers, inconsistent output; v2 adds explicit rules, "NEVER invent order status," a refusal message, and a JSON output contract. (slide 23/64)
- Good vs bad tool description: "Gets weather" (too short, no trigger condition) vs. an overly long marketing-style description (adds noise) vs. "Get current weather for a city. Use when user asks about weather, temperature, or conditions." (good — states function + trigger). (slide 43/64)
- Tool granularity example: too-small (`get_customer_name`, `get_customer_email`, `get_customer_phone`) vs too-large (`handle_all_customer_operations`). (slide 48/64)
- Tool description engineering example: vague "Search orders" (model calls it for any order-related question, even "what is an order?") vs. precise "Search orders by order_id or customer email. Use ONLY when user provides an order number or asks about specific order status." (slide 51/64)
- Anthropic vs OpenAI code-level API differences: `parameters` vs `input_schema`; response access via `message.tool_calls[0].function.name/.arguments` (OpenAI) vs `content[i].type == "tool_use"`, `.name`/`.input` (Anthropic); Anthropic system prompt is a separate `system=` field (with XML-tag structuring support like `<rules>`, `<constraints>`) vs. OpenAI system prompt living inside the `messages` array. (slides 24/64, 45/64)

**Easily-confused concept pairs:**

- **Zero-shot vs Few-shot vs One-shot**: Similarity — all are prompting-time strategies (no fine-tuning), differing only in number of examples supplied. Difference — zero-shot = no examples (fast/cheap, try first); one-shot = 1 example (better for holding format); few-shot = 2–5 examples (better consistency, more tokens). Distinguishing criterion: use few-shot when the model understands the task but produces unstable/wrong formatting, or when tone/reasoning consistency must be enforced across inputs. (slides 13–14/64)

- **CoT vs Tree-of-Thought**: Similarity — both elicit explicit reasoning from the model. Difference — CoT is a single linear reasoning chain; ToT explores multiple branching directions. Distinguishing criterion: use CoT for standard multi-step reasoning/debugging; reserve ToT (more complex, more token/latency cost) as an extension for problems genuinely requiring exploration of multiple solution paths, not as a default. (slide 17/64)

- **Instruction prompt vs Conversation prompt vs System prompt**: Similarity — all are ways of giving the model direction. Difference — Instruction prompt: single-turn direct command (Q&A, transform, summarize, classify); Conversation prompt: holds context across multiple turns (chatbot, support, tutor, multi-step debugging); System prompt: sets policy/boundary/output contract (agents, production assistants, use cases needing stable behavior). Distinguishing criterion: number of turns and whether stable governing policy is needed. (slide 8/64)

- **Direct vs Indirect prompt injection**: Similarity — both attempt to override intended model behavior. Difference — Direct: the user explicitly instructs the model to ignore its instructions; Indirect: malicious instructions are embedded in external content (document/email) the agent reads via a tool. Distinguishing criterion: injection source (user turn vs. tool-fetched content). (slide 35/64)

- **Sequential vs Parallel tool calls**: Similarity — both involve calling more than one tool to answer a request. Difference — sequential exists because of a data dependency (Tool B needs Tool A's output); parallel is possible only when tools are independent, and still needs an explicit merge/verify step. Distinguishing criterion: presence/absence of a data dependency between the tools. (slide 52/64)

## 4. Assumptions, Boundaries & Gaps

- **Prerequisite/assumption**: The lecture assumes basic familiarity with LLM API usage (OpenAI/Anthropic client calls) and Python, since code examples are given without introducing basic syntax or SDK setup. (Inferred from code-heavy slides 15, 18, 22, 42, 55–56/64 — not explicitly stated as a prerequisite.)

- **Explicit limitation — CoT is not universally beneficial**: overkill for tasks that are purely formatting or simple extraction. (slide 17/64)

- **Explicit limitation — Few-shot failure modes**: examples too similar to each other cause overfitting/poor generalization; wrong-format examples get copied faithfully; more than ~5 examples yields diminishing returns at higher token/latency cost; examples containing errors get faithfully reproduced by the model. (slide 16/64)

- **Explicit limitation — temperature is not a substitute for prompt clarity**: lowering temperature on a vague prompt just causes repeated mediocre output, not better output. (slide 11/64)

- **Explicit constraint (unexplained mechanism)**: "Đừng tune cả temp và top_p cùng lúc" (don't tune both temperature and top_p at the same time) — stated as a rule but the underlying interaction/reason is not explained in the source. Flagged as a gap.

- **Explicit limitation — no defense against prompt injection is perfect**: defense-in-depth (combining multiple layers) is presented as the mitigation, not a claim of eliminating risk. (slide 36/64)

- **Explicit risk — `tool_choice: required`**: can cause the model to call a tool with fabricated arguments if the user hasn't supplied enough information. Flagged as a known failure mode, not something the deck shows how to fully prevent beyond "use carefully." (slide 44/64)

- **Explicit framing — tool errors are expected in production**, not edge cases; the deck gives a handling table (slide 46/64) but does not go deeper into topics like idempotency-key design, distributed retries, or rate-limit-specific handling.

- **Gap — structured output validation**: the deck states "always validate JSON output" and that models can fail complex-schema/high-temperature JSON generation, but does not detail *how* to validate (e.g., schema validation libraries, retry-on-invalid strategies) beyond the general "output validation" defense bullet. Flagged, not filled in.

- **Gap — Guardrails Pattern implementation**: pre-guard/post-guard responsibilities are listed (detect injection, validate format, rate limit / mask PII, validate schema, block dangerous calls) but no concrete implementation guidance (tooling, libraries, detection techniques) is given. Flagged as mentioned-but-not-explained.

- **Gap — RAG chunk-size guidance (500–1000 tokens) and "rank by relevance, take top-k"** are stated as best practices without justification or method detail (e.g., which ranking/embedding approach). Flagged.

- **Gap — Prompt Evaluation Framework thresholds**: the "<90% pass → iterate" rule is stated but the deck doesn't explain how consistency/safety scores combine into an overall pass/fail decision, or what tooling is used for adversarial test-case generation.

- **External citations not elaborated in-deck**: Liu et al. 2023 ("Lost in the Middle") and Wei et al. 2022 (CoT) are cited as sources for claims but the deck presents only the conclusions, not methodology — treat these as attributed but not detailed. (Reference list, slide 64/64)

- **Numbering inconsistency in source**: the footer shows "62/64" twice — once on the "Tổng kết — Key Takeaways" slide and once on the "Lab #4" slide immediately before it — an apparent labeling artifact in the original deck, not a content issue. Flagged for awareness, not corrected.

## 5. Learning Priorities

**Essential** (required to understand the lecture):
- Prompt as interface framing; RTCF framework (Role/Task/Context/Format) and "start with Task+Format" rule
- Prompt iteration process (write → test → observe → improve)
- Zero-shot / Few-shot / CoT — what each is, when to escalate, and few-shot anti-patterns
- System prompt anatomy (Persona, Rules, Capabilities, Constraints, Output format) and the system-prompt testing checklist
- Tool Calling Flow (LLM decides → App executes → result → LLM final response) and the 4 tool-calling roles
- Tool schema anatomy (name, description, parameters, required fields) and tool description engineering (function / when to use / when not to use)
- Prompt injection (direct vs indirect) and the 6 defense strategies; defense-in-depth principle
- Tool error handling and the robust tool loop (MAX_ROUNDS + per-tool error handling)

**Important** (substantially improves understanding):
- Instruction vs Conversation vs System prompt distinction
- Negative prompting + positive-alternative pairing
- Token budget awareness (prompt-level) and token budget allocation buckets (context-level)
- Temperature/sampling parameter selection by use case, and the "temp doesn't fix vague prompts" caveat
- Structured output prompting approaches (JSON mode, prompt-based, XML tags, prefill) and the "always validate" caveat
- Context window management (5 buckets) and Lost in the Middle placement strategy
- Memory injection & context compression (summarize/drop/archive)
- Tool design principles (Single Responsibility, Idempotency, Granularity, Independent testing) and tool granularity trade-offs
- Parameter design best practices (required/optional, enums, defaults, example values) and tool return format best practices (structured JSON)
- tool_choice parameter values and the `required`-mode fabrication risk
- Sequential vs Parallel tool calls and the 3 tool use patterns (Conditional, Chaining, Parallel fetch+merge)
- Prompt evaluation framework (Correctness, Consistency, Safety) and guardrails pattern (pre-guard/post-guard)

**Supporting** (useful, not central):
- Tree-of-Thought (explicitly an extension, not a default)
- RAG context pattern specifics (chunk size, top-k ranking)
- Anthropic vs OpenAI API syntax differences (system prompt placement, schema field naming, response structure)
- Specific Lab 4 mechanics (exact test questions, self-review checklist wording, lab timing)
- Reference list / citation sources
