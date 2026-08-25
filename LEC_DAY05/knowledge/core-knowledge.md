# Core Knowledge

**Source:** `day05-lecture-slides-batch03.pdf` (39 slides) — "Thiết kế sản phẩm AI cho sự không chắc chắn" (Designing AI Products for Uncertainty), VinUni AI20K, Day 05, Batch 02.
**Language note:** deck is in Vietnamese; extraction preserves original Vietnamese terms alongside English equivalents where the deck itself uses them (e.g. "Recall", "Precision", "Copilot").

## 1. Core Concepts

- **AI product uncertainty (3 layers)** — Core framing concept (p.6, p.8): unlike traditional software, AI products have uncertainty in **Input** (user asks messily: missing context, vague terms, changes mind mid-way, prompt injection), **Output** (same intent → different answers; model updates change style; RAG/tools return different data), and **Process** (model self-reasons, multi-step tool chains, user can't tell if source is right). Matters because it's the foundational reason AI products need different design treatment than regular software. Related: AI product ≠ software product, error routing.

- **AI product ≠ software product** (p.7) — Traditional software: stable input → deterministic logic → clear right/wrong output; once a bug is fixed, it's gone. AI product: input, output, and process all have variance; the same request can yield different results depending on model, context, and data. Product implication: not just writing features, but designing acceptance thresholds, fallback, correction, logging, and an accountability owner. Key stated line: "AI product không hứa 'không bao giờ sai'; nó hứa 'khi không chắc hoặc sai, hệ thống vẫn dẫn user đi đúng hướng'" (AI product doesn't promise "never wrong"; it promises the system still guides the user right when uncertain or wrong).

- **Production/behavior drift** (p.9) — Even without code changes, product behavior can shift via: Model update (may improve on average but regress on old tasks), Context drift (underlying data changes — policy, price, docs, schedules), User drift (real users ask outside scope, lack info, use slang), Prompt drift (small patched rules accumulate into unpredictable behavior). Implication: AI products need eval, version log, and fallback from the very first prototype.

- **Error routing framework** (p.11) — Detect → Route → Recover → Learn. Detect = know when uncertain (confidence, missing field, stale data, out-of-scope request). Route = choose a safe path (ask again, offer multiple choices, human review, refuse). Recover = let user fix (undo, edit, report, correction path, manual fallback). Learn = save signal (approve/reject, edit distance, retry, handoff, reason). Stated principle: "Nếu prototype chỉ có happy path, đó chưa phải AI product" (a prototype with only the happy path is not yet an AI product).

- **Automation vs. Augmentation** (p.12–15) — A product decision, not a maturity hierarchy. Augmentation: AI increases human capability (suggests, summarizes, drafts, ranks); human makes the final decision; lower risk; learns from approve/reject. Automation: AI acts autonomously within a defined scope; needs threshold, fallback, logging; if wrong and hard to undo, risk rises sharply. Stated insight: augmentation is "không phải bản kém của automation" — often the correct step to reduce risk and gather real data before increasing automation.

- **Automation ladder** (p.15) — A single task can move through multiple automation levels as accuracy/trust increases (0%→100%): e.g., refund-email task: 60% accuracy → Copilot mode (user approves every email); 95% → send automatically but ask again for important emails; 99.5% → full automation (user only sets rules). Stated principle: there is no universal accuracy threshold across domains — 60% can still be useful if user only needs to approve; 99.5% can still be dangerous if AI autonomously acts wrong. Quote (Kevin Weil, CPO OpenAI): "Nếu model đúng 60%, bạn phải build product theo cách hoàn toàn khác." Safety rule: if an error is hard to undo, don't go full-auto.

- **Human-in-the-loop roles** (p.16) — Human-in-the-loop must have a *specific* role, not just "let a human approve": Reviewer (checks AI-drafted output: approve/edit/reject), Decider (AI presents options, human is accountable for the choice), Trainer (creates learning signal: correction, label, rank, reason feeding into eval set), Rescuer (intervenes on low-confidence/safety-risk/escalation/handoff cases). Stated spec check: writing "human review" without specifying the role and where the reviewed output goes is insufficient.

