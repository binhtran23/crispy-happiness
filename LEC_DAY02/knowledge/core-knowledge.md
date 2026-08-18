# Core Knowledge

Source: `6-day02-lecture-slides-v2.pdf` — "Xác Định Bài Toán Kinh Doanh Cho AI" (AI in Action, Day 02), 50 pages/slides. Extracted via text layer (pdftotext -layout). Language of source: Vietnamese; concept names kept in Vietnamese/English as used in slides.

## 1. Core Concepts

- **Double Diamond model (Kim Cương Đôi)** — Don Norman / British Design Council (2005). Diamond 1 = find the right problem (Discover: widen — explore the underlying problem; Define: narrow — pin down the real root problem). Diamond 2 = find the right solution (Develop: widen — many candidate solutions; Deliver: narrow — select and implement). Intuition: engineers/entrepreneurs are trained to *solve* problems; designers are trained to *discover* the real problem first. Why it matters: an excellent solution to the wrong problem can be worse than no solution. (p.12, explicit)

- **Human-Centered Design (HCD) process** — Don Norman's 4 (labelled 5 on slide) iterative steps run inside each diamond: Observation (observe target-representative users in real situations), Ideation (generate many ideas without judging/constraint, question everything), Prototype (build a fast prototype per candidate solution, goal = test before building), Test (watch users interact with the prototype), Iteration (continually refine). (p.13, explicit)

- **"Chatbot" is not one problem** — a single stakeholder brief ("I want a chatbot") can map to 5 different underlying problems (auto-FAQ, ticket routing, upsell/cross-sell, QA/analytics, sentiment analysis), each needing a different architecture and metric. Formalized as: "1 brief = 5 bài toán = 5 kiến trúc = 5 metric khác nhau." Why it matters: illustrates that AI failures are often problem-framing failures, not model failures ("AI không sai. Họ giải sai bài toán."). (p.9, explicit)

- **4 Lenses (Scan from self)** — a tool for finding candidate AI problems by scanning one's own life/work: (1) Lặp lại — what do I/my team do repeatedly every day/week? (2) Tốn thời gian — what takes longer than it should? (3) AI có thể tốt hơn — which product I use could AI improve? (4) Pain từ người khác — what do colleagues/friends complain about? Used as the starting technique for the afternoon lab. (p.16, explicit)

