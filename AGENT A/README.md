# ELITE AI — Digital CEO & Specialist Agent Platform
## Independent Architecture Research Report (Draft v0.1, for CEO review)

**Prepared:** 22 November 2026
**Prepared for:** Human CEO / ELITE AQMS-ERP ownership
**Status:** RESEARCH PROPOSAL — NOT AN APPROVED ARCHITECTURE
**Author:** Independent architecture research team (no access to ELITE AQMS-ERP source code, database, infrastructure or documentation)

---

## 0. Read this first — what this document is and is not

**What it is.** An independent, vendor-neutral technical blueprint proposal for an AI operating layer that sits *above and alongside* the existing ELITE AQMS-ERP. It covers architecture, agent design, orchestration, controlled learning, ERP integration strategy, identity and security, local/cloud/hybrid infrastructure, voice and task follow-up, monitoring and recovery, and indicative infrastructure and cost.

**What it is not.**

- It is **not** an audit of your ERP. We have not seen the code, schema, APIs, hosting or network.
- It is **not** a final architecture. Options are compared; the human CEO decides.
- It is **not** an implementation plan with code. No code is written, and nothing is deployed.
- It is **not** a security guarantee. No architecture is 100% secure; Section 3.1 states this explicitly and the residual-risk logic behind it.

**How claims are labelled throughout.**

| Tag | Meaning |
|---|---|
| **[FACT]** | Sourced from public official documentation or standards, cited in `05-references.md`. |
| **[ASSUMPTION]** | A stated working assumption about your business or ERP, to be confirmed or corrected. Carries an ID (A-xx). |
| **[OPTION]** | A design alternative, not a decision. Carries an ID (O-xx) where relevant. |
| **[UNKNOWN]** | A gap that must be closed before a decision is safe. Carries an ID (Q-xx). |
| **[ESTIMATE]** | An indicative figure or range with assumptions stated. Not a quotation. |

**Document map.**

| File | Contents |
|---|---|
| `README.md` | This file: executive summary, principles, facts and assumption registers |
| `01-target-architecture.md` | **A**, **G**, **E** — overall architecture, local/cloud/hybrid infrastructure, ERP integration strategy |
| `02-agents-and-orchestration.md` | **B**, **C**, **D**, **H** — CEO Agent and specialists, orchestration and parallelism, knowledge and controlled learning, voice and follow-ups |
| `03-security-and-governance.md` | **F**, **I** — authentication/authorization/agent identity, monitoring, audit, approvals, recovery, threat model |
| `04-roadmap-costs-risks-questions.md` | **J** — infrastructure and operational requirements, risks, alternatives, phased roadmap, autonomy ladder, unresolved questions, information request |
| `05-references.md` | Sources, with what each was used for |

---

## 1. Executive summary (one page)

**The core recommendation.** Build a **governed AI decision layer**, not an autonomous ERP operator. Three properties define it:

1. **The ERP remains the system of record and the only system of authority.** AI never writes to ERP tables directly. Every read is mediated by an integration facade with a stable, versioned contract; every write is a narrow, purpose-built *command* executed by ERP-side or facade-side code, with idempotency, authorization and an approval record. This is the single most important architectural constraint in this report.
2. **Deterministic where it is deterministic.** Quotation arithmetic, BOM explosion, tax, lead-time math and stock reconciliation must run in **deterministic engines** (existing ERP logic or a new parametric costing engine). LLMs are used for interpretation, extraction, retrieval, drafting and explanation — never as the calculator of record. This is the difference between a demo and a system a defence supplier can defend in an audit.
3. **Confidentiality is a tier, not a setting.** Work is partitioned by data class. Restricted programme/defence data is processed only in an **isolated on-premise enclave** with no internet egress; commercial data may use an approved India-region cloud model; public data may use anything. A model router enforces the mapping, and the mapping is a policy file, not a code path.

**What the CEO Agent actually is.** A *coordinator and reporter*, not a super-user. It decomposes objectives into tasks, dispatches specialist agents, tracks state, detects blockers and produces decision memos with evidence, confidence and cost implications. It holds **no standing authority to act on business records**. Authority is delegated per-task, per-agent, per-data-class, time-boxed, and revocable.

