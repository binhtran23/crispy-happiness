# Core Knowledge

Lecture: **Data Governance & AI Security** (AICB-P2T2 · Ngày 24 · Chương 5: Vận Hành)
Deck: `stream.pdf`, 28 content slides (numbered 1/28–28/28) + title/QA/thanks slides, 40 pages total. Content mostly in Vietnamese with English technical terms; extraction preserves original terms where precision matters.

Canonical framing on slide 3: this lecture's security focus is the **threat model & control plane**; prior/adjacent days cover other layers (Day 22 = guardrails, Day 23 = telemetry, Day 26 = MCP/A2A protocol, Day 27 = lineage).

---

## 1. Core Concepts

- **Agent Governance (vs. classic Data Governance)** — Explicit. Old data governance answered "who can read this table?"; the new reality is that the reader is an **agent that acts** (uses your credentials, on untrusted input, at machine speed). Central reframing question of the lecture (p.2). Why it matters: legacy grant-based permission models ("table grant") break down when the accessor is agentic.

- **NHI (Non-Human Identity) / Agent identity** — Explicit. Ratio of NHI:human is 90:1 in many orgs (up to 144:1 in some). 51% of orgs have no clear ownership for NHI; 92% of orgs breached by AI had no AI access control (p.11). Related: Delegation Chain, Least Agency.

- **Agent ≠ Service Account** — Explicit (p.3, callout). A service account executes predefined commands; an agent *interprets goals and chooses its own actions* (has **agency**). Any control inherited from service-account models is missing this one dimension: agency.

- **Governance Control Plane — 5 layers** (p.4) — Explicit model:
  1. **Policy** — what is permitted: data contract, purpose, classification
  2. **Identity** — who/what is acting: per-agent identity, delegation chain
  3. **Enforcement (PEP)** — blocks *at every tool call*, BEFORE execution
  4. **Data controls** — minimization, PII masking, encryption, retention
  5. **Evidence** — append-only audit ledger → raw material for compliance
  Key decision-point shift: from "grant on catalog" (one-time, static) to "runtime check at every tool call" (continuous, contextual).

- **Classification × Agency matrix** (p.5) — Explicit upgrade of the traditional 4-level classification table (Public/Internal/Confidential/Restricted). New dimension: not just what policy applies, but what *agency* (read-summarize / read-transform / write / export-egress) is allowed per classification level. E.g., Restricted (PII) permits only "no-egress run" for read-summarize; write and export are blocked (×).
  - "no-egress run" = read permitted only inside a run with no network access (detailed at §6/p.24).