- **4 Anti-Patterns** (ways teams waste money building AI in the wrong place): Trend-first (chasing the "agent" trend before actor/workflow/metric are clear), No baseline (building AI without a rule/manual baseline for comparison), No eval path (a nice demo but no test set, no way to know when it's "good enough" to deploy), No owner of failure (unclear who reviews wrong output, who rolls back, who owns compliance). Guiding principle: "Use AI when it creates more value than a simpler approach, not because it sounds more modern." (p.18, explicit)

- **Discovery Interview — 5 questions to ask a stakeholder**: (1) What is the pain point, how often does it occur? (2) What is the current workflow — which tools, steps, hand-offs? (3) What is the cost of the problem — minutes, money, SLA, or conversion lost? (4) What happens if AI is wrong — where is HITL/approval needed, or should it only "suggest"? (5) Who will say YES — what metric and risk decide the investment? If a stakeholder cannot describe the current workflow and failure cost, the team is "proposing a solution in the fog." (p.19, explicit)

- **AI HỢP KHI NÀO (when AI fits)** — task is repetitive but with moderate variation; task needs synthesis/search; task has multiple steps + multiple tools; NOT when task is fully deterministic (rule may be better there). (p.21, explicit)

- **Reasons a business invests in AI**: (1) Sống còn/Survival — not using AI risks being overtaken by competitors; (2) Hiệu quả/Efficiency — reduce cost, increase speed, improve throughput; (3) Khám phá/Exploration — invest to learn, avoid falling behind, find new opportunities. The reason for adopting AI determines how you build it, its automation level, and investment level. (p.21, explicit)

- **Build vs Buy — two complementary lenses**: (1) Chip Huyen (AI Engineering, 2025): Build in-house when AI is a core/survival advantage; Buy/SaaS when AI is mainly a productivity layer. (2) MIT CISR (Van der Meulen & Wixom, 2025) — three-tier spectrum: **Buy** (off-the-shelf, vendor-maintained, fast, little differentiation, dependent on vendor roadmap), **Boost** (buy a model + enhance with proprietary data via fine-tuning or RAG, needs good data governance), **Build** (fully custom model, highest control, most expensive, needs a strong AI team). Reality per slide: most teams sit in the middle — Boost (RAG/fine-tune) — not everyone needs to build from scratch. (p.22, explicit)

- **Setting expectations — 3 things to measure before shipping**: (1) Tác động kinh doanh/Business impact — Q: what value does AI create for the business? Measured by % tasks/messages automated, added capacity, response-speed improvement, labor time saved. (2) Sự hài lòng khách hàng/Customer satisfaction — Q: do users actually find it better? Measured by CSAT, direct feedback, usage behavior/drop-off/retries. (3) Ngưỡng hữu dụng/Usability threshold — Q: how good is "good enough to ship"? Measured by Quality (is output correct/useful), Latency (TTFT, TPOT, response time), Cost (cost per request). (p.23, explicit)

- **Demo vs Production gap** — Demo: ~1 weekend, verifies hypothesis, aligns stakeholders, tests UI/UX quickly. Production: months→years, must handle edge cases, guardrails, hallucination, and fight for each 1% of quality. Named "80% → 95% là đoạn đau nhất" (the 80%→95% stretch is the most painful) — cited example: LinkedIn reached 80% experience quality in 1 month, then took 4 more months to reach 95%. (p.24, explicit)

- **AI Product Lifecycle** — described only as "6 milestones từ ý tưởng đến vận hành" (6 milestones from idea to operation) plus a **Data Flywheel**: production feedback → improves data → improves model → improves architecture (a closed loop). The 6 milestones themselves are shown only as a diagram on slide 25; text layer does not enumerate them. (p.25, explicit for flywheel description; the 6-milestone list itself is a **gap** — see Section 4)

- **AI System = Model + Context + Planning + Tools** — a 4-component breakdown table with, for each component: when it's needed, its typical failure mode, how to control it, and its main cost driver.
  - *Model*: needed for open-ended tasks requiring reasoning/text generation. Failure mode: hallucination, inconsistency. Control: eval set, guardrails, structured output. Cost driver: tokens/latency.
  - *Context*: needed when the system must rely on documents/external state. Failure mode: wrong retrieval, stale context. Control: retrieval testing, freshness checks, citations. Cost driver: storage/retrieval.
  - *Planning*: needed for multi-step tasks requiring intermediate decisions. Failure mode: looping, over-planning. Control: step limits, policy, approvals. Cost driver: extra calls.
  - *Tools*: needed when the system must read/write to external systems. Failure mode: side effects, prompt injection. Control: sandboxing, allowlist, HITL. Cost driver: API calls/failures.
  (p.27, explicit)

- **Three solution levels — Rule/Script, LLM Feature, Agent** — a decision spectrum:
  - *Rule/Script*: input is stable, logic is clear, needs high predictability, heavy compliance requirements.
  - *LLM Feature*: input has moderate variation, output needs flexibility, has metrics and guardrails, humans can review.
  - *Agent*: many steps and many tools, state changes continuously, needs dynamic decision-making, has clear risk controls.
  Practical priority order: start from the left (Rule), only move right when value gained exceeds added complexity. (p.29, explicit)

- **Decision Tree: choosing solution level** — a visual decision tree from "your problem" to Rule / LLM Feature / Agent. Text layer contains only the title; the tree's actual branching logic is not present as text (diagram-only slide). (p.30, gap — see Section 4)

- **5 Workflow Patterns** — source: Anthropic, "Building Effective Agents" (2024). Guiding principle quoted: "Start with the simplest solution possible, only increase complexity when needed." Most real problems only need the first 3 patterns.
  1. **Prompt Chaining** — split a task into a sequential chain of steps with a gate/check between steps. Example: write outline → check → write article.
  2. **Routing** — classify the input, then send it to a specialized branch optimized for that type. Example: customer-service query → FAQ / refund / technical branch.
  3. **Parallelization** — run subtasks in parallel then aggregate (sectioning), or run multiple times and vote. Example: guardrail check + response generated simultaneously.
  4. **Orchestrator-Workers** — one LLM dynamically assigns work to worker LLMs; subtasks are not known in advance. Example: a coding agent editing multiple files.
  5. **Evaluator-Optimizer** — one LLM generates, another LLM evaluates, loop until it passes. Example: literary translation → review → fix.
  Patterns 4–5 and full **Agent** (LLM plans + calls tools + iterates autonomously, example: SWE-bench, computer use) cost more and accumulate errors more — only use when subtasks truly cannot be predicted in advance and tool autonomy is needed. Quoted: "Agents' autonomy makes them ideal for scaling tasks in trusted environments." (p.31–32, explicit)

- **AI Readiness Checklist — 5 quick questions**: Value (does the problem occur often and genuinely cost time/money/SLA?), Baseline (is there a manual or workflow baseline to compare against?), Eval (are there metrics, sample cases, or logs for reproducible evaluation?), Tolerance (is a finite error rate acceptable, or is HITL present at critical points?), Operations (is there an owner, rollback, logging, and policy for wrong output?). Rule: fewer than 3 YES answers → stop and clarify the problem/workflow before investing in AI. Key point: missing baseline or missing eval = you don't yet know what AI is actually better than. (p.33, explicit)

- **"Jagged Frontier"** — AI's capability boundary is uneven/hard to predict; the framework (checklist etc.) is a starting point, not a conclusion — must be tested in practice. Cited: Dell'Acqua, Mollick et al. (2023). (p.33, explicit)

- **Gate Criteria table** — "Rules of ML + practical AI product heuristics," giving Go/No-Go criteria per project stage:
  - Problem Scoping — Go if actor, workflow, pain, metric are clear; No-Go if problem is vague or solution is described before the problem.
  - Data Readiness — Go if data/logs/docs/SME support exist for eval; No-Go if no data source or data is too skewed.
  - Baseline/Model — Go if a heuristic, workflow, or human baseline is established; No-Go if it's unclear what AI needs to beat.
  - Build & Eval — Go if there's a test set, task-level eval, latency/cost budget; No-Go if there's only a manual demo with no reproducible eval.
  - Deploy Controls — Go if there's HITL, logging, approval, rollback, owner of failure; No-Go if it's unclear who approves output or who stops the system when it's wrong.
  Key takeaways: no metric → no gate; no baseline → you don't know what "better" means; no eval → it's still just a demo; no rollback → deploying is gambling. (p.34, explicit)

- **Problem Statement framework for AI systems** — a good problem statement must let you derive test cases, metrics, and boundaries. Six fields: *Actor/Operator* (who does this work daily?), *Current Workflow* (what steps/tools do they use now?), *Bottleneck* (which step is slow, error-prone, inconsistent, or needs too much synthesis?), *Impact* (loss measured in time, cost, SLA, error rate, or conversion), *Success Metric* (when is it considered successful, what's the threshold?), *Operational Boundary* (what is the system allowed/not allowed to do, and where is HITL needed?). Self-check: if you can't derive test cases, eval metric, and architecture boundary from what you wrote, the problem statement isn't tight enough yet. (p.36, explicit)

- **Problem Statement → Eval Plan** — a good Problem Statement is the bridge between the business problem and the technical eval suite. (p.37, explicit — stated as a single bridging claim, not elaborated further in the text layer)

- **North Star Metric Framework** — source: Lenny Rachitsky (a16z). Central question: which metric, if it goes up today, most accelerates the business flywheel? Six example category metrics shown: Revenue (ARR, GMV), Customer Growth (paid users, market share), Consumption (messages sent, nights booked), Engagement (MAU, DAU), Growth Efficiency (LTV/CAC, margins), User Experience (NPS, CSAT). Distinguishes **Output Metric (North Star)** — the ultimate result the team doesn't directly control (example: Airbnb's "nights booked") — from **Input Metrics (Levers)** — what the team does daily to push the output up (example: Airbnb's conversion rate, supply of homes, site traffic). Applied to the Problem Statement's "Success Metric" field: needs both an output metric (what does success mean?) and an input metric (what does the team do daily?). Explicitly warned: "Cải thiện hiệu suất" ("improve performance") is not a metric. (p.38, explicit)

- **Go / Not Yet / No-Go decision** — three possible outcomes of the discovery/feasibility phase:
  - **Go**: problem clear, baseline clear, eval clear, risk controls clear.
  - **Not Yet**: real pain exists, but data, metric, or workflow boundary is missing/unclear.
  - **No-Go**: rule-based is already good enough, the cost of AI being wrong is too high, or the cost of change exceeds the value.
  Lesson: the "correct" decision isn't always "build" — many successful AI projects start not by building immediately, but by measuring baseline, collecting logs, and trying a simpler workflow first. "Not Yet" is framed as "the most mature decision when there isn't enough data or baseline yet," not a failure. (p.40, explicit)

## 2. Relationships & Mechanisms

- **Double Diamond ⊃ HCD process**: HCD's 4 iterative steps (Observe → Ideate → Prototype → Test → Iterate) run *inside* each diamond of the Double Diamond model — HCD is the operational mechanism, Double Diamond is the higher-level structure (prerequisite/contains relationship). (p.11–13)

- **Problem-first, not AI-first** (mechanism, illustrated by 3 case studies — Team Cursor, Artifact, Lovable/Summer Labs): building AI without first validating problem/market/user fit leads to pivot, shutdown, or poor market adoption respectively. Lesson: don't build AI and hope people find a use for it — start from the end-to-end experience and identify exactly where AI solves a problem users genuinely care about. Reframes the central question from "HOW to build" to "WHAT TO BUILD." (p.15, explicit)

- **Two Google case studies contrasted** (mechanism/lesson pair):
  - Google Flu Trends: 97.5% correlation with CDC in 2008 → wrong in 100/108 weeks during 2011–2013, overestimated by >50%, missed H1N1. Root cause: wrong proxy metric (searching out of fear of flu ≠ actually having flu). Lesson: wrong problem framing / wrong proxy metric; a correlation that looks good early ≠ correct long-term.
  - Google Photos: question was "should we use AI for photo filtering?" Evaluation: rule-based was already good enough — fast, low-risk, easy to control. Decision: chose NOT to use AI. Lesson: AI doesn't add value everywhere; if rule-based is good enough, don't add complexity.
  Combined takeaway: before asking "which model," ask "am I optimizing the right variable" and "does AI actually add value here." (p.17, explicit)

- **Input → Process → Output mechanisms:**
  - *Discovery/feasibility pipeline*: Discovery Interview (5 questions) → AI Readiness Checklist (5 questions, ≥3 YES needed) → Gate Criteria (per-stage Go/No-Go) → Go/Not Yet/No-Go decision. Why each step matters: each stage filters out projects that lack clarity/baseline/eval before resources are committed, preventing "building AI on vague ground."
  - *Problem framing pipeline*: 4 Lenses (find candidate problems) → Quick Problem Card (structure the top candidate) → Peer validation (Pitch–Challenge–Vote) → Problem Statement (formalize with Actor/Workflow/Bottleneck/Impact/Metric/Boundary) → Eval Plan (derive test cases/metrics/boundaries) → Go/Not Yet/No-Go.
  - *Solution-level selection*: start at Rule/Script → move to LLM Feature only if variability/flexibility needs justify it → move to Agent only if multi-step/multi-tool/dynamic-decision needs justify it (each rightward move must be justified by value > complexity added).
  - *AI Product Lifecycle mechanism*: Data Flywheel — production feedback → improves data → improves model → improves architecture (an explicit closed loop, p.25).

- **Relationship: reason for adopting AI (Survival/Efficiency/Exploration) → determines build approach, automation level, investment level.** (dependency, p.21)

- **Relationship: Build vs Buy decision ↔ where the problem sits on the AI-fit spectrum.** Chip Huyen's lens (core/survival advantage → Build; productivity layer → Buy) and MIT CISR's Buy/Boost/Build spectrum are presented as two complementary (not conflicting) views of the same build-vs-buy decision. (p.22)

- **Trade-off: solution complexity vs cost/error accumulation.** Moving from Rule → LLM Feature → Agent, and from Prompt Chaining/Routing/Parallelization → Orchestrator-Workers/Evaluator-Optimizer → full Agent, increases capability but also increases cost and compounds errors — framework explicitly frames this as "only pay this cost when subtasks truly can't be predicted and tool autonomy is needed." (p.29, 32)

- **Dependency: Problem Statement quality → Eval Plan quality → Gate Criteria pass/fail.** A tight Problem Statement (one that lets you derive test cases, metrics, boundaries) is a prerequisite for a valid Eval Plan, which in turn is required to pass the "Build & Eval" and "Baseline/Model" gates. Explicit chain: "no metric → no gate; no baseline → don't know what's better; no eval → still just a demo." (p.34, 36–37)

## 3. Examples & Distinctions

- **Examples of "questions that changed the world"** (illustrating Norman's "question everything, even the obvious"): Newton's "if the apple falls, does the moon fall too?" → foundation of classical mechanics; a 3-year-old's "why do I have to wait to see the photo?" (Jennifer Land, 1943) → led to the Polaroid instant camera invention (by her father); "what if we rent out empty rooms to travelers?" (Brian Chesky & Joe Gebbia, 2007) → Airbnb, valued at $100B+. (p.14, explicit)

- **Weekly Report worked example** (Quick Problem Card, p.49): Problem = "spending 3h/day compiling feedback from 200+ customer emails into a report for the product team." Actor = Product Manager (~50 PMs company-wide, recurring weekly). Current workflow (5 steps): receive email → read each email → self-summarize → merge into doc → send to team. Bottleneck = step 2, read & summarize (~2.5h/day, 150h/week across 50 PMs). AI fit = step 2 — AI summarizes each email, PM reviews/edits before sending. Success metric = reduce 3h → under 30 min/day, more consistent report quality. Quick gut-check = "LLM" chosen because natural-language processing is diverse/varied, not rule-based-able. This is a fully worked instance of the Quick Problem Card / Problem-Statement-lite pattern.

- **Rule/Script vs LLM Feature vs Agent** — distinguishing criteria (see Core Concepts for full table): similarity = all are candidate "solution levels" for an AI-adjacent problem; difference = input stability, need for flexible output, decision dynamism, and compliance/predictability needs. Distinguishing criterion: start left (Rule), move right only when value > added complexity. (p.29)

- **Output Metric (North Star) vs Input Metrics (Levers)** — similarity: both are metrics tracked for the same product/goal. Difference: Output Metric is the end result the team does not directly control (e.g., Airbnb "nights booked"); Input Metrics are the day-to-day levers the team directly acts on to move the output (e.g., Airbnb conversion rate, home supply, site traffic). Distinguishing criterion: does the team control it directly (Input) or only influence it indirectly (Output)? (p.38)

- **Demo vs Production** — similarity: both are stages of the same AI product. Difference: Demo is ~1 weekend and optimizes for hypothesis verification/stakeholder alignment/UI test; Production takes months to years and must additionally handle edge cases, guardrails, hallucination, and incremental quality gains. Distinguishing criterion: whether the system needs to survive real-world edge cases and sustained quality bar, not just prove a concept. (p.24)

- **Go vs Not Yet vs No-Go** — similarity: all are legitimate outcomes of the discovery/feasibility process. Difference: Go = clarity across problem/baseline/eval/risk controls; Not Yet = real pain but missing data/metric/workflow clarity (temporary block); No-Go = rule-based already sufficient, or error cost/change cost too high (permanent rejection, not just "not ready yet"). Distinguishing criterion: is the blocker "insufficient information/data" (Not Yet) or "insufficient value/too much risk even with full information" (No-Go)? (p.40, explicit)

## 4. Assumptions, Boundaries & Gaps

- **Diagram-only slides with no extractable body text** (text layer empty or title-only on these pages; content exists only as a visual diagram the extractor could not read):
  - p.5, "Building AI Product" — subtitle "AI Product chứa Build Product bên trong — không thay thế" (AI product contains build-product inside it, doesn't replace it), but the diagram itself is not in the text layer.
  - p.25, "AI Product Lifecycle" — "6 milestones từ ý tưởng đến vận hành" is stated, but the 6 milestones themselves are only shown in the diagram, not present as text.
  - p.26, "Vai Trò Của UX Trong AI Product" (Role of UX in AI Product) — title only, no body text extracted at all.
  - p.30, "Decision Tree: Chọn mức giải pháp" — title/subtitle only ("Từ bài toán của bạn → Rule, LLM Feature, hay Agent?"); the tree's branching logic is not in the text layer.
  - p.44–48 (Lab overview, Phase 1+2, Workflow Diagram guidance, Worked Example "Weekly Report — Trước và Sau AI") — largely diagram/template slides with minimal extractable text; treated as lab-administrative rather than core conceptual content.
  Per extraction method (text-only), these are flagged as gaps rather than filled in — a vision-based re-read of these specific pages would be needed to recover their content.

- **Prerequisites/assumptions embedded in the frameworks:**
  - AI Readiness Checklist assumes the team can honestly self-assess 5 binary(ish) YES/NO conditions; threshold of "≥3 YES" is stated as a rule but the checklist itself calls out that AI capability is a "Jagged Frontier" — i.e., even a checklist pass is not a guarantee, real-world testing is still required. (p.33)
  - The Gate Criteria table assumes a linear/staged project structure (Problem Scoping → Data Readiness → Baseline/Model → Build & Eval → Deploy Controls); no explicit handling of iterating backward if a later gate fails is described in the slides.
  - The 5 Workflow Patterns are explicitly scoped as "đủ cho hầu hết bài toán" (enough for most problems) with the more complex patterns (Orchestrator-Workers, Evaluator-Optimizer, full Agent) reserved for cases where subtasks truly cannot be predicted — the slides don't give a precise/quantified boundary for when this threshold is crossed, only qualitative guidance ("chỉ dùng khi không predict được subtasks").

- **Limitations/edge cases explicitly named in the source:**
  - Google Flu Trends failure case: correlation looking strong early does not guarantee long-term validity — the proxy metric (flu-related searches) diverged from ground truth (actual flu cases) over time. (p.17)
  - "80% → 95% là đoạn đau nhất" — the slides state this stretch is the hardest/slowest part of AI productization but do not explain the underlying causes/mechanisms beyond naming edge cases, guardrails, and hallucination as production-specific challenges. (p.24)
  - No-Go conditions explicitly include: "hậu quả khi sai quá đắt" (cost of being wrong too high) and "change cost lớn hơn value" (cost of process change exceeds value) — both flagged as legitimate reasons to reject an AI project even when technically feasible. (p.40)

- **Concepts mentioned but not elaborated in the source (flagged, not filled from external knowledge):**
  - "Problem Statement tốt là cầu nối giữa bài toán kinh doanh và bộ eval kỹ thuật" (p.37) is asserted as a single-line claim without further explanation of *how* that bridging mechanism works beyond what's already covered by the Problem Statement's 6 fields.
  - The specific mechanics of RAG and fine-tuning (mentioned only as labels under "Boost" in the Build/Boost/Buy spectrum, p.22) are not explained — the slides assume prior familiarity with these terms.
  - HITL (Human-in-the-loop) is referenced repeatedly (Discovery Interview, Readiness Checklist, Problem Statement, Gate Criteria, Rule/Workflow/Agent table) as a control mechanism but is never itself defined on the slides — assumed prior knowledge.
  - "Prompt injection" is named as a Tools failure mode (p.27) without definition — assumed prior knowledge.

## 5. Learning Priorities

**Essential** (required to understand the lecture):
- Double Diamond model + "find the right problem before the right solution" (p.11–12)
- "Chatbot is not one problem" example — problem framing before solution (p.9)
- Rule / LLM Feature / Agent — three solution levels and the "start simple, add complexity only when justified" principle (p.29)
- 5 Workflow Patterns (Anthropic) — especially Prompt Chaining and Routing as the default-sufficient patterns (p.31–32)
- AI Readiness Checklist (5 questions, ≥3 YES rule) (p.33)
- Problem Statement framework (6 fields: Actor, Current Workflow, Bottleneck, Impact, Success Metric, Operational Boundary) (p.36)
- Go / Not Yet / No-Go decision framework, including "Not Yet is not failure" (p.40)
- AI System = Model + Context + Planning + Tools, with failure modes/controls per component (p.27)

**Important** (substantially improves understanding):
- HCD process (Observe/Ideate/Prototype/Test/Iterate) as the mechanism inside Double Diamond (p.13)
- Problem-first not AI-first — 3 case studies (Cursor, Artifact, Lovable) (p.15)
- Google Flu Trends vs Google Photos — proxy metric failure vs "rule-based is enough" (p.17)
- 4 Anti-Patterns (Trend-first, No baseline, No eval path, No owner of failure) (p.18)
- Discovery Interview 5 questions (p.19)
- Build vs Buy — Chip Huyen lens + MIT CISR Buy/Boost/Build spectrum (p.22)
- Demo vs Production gap, "80%→95%" (p.24)
- Gate Criteria table (per-stage Go/No-Go conditions) (p.34)
- North Star Metric framework — Output vs Input metrics (p.38)

**Supporting** (useful, not central):
- 4 Lenses for scanning problems from self (p.16) — primarily a lab technique
- "Questions that changed the world" examples (Newton, Polaroid, Airbnb) (p.14)
- Setting expectations — 3 measurement categories (business impact, customer satisfaction, usability threshold) (p.23)
- AI Product Lifecycle / Data Flywheel (p.25) — content partly gap (see Section 4)
- Reasons to invest in AI: Survival/Efficiency/Exploration (p.21)
- Quick Problem Card worked example (Weekly Report) (p.49) — illustrative instance, not a new concept
- Lab logistics: Pitch-Challenge-Vote, Deliverables structure (p.43–50)