**Autonomy model.** Five levels (L0 observe → L4 autonomous-in-policy). Every capability starts at L0/L1 and is promoted only on measured evidence plus explicit written authorization. Promotion and demotion are recorded events with an owner — not configuration drift. See `04-roadmap-costs-risks-questions.md` §4.

**Infrastructure recommendation (pending CEO decision).** A **hybrid with separated confidentiality tiers** is the most defensible default for a defence-linked manufacturer: on-premise GPU inference for restricted and confidential work, an approved India-region cloud tenant for burst capacity and non-restricted workloads, and no path by which restricted content can reach the cloud tier. A fully air-gapped build is technically feasible and is the most conservative option, but it raises capital cost, slows model refresh, and requires in-house MLOps capability. All three options are compared in `01-target-architecture.md` §4.

**Fastest safe first value.** Not "AI for everything". The first production capability should be a **read-only, evidence-linked enquiry-to-quotation assistant**: parse a customer enquiry and drawings, retrieve comparable past jobs, propose a cost structure using the *existing* costing rules, and hand a human a draft with every number traceable to a source. It touches no protected record, is reversible, and produces the training ground for everything later.

**What must happen before integration design can start.** Roughly fifteen concrete facts about the ERP and environment are needed — stack, database engine and version, hosting, auth model, whether an API layer exists, document storage, module boundaries, network topology, existing identity provider, data classification policy, and contractual constraints. The full request is in `04-roadmap-costs-risks-questions.md` §7. Without these, any integration design is speculation.

**Honest bottom line.** The hard parts of this programme are not the models. They are (a) data classification and access control that survives an audit, (b) ERP contract discipline, (c) evaluation and approval evidence good enough to justify autonomy, and (d) the organisational willingness to keep a human in the loop when the AI is usually right. Budget accordingly: models are the cheap part.

---

## 2. Design principles (proposed, for CEO ratification)

| # | Principle | Consequence if adopted |
|---|---|---|
| P1 | ERP is the system of record; AI is a decision-support layer | No direct DB writes; facade + command API only |
| P2 | No unrestricted database access for any agent | Every data access is a named, scoped tool call |
| P3 | Deterministic engines compute; models interpret | Costing/BOM/stock math never delegated to an LLM |
| P4 | Confidentiality by tier, enforced by routing | Data class decides model, storage and network zone |
| P5 | Least authority, short-lived, per-task | Tokens minted per task; no standing wildcard credentials |
| P6 | Human approval is a first-class system component | Approval service with quorum, SLA, evidence, audit |
| P7 | Everything the AI asserts is traceable | Citation to source record or document; "unsourced" is flagged |
| P8 | Learning is curated, versioned and scoped | Approved corrections become *precedents*, not global rules |
| P9 | Fail closed and fail visible | No silent degradation; refusal is a valid output |
| P10 | Reversibility before autonomy | Autonomy only where a bounded, tested rollback exists |
| P11 | One integration contract, versioned | ERP changes break loudly, not silently |
| P12 | Assume the model is compromised | Treat model output as untrusted input to every downstream system |

---

## 3. Facts register (public, sourced)

These are the only external claims treated as established. Citations in `05-references.md`.