- **Three pillars of AI product design** (p.17–18) — Requirement, UX, Eval — each must be redesigned around the fact that AI will be wrong. Requirement: not just "feature," but outcome + threshold + failure behavior (e.g., "Hỏi X → Y khoảng 85%; dưới 60% thì hỏi lại user"). UX: not just "pretty screen when right," but designed for when AI is wrong (user sees the error, can fix it, and can trust the product again) — graceful failure + trust recovery is core, not an add-on. Eval: not just pass/fail, but "run 100 times → what % is 'good enough,' which errors can't exceed a threshold" — a PM decision on quality distribution, not just QA's job.

- **4-path user stories / 4-path AI spec** (p.19, p.37) — AI requirement/spec must describe outcome + confidence threshold + fallback, generating 4 paths instead of 1: Happy (correct, confident), Low-confidence (uncertain — ask again or offer choices), Failure (wrong — user recovers via undo/edit/handoff), Correction (user fixes — data goes into correction log / rule / test set). Bank-chatbot example illustrates ≥60% confidence → answer + cite source + update date; <60% → "Tôi không chắc, chuyển nhân viên."

- **Eval as quality distribution, not pass/fail** (p.21) — Traditional software is always 100/100/100; AI oscillates (example: 82/65/91 across runs). Process: (1) look at 50–100 results, (2) classify errors, (3) write eval cases from repeated error patterns. Stated principle: don't conclude AI quality from a single correct demo — run many cases (happy, ambiguous, missing data, prompt injection, edge case). PM/product-builder's job: classify errors by cause (requirement / data-tool / UX / safety / missing eval case) before fixing.

- **False Positive vs. False Negative (báo nhầm vs. bỏ sót)** (p.10, p.22) — Two directions of AI wrongness whenever AI decides "yes/no." False Positive ("có" but actually "không"): warns/blocks something actually fine — damage is extra work for humans and erosion of trust in warnings over time. False Negative ("không" but actually "có"): lets something real through — damage is uncaught risk, unprotected people/cases, and the error disappearing from the responsible person's view. Stated product question: which type of wrong is more costly for *your* product?

- **Precision and Recall** (p.23) — Two metrics, each measuring a different kind of wrong. Precision = "when AI says yes, how much can I trust it?" = correct positives / total AI "yes" calls; low precision → many false alarms → users lose trust and start ignoring warnings. Recall = "of all truly-yes cases, how many did we catch?" = caught / total truly needed catching; low recall → many missed cases go unhandled. Worked example: AI flags 40/1000 transactions as suspicious, 30 correct → precision = 75%; actually 50 bad transactions exist, AI caught 30 → recall = 60%.

- **Graceful failure** (p.27) — Concrete UX mechanism (not just an acknowledgment that "AI can be wrong") for reducing damage and preserving trust when AI is wrong: offer multiple choices instead of one absolute answer, let user edit output directly, fallback to manual/human review, log corrections to turn errors into signal (damage reduction); explain reasoning (Grammarly), show confidence so user decides (Kayak), offer alternative/undo/report/turn-off options (trust preservation).

- **Four new AI UI components** (p.28) — Prompt (suggest how to start), Editable Plan (user-editable plan, generated from prompt), Showing Work (show process just enough — e.g. progress bar / "reading 15 sources... 60%"), Follow-up (suggest an active next step). Quote (Aparna Chennapragada): AI interface must be "vừa đủ minh bạch" — too long becomes a cron job; too short and the user doesn't know if AI is on the right track.

