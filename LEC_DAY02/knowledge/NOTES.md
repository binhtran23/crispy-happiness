# LEC_DAY02 — Xác Định Bài Toán Kinh Doanh Cho AI (Defining the Business Problem for AI)

## Keypoints
- Double Diamond: Diamond 1 = find the right problem (Discover → Define), Diamond 2 = find the right solution (Develop → Deliver). Don't skip either.
- HCD (Observe → Ideate → Prototype → Test → Iterate) runs inside each diamond as the operational mechanism.
- "Chatbot is not one problem" — 1 brief can map to 5 different problems, each needing different architecture and metrics.
- 4 Anti-Patterns: Trend-first, No baseline, No eval path, No owner of failure.
- AI Readiness Checklist: Value, Baseline, Eval, Tolerance, Operations — fewer than 3 YES → stop and clarify.
- AI System = Model + Context + Planning + Tools, each with distinct failure modes and controls.
- Three solution levels: Rule/Script → LLM Feature → Agent. Start left, move right only when value > added complexity.
- 5 Workflow Patterns (Anthropic): Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer. Most problems need only the first 3.
- Problem Statement 6 fields: Actor, Current Workflow, Bottleneck, Impact, Success Metric, Operational Boundary. Self-check: can you derive test cases, metrics, and boundaries from it?
- North Star Metric: Output metric (result you don't control) vs Input metrics (levers you act on daily). "Improve performance" is not a metric.
- Go / Not Yet / No-Go: "Not Yet" = discipline, not failure. "No-Go" = legitimate when rule-based suffices or error cost is too high.

## Terms
- Double Diamond (Kim Cương Đôi) — 2-phase design model: find the right problem, then find the right solution
- Human-Centered Design / HCD (Thiết kế lấy con người làm trung tâm) — iterative process: Observe → Ideate → Prototype → Test → Iterate
- Anti-Pattern (Kiểu sai lầm) — recurring mistake pattern in AI project decisions
- Baseline — manual or rule-based reference point to compare AI performance against
- HITL / Human-in-the-loop (Con người trong vòng lặp) — human review/approval at critical decision points
- Jagged Frontier (Ranh giới lởm chởm) — AI's uneven, hard-to-predict capability boundary
- Prompt Chaining — sequential LLM steps with a gate/check between each
- Routing (Định tuyến) — classify input, send to specialized branch
- Parallelization (Song song hóa) — run subtasks simultaneously then aggregate
- Orchestrator-Workers — one LLM dynamically assigns work to other LLMs
- Evaluator-Optimizer — one LLM generates, another evaluates, loop until pass
- North Star Metric (Chỉ số Ngôi Sao Bắc Đẩu) — the single metric that best captures product/business success
- Data Flywheel (Vòng quay dữ liệu) — production feedback → better data → better model → better architecture (closed loop)
- Build/Boost/Buy — spectrum of AI adoption: off-the-shelf vs enhance with proprietary data vs fully custom

## Covered / To revisit
- [x] Double Diamond + HCD — solid understanding, correctly identified prototype/test gap in scenario
- [x] Problem framing & Anti-Patterns — identified No owner of failure, initially missed No eval path (corrected)
- [x] AI Readiness Checklist — understood the logic, but mixed up Value vs Baseline when self-assessing
- [x] AI System 4 components — correctly identified Tools risk for write operations, understood Context risk after correction
- [x] Solution Levels (Rule/LLM Feature/Agent) — excellent chatbot routing example combining Rule + LLM Feature
- [x] 5 Workflow Patterns — correctly chose Routing over Orchestrator-Workers with sound reasoning
- [x] Problem Statement 6 fields — correctly evaluated a sample statement, identified test cases/metric/boundary
- [x] North Star Metric — covered in context of Problem Statement; output vs input distinction introduced
- [x] Go / Not Yet / No-Go — applied "Not Yet" correctly in scheduling example

## Misconceptions / Examiner findings
- AI Readiness Checklist — confused Value (does the problem cost enough?) with Baseline (is there a comparison point?) when working through the scheduling scenario. Gap: needs practice distinguishing "is the problem worth solving" from "do we have a reference to compare against."
- Anti-Patterns — initially identified only 1 of 2 anti-patterns in the legal summarizer scenario (caught No owner of failure, missed No eval path). After correction, understood the connection — these two often travel together.
- AI System components — initially placed access control risk under Tools rather than Context. After explanation, understood that retrieval-layer design (including who can access which docs) belongs to Context, while Tools is about write operations / side effects.
- Problem Statement — correctly evaluated the support-agent statement but didn't notice the success metric only measured speed, not quality. Gap: remember that a tight metric needs both speed/efficiency AND quality/accuracy dimensions.