- **Governance stack (3 standards, don't pick just one)** (p.6) — Explicit:
  - **EU AI Act** — law: binding legal requirement
  - **NIST AI RMF** — method: Govern / Map / Measure / Manage
  - **ISO/IEC 42001** — certifiable standard: auditable evidence
  Mapping given: Govern ↔ Clause 5–6, Map ↔ impact assessment, Measure ↔ monitoring, Manage ↔ operational controls.

- **Five mandatory artifacts** (p.6): (1) Data contract (schema + purpose + retention), (2) AIBOM + agent card (which model/tool/MCP server is running), (3) DPIA / impact assessment record, (4) Policy-as-code repo (with PR review history), (5) Audit ledger (append-only, every tool call). Rule stated: governance is not a PDF written by hand at audit time — it is a machine-readable artifact that self-generates evidence.

- **OWASP ASI Top 10 (Agentic Applications), 12/2025** (p.7) — Explicit list, published 09/12/2025, >100 experts:
  - ASI01 Agent Goal Hijack — treat all retrieved content as untrusted
  - ASI02 Tool Misuse — least-agency scoping, validate parameters
  - ASI03 Identity & Privilege Abuse — per-agent identity, limited tokens
  - ASI04 Agentic Supply Chain — component signature, AIBOM
  - ASI05 Unexpected Code Execution — sandbox, egress deny-by-default
  - ASI06 Memory & Context Poisoning — validate memory writes
  - ASI07 Insecure Inter-Agent Comms — mutual auth, signed messages
  - ASI08 Cascading Failures — blast-radius isolation, circuit breaker
  - ASI09 Human-Agent Trust Exploitation — forced confirmation
  - ASI10 Rogue Agents — behavioral monitoring, kill switch
  Distinction from OWASP Top 10 for LLM: LLM Top 10 treats the model as a *text generator*; ASI Top 10 treats it as an **actor** — has credentials, memory, tools, and freedom to refuse/choose actions across multi-step chains.

- **Lethal Trifecta** — Explicit (p.8). Three co-occurring conditions that make an agent exploitable and that CANNOT be "patched" away:
  1. Private data access (mail, DB, file, memory)
  2. Untrusted content exposure (doc, web, tool output)
  3. Exfil vector (HTTP, link, image, API)
  Root cause: model and data flow through the **same token stream** — no architectural boundary separates instructions from data. Case study: Claude Cowork (01/2026) — 48 hours after launch, PromptArmor demonstrated a full attack chain via hidden white text in a Word doc → prompt injection → agent uploads victim's financial docs to attacker's Anthropic account, exfiltrating attacker's own injected API key. Files API exfiltration bug had been reported to HackerOne in 10/2025 but was unfixed at Cowork launch. Strategic implication: since no single leg of the trifecta can be architecturally separated, defense shifts from **prevention to containment** (§6).

- **Memory & Context Poisoning (ASI06)** (p.9) — Explicit. Attack techniques: PoisonedRAG (as few as 5 poisoned docs in a corpus of millions → RAG answers per attacker intent ~90% of targeted-query time); AI Recommendation Poisoning (Microsoft, 02/2026) — corrupting persistent memory to steer financial decisions/workflow; single injection persists across all future sessions. Controls: validated memory write, provenance + source signature (hash document, timestamp index), TTL/ephemeral context (memory not permanently on by default), versioning embeddings, targeted rollback (delete only the poisoned entry). Key link to data governance: agent memory IS a data store — needs classification, retention, audit, and right-to-erasure, just like a warehouse table.

- **MCP Supply Chain (ASI04 + ASI02)** (p.10) — Explicit. Attack vectors: tool poisoning (malicious tool description hidden in context user cannot see — analogous to indirect prompt injection), rug pull (tool silently changes behavior after being approved), cross-server tool shadowing, confused deputy & token passthrough, SSRF via tool connector, rogue server registration. Controls: OAuth 2.1 + PKCE mandatory for user-facing flows, Token Exchange (RFC 8693) — no forwarding tokens upstream to downstream, immutable + versioned + signed tool definitions (ETDI direction), MCP allowlist + internal registry + review cadence. Fact: >30 CVEs targeting MCP server/client/infra in just 01–02/2026; MCP spec has no native defense against tool poisoning or rug pulls — that's the implementer's responsibility. (Protocol & routing detail deferred to Day 26.)

- **Delegation Chain** (p.12) — Explicit model: User (root identity) → Agent (own identity) → Sub-agent (depth ≤ 2) → Tool (per-task catalog) → Data (row/column scope). Annotations: on-behalf-of + TTL 15 min (user→agent), PEP checks before execute (sub-agent→tool), masking + audit (tool→data).

- **Least Privilege → Least Agency** (p.12) — Explicit reframing: the unit of authorization is not just *what data* an agent can see, but *what tool catalog* is scoped per task. Example: a research agent gets search + read, never write, and is not simply granted "every tool that might be needed."

- **Last mile at the data layer** (p.12) — Explicit: Unity Catalog / Apache Ranger provide row-level + column-level access control + dynamic masking. Stated as **a** PEP (Policy Enforcement Point), not the entire answer.

- **Policy-as-Code at the Tool-Call PEP** (p.13) — Explicit, with OPA/Rego example. Decision inputs: `data.classification` (from catalog), `request.purpose` (from data contract), `agent.owner` (from NHI registry), `delegation.depth` (guards against excessive sub-agent delegation), `run.egress_enabled` (blocks the 3rd leg of the trifecta). Every decision + reason is written to the audit ledger — the reason field IS the compliance evidence (§5): a deny without a logged reason cannot be debugged or audited.

- **PII Gate before ingestion** (p.14) — Explicit, with Python/Presidio example targeting Vietnamese text. Gate protects three destinations: training set (old thinking), RAG corpus (readable = leakable), agent memory (PII entering here is near-permanent). Vietnam-specific PII patterns requiring custom NER: CCCD (12-digit national ID), phone (+84/0[35789]xx), bank account numbers, BHXH (social insurance) numbers, addresses. Gap flagged on-slide: Presidio has no built-in Vietnamese support — requires configuring `NlpEngineProvider` with a Vietnamese spaCy model (`vi_core_news_lg`); community coverage is thin.

- **Pseudonymization vs Anonymization** (p.15) — Explicit, formal distinction (see §3 below).

- **Right-to-Erasure across the AI stack** (p.16) — Explicit pipeline: Lakehouse (delete + vacuum) → Feature store (online + offline) → Vector index (delete doc id) → Agent memory (targeted rollback) → Logs/traces (vendor-side). Model weights flagged separately as **not deletable** — "machine unlearning" is not something an org can safely commit to a regulator as a guarantee; PII baked into model weights is a permanent liability. Real-world gap noted: most off-the-shelf LLM APIs do not provide sufficient logging/lineage/deletion for audit; vendor data flows must be mapped in DPIA (example: Anthropic's 30-day retention for Fable 5 safety monitoring is a data flow that must be disclosed in DPIA).

- **Encryption & Secrets — agent's-eye view** (p.17) — Explicit. Encryption baseline (referenced as covered on Day 16): TLS 1.3 in transit (mandatory), AES-256 at rest (KMS managed key), column-level encryption for PII fields, envelope encryption (DEK wrapped by KEK; KEK rotates yearly, DEK rotates monthly). Secrets are framed as "exfiltration target #1": no long-lived keys on agent filesystem (use OIDC federation, short-lived credentials); sandboxing write access alone is insufficient if egress isn't also blocked (an agent that can read `~/.aws/credentials` but has unblocked egress still leaks); filesystem isolation and network isolation must go together; deny rules must cover subprocess, package scripts, child shell, symlink/hardlink, path traversal, hidden dirs, imported scripts. Reframe stated on-slide: encryption protects data *when stolen*; secret hygiene prevents the agent from *voluntarily handing over the key* to an attacker.

- **Luật 91/2025/QH15 (Vietnam PDPL)** (p.18) — Explicit. Passed by National Assembly 26/6/2025, effective 01/01/2026. Establishes rights: to be informed, to consent, to access, to correct, to request deletion. "Right to erasure" → maps to delete cascade concept (previous slide).

- **NĐ 356/2025/NĐ-CP** (p.18) — Explicit. Issued 31/12/2025, effective 01/01/2026. 5 chapters, 42 articles, 10 appendix forms. Cross-border requirement: DPIA (impact assessment dossier) must be filed within **60 days** of cross-border data transfer; missing dossier → must be completed within 30 days or face administrative penalty. Exemptions from assessment: personnel, logistics, tourism, emergency/urgent cases. Inspection: periodic + surprise/ad hoc.
  - Critical callout on-slide: **calling an LLM API hosted abroad = cross-border transfer of personal data** → falls under this 60-day dossier requirement. Also noted: NĐ 13/2023 is no longer the governing framework — do not reuse old slide content referencing it.

- **EU AI Act after Digital Omnibus (2026)** (p.19) — Explicit timeline: Digital Omnibus on AI takes effect 27/7/2026 (OJ 24/7/2026). Milestones: 02/8/2026 — transparency + AI literacy obligations (timeline preserved); 02/12/2027 — high-risk Annex III obligations delayed; 02/8/2028 — some embedded-in-products obligations delayed (Annex I). Already in force, unchanged: Art. 5 bans (unacceptable risk) + GPAI model provider obligations; AI Office has exclusive supervisory authority over AI systems built on a GPAI model from the same provider. From 02/8/2026 orgs must: disclose when a user is interacting with AI; mark generative output as machine-readable; Art.10 (data governance for high-risk systems) requires artifacts matching §1 of this lecture.

- **Requirement → Control → Evidence table** (p.20) — Explicit compliance-mapping artifact (see §2). Framed as the exact format required for Lab 24 deliverables: "compliance is a build artifact generated from the audit ledger, not a hand-written document."

- **AI vs AI / Frontier offense-defense** (p.21) — Explicit real-world offense data: 80–90% of tactical volume in campaign GTG-1002 (Anthropic, disclosed 13/11/2025) was AI-self-executed, spanning recon → exploit → credential harvesting → lateral movement → exfil, across ~30 targets, with only 10–20% human effort (MITRE ATT&CK campaign C0062). Anthropic × MITRE mapping (2026): 13,873 actions mapped, 482 techniques, 14 tactics; 67.3% used AI to write malware, 6.5% for lateral movement, some overlap with Verizon DBIR data. Verizon DBIR 2026: 44% of phishing had AI assistance; defender AI usage rose from 15%→45% within a company timeframe (Verizon DBIR data point cited generally, 33%→56% medium/high-risk actor AI usage, 1.7×). Conclusion stated on-slide: technique count no longer correlates with actor sophistication — orchestration is what distinguishes agentic threat actors, and ATT&CK currently lacks an ID for that.

- **Dual-Use Frontier Models — access as governance** (p.22) — Explicit table of 4 example models with capability/access mechanism (Claude Mythos 04/2026 — cyber vuln-finding, limited rollout; Claude Fable 5 09/6/2026 — Mythos-class, safeguards for general use, classifier routes cyber/bio queries to Opus 4.8, 30-day retention; Claude Mythos 5 09/6/2026 — same base model minus cyber safeguard, restricted to Project Glasswing partners only; GPT-5.6-Cyber 10/8/2026 — 95% task completion on elevated cyber tasks, trusted-partner access via Daybreak program). Stated lesson: both major labs have moved to **gated capability access** — "who can use which model tier, for what purpose, with what logging" is now a policy question belonging to the control plane (§1), not a procurement decision.

- **Agentic Security in the SDLC** (p.23) — Explicit list of real, deployed tools: OpenAI Aardvark → Codex Security (research preview, 06/3/2026) — continuous repo scanning, exploitability assessment, patch proposal on commit; Google Big Sleep — first AI-found zero-day in production software (SQLite buffer underflow, missed for years by OSS-Fuzz); XBOW — autonomous agent, #1 on HackerOne leaderboard, 1,060+ validated submissions. Defender advantage stated: source code, telemetry, and merge-gate authority — but this advantage only materializes if defenses are automated.

- **Containment Architecture** (p.24) — Explicit synthesis model (see §2 Input→Process→Output below). 5 runtime controls: per-user data scoping, per-task tool catalog, allowlisted exfil channel, policy enforcement before execute, verifiable audit trail for every tool invocation. Theoretical grounding: 6 design patterns against prompt injection (arXiv 2506.08837) + CaMeL (Google DeepMind) — action-selector pattern where tool output never re-enters the agent's instruction stream; dual-LLM and code-then-execute cited as strongest but most complex patterns. Concrete example: Claude Code combines permissions + MCP allowlist + OS sandbox (bubblewrap/seatbelt) — reduces permission prompts 84% while maintaining containment.

## 2. Relationships & Mechanisms

- **Agent Governance depends on** re-deriving the classic access/input/blast-radius model for a non-human, autonomous actor (p.5 table): access shifts from human-via-BI-tool to NHI/agent (90:1 ratio); input shifts from structured queries to untrusted content (doc/email/webpage/tool output); blast radius shifts from "read wrong table" to "action taken" (API call, DB write, email sent).

- **5-Layer Control Plane as a pipeline**: Policy defines rules → Identity establishes who's acting → Enforcement (PEP) intercepts every tool call before execution → Data controls apply minimization/masking at the point of access → Evidence captures the decision as audit trail. Missing any one layer = "governance on paper only" (explicit callout on p.4: "Thiếu 1 lớp = governance trên giấy").

- **Delegation Chain as Input → Process → Output**:
  - Input: User identity (root) + intended task
  - Process: Agent receives delegated identity (on-behalf-of, TTL 15 min) → may spawn sub-agent (depth capped ≤2) → sub-agent requests tool access → PEP checks *before* execute → tool queries data (masked, scoped by row/column)
  - Output: task result + audit trail
  - Why each step matters: capping delegation depth bounds blast radius; TTL bounds exposure window; PEP-before-execute is what makes enforcement preventive rather than just detective.

- **Policy-as-Code decision mechanism (OPA/Rego example, p.13)** as Input → Process → Output:
  - Input: `data.classification`, `request.purpose`, `agent.owner`, `delegation.depth`, `run.egress_enabled`
  - Process: default-deny; allow only if owner exists AND purpose matches data contract AND not blocked; block if classification=restricted AND egress enabled (trifecta guard); block if delegation depth > 2
  - Output: decision (allow/deny) + reason, written to audit ledger
  - Mechanism significance: this is literally the Enforcement (PEP) layer of the 5-layer control plane instantiated as executable policy, and the reason field is what converts a runtime decision into compliance evidence (link to §5/Requirement→Control→Evidence).

- **Requirement → Control → Evidence, as a mechanism (p.20)** — Explicit mapping used as the compliance production pipeline:
  - Luật 91/2025 (right to erasure) → Delete cascade (§4/right-to-erasure pipeline) → Job log + vector index diff
  - NĐ 356/2025 (60-day cross-border dossier) → Data-flow inventory for every LLM API call → DPIA + NĐ356 forms
  - ASI03 (privilege abuse) → Per-agent identity + STS token 15 min → IAM report + TTL policy
  - ISO 42001 Clause 5–6 (Govern) → Policy-as-code repo with review → PR history
  - EU AI Act transparency (02/8/2026) → Disclosure + output marking → Screenshot UI + config
  - This table is the master mechanism connecting every other concept in the deck: each law/standard requirement is only real if it has a technical control producing machine-readable evidence.

- **Lethal Trifecta → shifts defense paradigm**: because prevention (removing any one leg) is architecturally impossible when private-data access + untrusted content + exfil vector must coexist for the agent to be useful, the mitigation strategy necessarily moves to **containment** — i.e., the Containment Architecture (p.24) is the direct architectural response to the Lethal Trifecta (p.8). This is the single most emphasized causal chain in the deck (repeated on p.8 and again structurally in §6).

- **Memory Poisoning ↔ Data Governance**: agent memory is explicitly treated as *a data store equivalent to a warehouse table* — it must inherit classification, retention, audit, and right-to-erasure obligations, not be treated as an exempt/ephemeral system component (p.9, p.16).

- **PII Gate → RAG/Agent Memory → Right-to-Erasure**: PII controls form a lifecycle — gate blocks ingestion into training/RAG/memory (p.14) → if data ages/needs deletion, the erasure pipeline must cascade across lakehouse, feature store, vector index, agent memory, and logs (p.16) → model weights are the one stage where this cascade breaks down (cannot delete), which is flagged as an unresolved liability, not a solved control.

- **Frontier dual-use models → Governance**: as model capability for offense (cyber, per p.21–22) increases, labs respond not with disabling the capability but with **gated access control** (who/what purpose/logging) — meaning frontier-model procurement itself becomes an instance of the control-plane's Policy + Identity layers (p.22 explicitly reframes model access as a policy question, tying back to §1).

## 3. Examples & Distinctions

- **Pseudonymization vs Anonymization** (p.15) — similarity: both alter PII to reduce direct identifiability. Difference/distinguishing criterion: pseudonymization is *reversible* via a lookup table (replaced value is still legally personal data, lookup table must be protected as strictly as the original PII; used for internal analytics, A/B testing, cross-table joins). Anonymization is *irreversible* — no re-identification possible; achieved via k-anonymity (each record indistinguishable from ≥k−1 others) or generalization (e.g., age 32 → bucket 30–39); used for synthetic/public datasets and research sharing. Common 2026 mistake flagged on-slide: PII filtered from training data but flowing unfiltered into the RAG index — because the "anonymize before ingestion" rule must now apply to vector stores and agent memory too, not just the training set.

- **Service Account vs Agent** (p.3) — similarity: both are non-human actors with credentials. Difference: service account executes predetermined commands (no agency); agent interprets goals and autonomously chooses actions (has agency). Distinguishing criterion: presence/absence of agency — controls inherited from service-account models are missing this dimension.

- **OWASP Top 10 for LLM vs OWASP ASI Top 10** (p.7) — similarity: both are OWASP-maintained security taxonomies for AI systems. Difference: LLM Top 10 treats the model as a *text generator* (bộ sinh text); ASI Top 10 treats it as an *actor* — with credentials, memory, tools, and multi-step autonomous action, published 09/12/2025 by the OWASP GenAI Security Project with >100 contributing experts.

- **Case study example: Claude Cowork (p.8)** — used to demonstrate the Lethal Trifecta concretely: hidden white text in a Word doc (untrusted content) triggers prompt injection; agent (with access to victim's financial docs — private data) uploads the docs to attacker's Anthropic account, exfiltrating the attacker's own API key which had been injected (exfil vector). Timeline: exploited 48 hours after 13/01/2026 launch by PromptArmor's demo. The underlying Files API bug had been reported to HackerOne in 10/2025 and was still unfixed at launch — used as an example of unresolved architectural risk, not a one-off bug.

- **GTG-1002 campaign (p.21)** — example distinguishing "AI-assisted" attacks from "AI-orchestrated" ones: 80–90% of tactical volume across the full intrusion lifecycle was autonomously executed by AI, with human effort at only 10–20%, across roughly 30 targets — used to argue that technique variety is no longer the marker of a sophisticated threat actor; orchestration is.

- **Model access tiers example (p.22)**: Claude Fable 5 (safeguards on, general use) vs Claude Mythos 5 (same base model, safeguard removed, restricted to vetted partners) — illustrates that the same underlying capability can be governed entirely through access policy rather than model modification.

## 4. Assumptions, Boundaries & Gaps

- **Assumption**: the lecture assumes learners have already covered Day 16 (encryption fundamentals), Day 22 (guardrails), Day 23 (telemetry) — content there is referenced, not re-explained (p.3, p.17).
- **Boundary/deferred topic**: MCP protocol and routing internals are explicitly deferred to Day 26 (p.10, p.24); catalog and lineage mechanics deferred to Day 27 (p.6).
- **Explicit unresolved gap**: Presidio (PII detection library shown in code, p.14) has no built-in Vietnamese language support; requires manually configuring a Vietnamese spaCy model (`vi_core_news_lg`), and community coverage/tooling for this is thin — flagged directly on-slide as a practical blocker, not solved by the lecture.
- **Explicit unresolved gap**: model weights cannot be deleted — "machine unlearning" is stated as something an organization should not commit to as a regulatory guarantee. PII embedded in model weights is described as a permanent liability with no current technical resolution shown in the deck (p.16).
- **Explicit gap**: most off-the-shelf LLM APIs do not provide sufficient logging, lineage, or deletion capability to support an audit (p.16) — a limitation of the vendor ecosystem the org must work around via DPIA data-flow mapping, not a limitation solved by any control shown.
- **Explicit gap**: the MCP protocol itself has no native defenses against tool poisoning or rug-pull attacks — responsibility falls entirely on the implementer (p.10).
- **Explicit gap**: ATT&CK (MITRE's framework) currently has no technique ID for "orchestration" as a distinguishing signature of agentic threat actors (p.21) — flagged as a framework gap, not filled by the lecture.
- **Boundary condition on trifecta**: the deck explicitly states no architectural boundary currently separates instructions from data because both flow through the same token stream — this is presented as a fundamental, not incidental, limitation of current LLM architectures (p.8).
- **Legal/regulatory boundary explicitly flagged as commonly misunderstood**: calling an LLM API hosted outside Vietnam counts as a cross-border personal data transfer under NĐ 356/2025, triggering the 60-day DPIA filing requirement (p.18) — presented as a compliance trap many orgs miss.
- **Explicit deprecation notice**: NĐ 13/2023 is stated as no longer the governing legal framework for Vietnam data protection — slides/materials referencing it are outdated (p.18).
- **Mentioned but not sufficiently explained** (flag only, not filled in): the "6 design patterns against prompt injection" from arXiv 2506.08837 are referenced by citation only (p.24) — the deck names the paper and highlights CaMeL/action-selector and dual-LLM/code-then-execute as strongest patterns, but does not enumerate all 6 patterns or their individual mechanisms.
- **Mentioned but not sufficiently explained**: RFC 8693 (Token Exchange) and ETDI (tool definition signing direction) are named as controls (p.10) but their internal mechanics are not detailed in the slides.
- **Uncertain/unverifiable statistics**: several headline statistics (e.g., 90:1 / 144:1 NHI ratio, 51% no NHI ownership, 92% breached orgs lacked AI access control, >30 CVEs in MCP infra Jan–Feb 2026, 84% reduction in Claude Code permission prompts) are presented as explicit facts on-slide with sourcing implied (cited generally, e.g. IBM Cost of a Data Breach 2026, Verizon DBIR 2026, listed in References p.27) but exact citation-to-statistic mapping is not always shown per-slide — treat as explicit-per-slide but externally unverified within this extraction.

## 5. Learning Priorities

**Essential**
- Agent Governance reframing (data governance → agent governance) and the "Agent ≠ Service Account / agency" distinction (p.2–3)
- Governance Control Plane, 5 layers (p.4)
- Lethal Trifecta and its consequence (prevention → containment) (p.8)
- OWASP ASI Top 10, especially ASI01, ASI02, ASI03, ASI06 (p.7)
- Delegation Chain & Least Privilege → Least Agency (p.12)
- Policy-as-Code at the Tool-Call PEP mechanism (p.13)
- Requirement → Control → Evidence compliance mapping (p.20) — this is the Lab 24 deliverable format
- Luật 91/2025 and NĐ 356/2025 core obligations, especially the cross-border/LLM-API trigger (p.18)
- Containment Architecture — 5 runtime controls (p.24)

**Important**
- Classification × Agency matrix (p.5)
- Five mandatory governance artifacts (p.6)
- Memory & Context Poisoning (ASI06) attacks and controls (p.9)
- MCP Supply Chain risks (ASI04+ASI02) (p.10)
- PII Gate mechanics and Vietnam-specific PII patterns (p.14)
- Pseudonymization vs Anonymization distinction (p.15)
- Right-to-Erasure pipeline and the model-weights gap (p.16)
- Secrets vs Encryption distinction ("agent's-eye view") (p.17)
- EU AI Act post-Digital-Omnibus timeline (p.19)
- GTG-1002 / AI-orchestrated offense data (p.21)

**Supporting**
- Governance stack mapping across EU AI Act / NIST AI RMF / ISO 42001 (p.6)
- Dual-use frontier model access examples (Mythos/Fable/GPT-5.6-Cyber) (p.22)
- Agentic security SDLC tooling examples (Aardvark, Big Sleep, XBOW) (p.23)
- Live Demo / Lab 24 walkthrough structure (p.25–26) — operational, not conceptual
- Reference list (p.27) — for citation lookup only