- **Failure taxonomy (5 layers)** (p.29) — Promise (what does user expect?), Intent (does AI understand user's real intent?), Data/Tool (right source/tool available?), Safety/Behavior (risky behavior — prompt injection, agreeing despite conflicting data), UX Recovery (how does user recover?). Used to classify any bug/finding before deciding a fix. Illustrated with MONI/NEO chatbot examples.

- **Bug → Decision template** (p.30) — Every finding must be rewritten as a product decision using the template: "Khi user [trigger], AI/product [failure], hậu quả là [impact], lỗi thuộc layer [taxonomy], nên sửa bằng [requirement/eval/UX/data/automation], đo bằng [metric hoặc signal]." Debrief formula (p.31): don't write "bot lỗi" (bot is buggy); write finding → layer → product decision → SPEC field → test/failure path.

- **AI Product Canvas** (p.36) — A single page to keep product decisions from drifting toward "just demo it": Value (for whom, what pain, what AI solves that current methods don't), Trust (what happens when AI is wrong — user awareness, fix, undo, handoff, trust regain), Feasibility (cost/request, latency, data, main risk, kill threshold). Includes Learning signal question: where does user correction go, and which signals (approve, edit, retry, handoff, report) show the product is improving or degrading.

- **Find → Synthesize → Decide research flow** (p.32–35) — Evidence (what did user say/encounter — quotes, screenshots, review clusters, competitor examples) → Insight (deeper pattern — e.g., users don't lack information, they lack decision guidance) → Opportunity (where AI helps) → Build slice (proof: one flow of input → AI → output → failure path). Research toolkit: Self-use, Social/interview ("khi nào bạn bị kẹt gần đây nhất?"), Review mining (30–50 reviews, AI-clustered, choose top evidenced failure mode). Diverge→Cluster→Score→Commit for narrowing ideas to one buildable slice, scored by evidence strength, user value, AI fit, build feasibility, failure risk.

- **Learning signal** (p.38) — If a product doesn't collect learning signal, it does not improve. Three questions: (1) Where does user correction go (fixed transaction, changed label, rejected suggestion, reported wrong answer → saved as learned data)? (2) What signal shows improvement/degradation (approve rate, edit distance, retry, handoff, time-to-resolution, report-wrong)? (3) Does the data have marginal value (the model already has general knowledge — product wins via domain data, user-specific data, and human judgment)?

## 2. Relationships & Mechanisms

- **Uncertainty → Product decisions (mechanism, p.8):** the 3 uncertainty layers (input/output/process) are not just descriptive — the slide explicitly frames "thiết kế đúng" as converting each into a concrete product decision: when to ask again, where to show source, when to hand off to a human, how to log corrections. Depends-on: recognizing uncertainty layers before designing UX/eval.

- **Drift → Need for eval/versioning (mechanism, p.9):** four types of drift (model, context, user, prompt) enable/cause the requirement for eval, version logs, and fallback to exist even in a thin prototype — this is a "therefore" relationship stated directly in the deck ("Vì vậy...").

- **Error type asymmetry → UX/automation priority (trade-off, p.10, p.22–25):** which error (false positive vs false negative) is more costly for a given domain determines which UX safeguard to prioritize (recovery/undo for FP-costly domains like spam-into-folder; early warning/escalation for FN-costly domains like child-content filtering or X-ray reading) and how conservative automation should be. No universal answer — domain-dependent product decision (p.25).

- **Detect → Route → Recover → Learn (process, p.11):** `Input: AI uncertainty signal (confidence, missing field, stale data, out-of-scope) → Process: choose safe path (ask again / offer choices / human review / refuse) → Output: user can fix (undo / edit / report / fallback) → Feedback: signal saved (approve-reject, edit distance, retry, handoff, reason)`. Each step matters because skipping Detect means silent wrongness; skipping Route means no safety net; skipping Recover means user distrust compounds; skipping Learn means the product never improves.

- **Task boundary → Automation feasibility (mechanism, p.14):** breaking a broad "can this be automated?" question into 4–6 discrete tasks is a prerequisite for correctly answering automation vs. augmentation per task (different tasks in the same workflow — e.g. FAQ deadline vs. debug coaching vs. rubric grading vs. question routing — warrant different automation levels based on risk and stability).

- **Automation ladder mechanism (p.15):** accuracy/trust level (0→100%) enables progressively higher levels of automation, but the mapping is not fixed — it depends on whether errors are undoable (safety constraint) and whether the domain tolerates the given error rate at all (no universal threshold).

- **Requirement → UX → Eval, three pillars as a single design loop (p.18):** requirement defines the threshold and failure behavior; UX operationalizes what the user experiences at that threshold (graceful failure); eval measures whether the resulting quality distribution meets the requirement. They are interdependent — a requirement without an eval definition of "% acceptable" is incomplete (p.19), and eval without UX-defined recovery paths has nowhere to route failures (p.11, p.27).

- **Precision/Recall trade-off (p.23–25):** improving Recall (catch more) typically raises False Positives (Precision falls), and vice versa — this is implied by the definitions and demonstrated via the domain table (p.25) where different domains deliberately prioritize one over the other (e.g., child-content filtering prioritizes Recall; loan approval typically leans Precision).

- **Bug → Layer → Decision → Spec → Test (chain, p.29–31):** a raw finding (e.g., "prompt injection happened") is classified into a taxonomy layer (Safety/Behavior), which determines the type of product decision needed (refusal + handoff + red-flag logging), which must then be written into a concrete SPEC field, which must be verifiable by a test/failure-path case. Skipping any link keeps the finding as "chuyện kể trên lớp" (a classroom anecdote) rather than an actionable product fix.

- **Evidence → Insight → Opportunity → Build slice (p.34):** each stage depends on the previous — evidence (raw observation) must be synthesized into insight (a pattern about what users actually lack, e.g. "decision guidance" not "information"), which then defines an opportunity for AI (specific: "ask 3 questions, suggest 2-3 choices"), which must be provable via one buildable flow. Explicitly rejects vague acceptance ("AI assistant for healthcare") in favor of a specific, buildable slice.

## 3. Examples & Distinctions

- **Google Bard vs. Gamma/Slide AI vs. Air Canada** (p.4) — three failure cases opening the lecture: Bard = one factual error becomes a brand/stock/trust risk (demo can create too much trust too fast); Gamma/Slide AI = 0→80% fast but the last 20% requires regenerate/fix layout/fix data, driving users back to old tools (generation ≠ correction-friendly); Air Canada = chatbot fabricated refund policy, company held liable (the company/system is accountable for bot errors, not just the bot). Shared insight: same AI tech, but products differ in where they set boundaries, how outputs are corrected, and who is accountable when AI is wrong.

- **False Positive vs. False Negative:** similarity — both are "AI is wrong" whenever a yes/no decision is made; difference — FP wrongly flags/blocks something fine (cost: extra human work, erosion of trust in warnings); FN wrongly lets something real through (cost: uncaught risk, unprotected cases). Distinguishing criterion: direction of the wrong decision relative to ground truth ("có" vs "không").

- **Precision vs. Recall:** similarity — both quantify a "yes/no" decision system's error; difference — Precision measures trustworthiness of positive calls (correct positives / all positive calls, tied to False Positives); Recall measures coverage of true positives (caught / all true positives, tied to False Negatives). Distinguishing criterion: what denominator each ratio is computed against (all AI "yes" calls vs. all real "yes" cases).

- **Automation vs. Augmentation:** similarity — both are ways AI can be deployed into a workflow; difference — augmentation keeps the human as final decider (AI suggests), automation lets AI act/decide within scope. Distinguishing criterion: who executes the final action, and correspondingly, who bears accountability/risk when wrong.

- **Spam filter false positive vs. false negative** (p.10) — concrete worked example distinguishing which is worse depending on context: real email marked spam (may cause missed opportunities/appointments/invoices, needs undo/whitelist/review) vs. spam reaching inbox (annoying but user can delete; risk increases specifically if phishing/fraud is involved).

- **Copilot's 4-question UX example** (p.26) — used to distinguish deliberate UX design choices at each of the 4 paths: when right (grey suggestion, Tab to accept), when uncertain (shorter/less-confident suggestion, user writes on their own), when wrong (keep typing = suggestion disappears, near-zero correction cost), when losing trust (turn off per file/language, turn back on anytime). Contrasted with Microsoft Tay as a counter-example lacking any recovery path when user behavior attacked the bot.

- **Domain-by-domain error cost table** (p.25) — explicit contrast across four domains (child-content filtering, code suggestion, X-ray reading, loan approval) showing that "which error is worse" is not universal but domain-specific, directly grounding the "no single answer" principle.

## 4. Assumptions, Boundaries & Gaps

- **Assumption:** the deck assumes a baseline familiarity with what a chatbot/copilot/AI agent is (no definitions given for "agent," "RAG," "prompt injection," "confidence" as a numeric concept) — these terms are used but not formally defined in-deck. Flag: terms like "confidence," "threshold," "RAG," and "tool chain" are used operationally throughout (e.g., p.6, p.8, p.19) without an explicit technical definition of how confidence is computed.

- **Boundary:** the deck explicitly states there is **no universal accuracy/automation threshold** across domains (p.15, p.25) — this is presented as a design principle, not resolved into a formula; learners must reason per-domain.

- **Gap:** the specific mechanics of *how* to compute or calibrate a model's "confidence" score (used repeatedly as a threshold input, e.g. ≥60%/<60% in the bank chatbot example, p.19) are not explained — the deck treats confidence as a given input into the routing decision, not something it teaches how to derive.

- **Gap:** "marginal value" of collected correction data (p.38, item 3) is asserted ("model đã biết kiến thức chung; product thắng bằng dữ liệu domain, user-specific và human judgment") but not elaborated with a method for determining when data stops being valuable or how much data is "enough."

- **Limitation/edge case (high-value for Socratic questioning):** prompt injection is repeatedly flagged as a top failure mode (p.8, p.20, p.29, p.31) with a mitigation pattern (instruction hierarchy, refusal, red-flag logging) but no discussion of how robust these mitigations actually are or their known limitations — presented as a mitigation checklist rather than a solved problem.

- **Assumption:** the "4-path user story" framework (Happy/Low-confidence/Failure/Correction) assumes AI systems can reliably self-report a confidence signal usable for routing (p.19, p.37) — this assumption itself is not questioned or stress-tested in the deck.

- **Boundary:** the deck frames PM/product-builder as responsible for deciding "quality distribution" (p.18, p.21) — i.e., product management owns the % threshold decision, not engineering/QA alone — but doesn't address how this decision should be negotiated cross-functionally or resourced.

- **Edge case explicitly flagged:** Microsoft Tay (p.26) is cited as a counter-example of a product with no recovery path against adversarial user behavior — flagged as a cautionary example but not analyzed in depth (no discussion of what specifically failed in Tay's design).

## 5. Learning Priorities

**Essential** (required to understand the lecture):
- AI product ≠ software product; AI = uncertainty (input/output/process) (p.6–8)
- Error routing framework: Detect → Route → Recover → Learn (p.11)
- Automation vs. Augmentation as a product decision, not a hierarchy (p.13–15)
- Three pillars: Requirement (threshold+fallback), UX (graceful failure), Eval (quality distribution, not pass/fail) (p.17–21)
- False Positive vs. False Negative, and Precision vs. Recall as distinct lenses (p.10, p.22–23)
- "Which error costs more" is domain-specific and must drive UX/automation priority (p.24–25)
- Bug → Decision → SPEC mapping template (p.30–31)
- 4-path user stories (Happy / Low-confidence / Failure / Correction) (p.19, p.37)
- Learning signal: without it, AI product doesn't improve (p.38)
- Five closing principles (p.39)

**Important** (substantially improves understanding):
- Production/behavior drift (model, context, user, prompt) requiring eval/versioning from prototype stage (p.9)
- Human-in-the-loop roles: Reviewer / Decider / Trainer / Rescuer (p.16)
- Automation ladder concept — a task can move through multiple automation levels (p.15)
- Failure taxonomy (Promise / Intent / Data-Tool / Safety-Behavior / UX Recovery) (p.29)
- AI Product Canvas (Value / Trust / Feasibility / Learning signal) (p.36)
- Find → Synthesize → Decide research flow; Evidence → Insight → Opportunity → Build slice (p.32–35)
- Task boundary: splitting a workflow into 4–6 tasks before deciding automation (p.14)

**Supporting** (useful, not central — safe to skip under time pressure):
- Specific opening failure-case anecdotes (Bard, Gamma/Slide AI, Air Canada) as motivating hooks (p.4)
- "AI Raced Ahead, UX Stood Still" historical chart framing (p.5)
- Four new AI UI component patterns: Prompt / Editable Plan / Showing Work / Follow-up (p.28)
- Diverge → Cluster → Score → Commit brainstorming mechanics (p.35)
- Individual domain-cost worked examples beyond the general precision/recall pattern (p.25)