| ID | Fact |
|---|---|
| F-01 | OWASP's current LLM risk taxonomy (2025) leads with **prompt injection** (direct and indirect, including hidden instructions in documents and images), **sensitive information disclosure**, **supply chain**, **data and model poisoning**, and **excessive agency** — the latter defined as excessive functionality, permissions or autonomy. |
| F-02 | NIST AI 600-1 (Generative AI Profile, July 2024) is the cross-sectoral companion to the AI Risk Management Framework, organised by GOVERN / MAP / MEASURE / MANAGE, listing generative-AI-specific risks and suggested actions. |
| F-03 | India's Digital Personal Data Protection Act 2023 and the DPDP Rules 2025 (notified 13 November 2025) commence in phases; the principal fiduciary obligations — notice, consent, security safeguards, breach reporting, data-principal rights — take effect **13 May 2027**. Rule 6 requires minimum safeguards including encryption/obfuscation/masking or tokenisation, access control, logs and monitoring sufficient to detect unauthorised access, continuity measures, and one-year log retention. |
| F-04 | CERT-In Directions of 28 April 2022 (under s.70B(6), IT Act 2000) require covered entities to report listed cyber incidents to CERT-In **within 6 hours**, to maintain ICT logs for a **rolling 180 days** within Indian jurisdiction, and to synchronise clocks to NIC/NPL-traceable time sources. |
| F-05 | ISO/IEC 42001:2023 is a certifiable AI management system standard, using the same harmonised clause structure (4–10) as ISO/IEC 27001:2022, with AI-specific Annex A controls and a required AI system impact assessment. |
| F-06 | vLLM serves open-weight models behind an **OpenAI-compatible HTTP API**, with PagedAttention, continuous batching, tensor/pipeline parallelism and quantisation; it targets **Linux** (no native Windows support) and its built-in `--api-key` does **not** protect every endpoint, so it must sit behind a reverse proxy/authenticating gateway. |
| F-07 | NVIDIA NIM supports fully **air-gapped** LLM deployment: model assets are pre-staged on a connected machine, transferred by approved media, then served with no outbound network and no registry credentials. |
| F-08 | LangGraph provides durable checkpointing, human-in-the-loop interrupts (pause, inspect/edit state, resume) and time-travel replay/fork; checkpointing saves at super-step boundaries, so nodes must be written to be safe on re-execution. Independent commentary (Temporal) notes that checkpoints preserve *state*, not *execution*, so process death still needs an external durable orchestrator. |
| F-09 | The Model Context Protocol (current revision 2026-07-28) standardises tool/resource exposure over JSON-RPC, and specifies that HTTP transports use **OAuth 2.1**, publish Protected Resource Metadata (RFC 9728), bind tokens to the resource audience (RFC 8707), and forbid token passthrough. |
| F-10 | OAuth 2.0 Token Exchange (RFC 8693) is the standard mechanism for delegated, down-scoped tokens carrying an `act` (actor) claim; DPoP (RFC 9449) binds a token to a key the agent holds. Major platforms (Microsoft Entra Agent ID, AWS AgentCore, Google Cloud Agent Identity) now implement agent identity on these primitives, with SPIFFE/SPIRE as the workload-identity layer. |
| F-11 | Transactional outbox, log-based change data capture (Debezium) and idempotent consumers are the established patterns for integrating with a system of record without dual-write inconsistency; CDC publishes *table changes*, whereas an application-level outbox publishes *business events*. |
| F-12 | Hybrid retrieval (lexical + dense, fused by reciprocal rank) plus reranking is the standard production RAG pattern; access control must be applied **inside** the query, before ranking, because post-filtering leaks content through counts, snippets and timing. |
| F-13 | Tamper-evidence has graded levels: retention/WORM (e.g. S3 Object Lock compliance mode, which no identity including the account root can shorten), hash-chained append-only logs, Merkle trees with signed checkpoints, and external anchoring. WORM alone proves only that a stored object was not overwritten; hash chaining proves entries were not altered in isolation. |
| F-14 | OpenTelemetry GenAI semantic conventions define `invoke_agent`, `execute_tool`, `chat` and retrieval spans with attributes such as `gen_ai.operation.name`, `gen_ai.request.model`, token usage and finish reasons; the conventions are still Development status and live in a dedicated repository, so names may change. |
| F-15 | India-region processing is available from major cloud AI providers: AWS Bedrock offers India geographic cross-Region inference across Mumbai (ap-south-1) and Hyderabad (ap-south-2) for selected Anthropic and OpenAI models; Google Vertex AI publishes data-residency support for `asia-south1` (Mumbai) and `asia-south2` (Delhi); Microsoft Azure publishes India-geo data residency; OpenAI publishes India regional processing for selected models. Residency terms differ per model and feature and must be verified contractually, not assumed. |
| F-16 | Open-source Indic speech recognition is viable on-premise: AI4Bharat's IndicWhisper (Whisper fine-tuned on ~10.7k hours across 12 Indian languages) and IndicConformer (600M, 22 scheduled languages, MIT licence) are publicly released; independent benchmarks still show word error rates above ~20% on noisy telephone speech for most systems. |

---

## 4. Assumption register (to be confirmed or corrected by the CEO)

Every one of these, if wrong, changes something material. Each has a "if wrong" note.

| ID | Assumption | If wrong |
|---|---|---|
| A-01 | ELITE AQMS-ERP is a custom-built (or heavily customised) application with a relational database, currently in production and used for core transactions. | Integration approach changes entirely (e.g. packaged ERP with vendor APIs). |
| A-02 | The ERP has little or no stable, documented, versioned API surface today. | If a good API exists, integration is cheaper and Phase 3 shortens. |
| A-03 | The company is a Tier-2/Tier-3 supplier into Indian defence programmes and also serves industrial customers, so some data is contractually restricted and some is ordinary commercial. | If all data is defence-restricted, only Option A (on-prem/air-gapped) survives. |
| A-04 | There is no dedicated in-house AI/ML platform team today; there is an ERP/IT team of a few people. | If a platform team exists, the MLOps build is faster and cheaper. |
| A-05 | Factory connectivity is reliable enough for an on-prem cluster but there is no dedicated GPU infrastructure today. | Constrains Phase 1 to CPU-only or cloud for non-confidential work. |
| A-06 | Employee communication today is a mix of phone/WhatsApp/email/paper, with no single task-tracking system of record. | If a task system exists, follow-up agents integrate with it instead of building one. |
| A-07 | Drawings are PDF/DWG/DXF/STEP files held in file shares or attached to ERP records, not in a PLM with structured metadata. | With a PLM, drawing intelligence should read PLM structure instead of parsing filenames. |
| A-08 | Quotation costing currently uses a mix of spreadsheets, historical job costs and estimator judgement. | If a parametric costing engine already exists, it becomes the deterministic core rather than being built. |
| A-09 | The company holds (or can obtain) an ISO 27001-aligned ISMS, or is willing to build one alongside this programme. | Without an ISMS baseline, the governance load rises sharply. |
| A-10 | The CEO is willing to act as the approval authority for autonomy promotion and for high-value commercial decisions. | If delegated, a decision-rights matrix must be defined first (Q-14). |
| A-11 | Budget tolerance for Phase 0–2 is in the tens of lakhs to low crores, not tens of crores, and the programme must show value within 2–3 quarters. | If capital is available earlier, the on-prem enclave can be built sooner. |
| A-12 | No workflow currently permits an AI system to commit a commercial document (PO, quotation, payment) without human signature, and this will remain true initially. | If the business wants straight-through processing early, the entire autonomy and audit design changes. |

---

## 5. Five decisions the CEO must make (and nothing else yet)

These are the only decisions needed to move to Phase 1. Everything else can be deferred.

| # | Decision | Options | Our recommendation (advisory only) |
|---|---|---|---|
| D-1 | **Confidentiality posture** — where may restricted data be processed? | (a) all on-prem/air-gapped; (b) hybrid with separated tiers; (c) cloud-first in India region | (b) hybrid with separated tiers, with (a) as the fallback if contractual review forbids any cloud processing — but this must be confirmed by your contracts and security advisor, not by us |
| D-2 | **First capability** — what proves value first? | enquiry-to-quotation assistant; drawing/BOM extraction; vendor follow-up; management reporting | enquiry-to-quotation assistant (read-only, evidence-linked) |
| D-3 | **Autonomy ceiling for year one** | L1 draft-only; L2 approved-execute; L3 bounded autonomous | L1 for commercial outputs, L2 for internal task routing and reminders |
| D-4 | **Build vs buy for the agent runtime** | custom; LangGraph/Temporal; vendor agent platform | LangGraph-style graph runtime + a durable workflow engine, self-hosted, wrapped in our own policy and approval services |
| D-5 | **Who owns AI risk** | CEO directly; a named AI risk owner + steering group | named owner (COO or CTO-equivalent) reporting to CEO, with a quarterly AI risk review — ISO/IEC 42001-style governance |

Until D-1 and D-2 are answered, no procurement and no integration work should begin.

---

## 6. Independent-research disclaimer

This report was produced without access to ELITE AQMS-ERP source code, database schema, infrastructure, contracts, or staff interviews. Every statement about the ERP is either an explicit assumption or a question. Public facts are cited. Figures marked **[ESTIMATE]** are indicative and must be replaced by vendor quotations and measured benchmarks before any budget commitment. Nothing here constitutes a security assurance, a legal opinion, or a compliance certification. No architecture is 100% secure: the objective is to make compromise costly, detectable, containable and recoverable, and to keep the blast radius of any single failure inside one data class and one workflow.
