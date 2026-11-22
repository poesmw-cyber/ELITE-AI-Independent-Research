# 04 — Infrastructure, Costs, Risks, Roadmap and the Questions We Cannot Answer

Covers assignment section **J** (estimated infrastructure and operational requirements), plus risks, alternatives summary, phased roadmap, autonomy ladder, unresolved questions and the minimum information needed to proceed.

---

## 1. Infrastructure requirements (J)

All figures are **[ESTIMATE]** for a company of the stated profile (A-01, A-04, A-11). They exist to frame a budget conversation, **not** to be quoted. Replace every number with a vendor quotation and a measured benchmark before commitment.

### 1.1 Environment tiers

| Environment | Purpose | Shape | Notes |
|---|---|---|---|
| **Dev** | Prompt/agent development, unit evals | 1 workstation-class GPU or CPU-only + cloud sandbox for C1 | No restricted data, ever |
| **Eval / staging** | Golden-set evaluation, red-team suite, integration tests against anonymised ERP copy | 1–2 GPUs; isolated | Must mirror production model/quantisation, or evals lie |
| **Production enclave** | C3/C4 inference, knowledge, orchestration, audit | 1 node, 4–8 datacentre GPUs; N+1 recommended for availability | No internet egress |
| **Cloud tier (India region)** | C1/C2 burst, non-restricted drafting, provider evals | Pay-per-use | Contract must cover no-training, residency, notice |
| **DR** | Enclave recovery | Cold standby or 1 spare GPU node; offsite immutable backup | Restore drills, not just backups |

### 1.2 Indicative hardware, on-prem enclave **[ESTIMATE]**

| Item | Small (pilot, 30B-class) | Recommended (70B-class + growth) | Notes |
|---|---|---|---|
| GPU server | 4× L40S 48GB (or 4× L4) | 8× H100/H200 80GB+ | Single node preferred over multi-node for restricted data |
| CPU / RAM | 2× 32-core, 512 GB | 2× 48-core, 1–2 TB | RAG, ingestion, orchestration are RAM-hungry |
| Storage | 15–30 TB NVMe + 30 TB HDD | 30–60 TB NVMe + 60 TB backup | Model weights are 40–150 GB each; documents grow |
| Network | 10 GbE to Zone 0 | 25 GbE, redundant | Read replica traffic |
| Backup | Local NAS + immutable copy | WORM object store (on-prem), offsite encrypted | **[FACT F-13]** |
| Power/cooling | 6–10 kW rack | 10–15 kW rack | **Check the server room's rating first (Q-11)** |
| UPS/generator | Sized for graceful shutdown | Full ride-through | A GPU node without clean shutdown is a corrupted node |

Indicative capital range **[ESTIMATE, India market, 2026]**: a 4-GPU L40S-class server is typically in the **₹60 lakh – ₹1.2 crore** range fully configured with storage and networking; an 8×H100-class node is typically **₹2.5 – ₹4 crore**. Add 10–20% for installation, power infrastructure and spares. GPU prices and lead times vary substantially — obtain three quotations.

### 1.3 Software stack (all self-hostable, no mandatory SaaS)

| Layer | Candidate | Licence note |
|---|---|---|
| Inference | vLLM (or NVIDIA NIM) **[FACT F-06/F-07]** | Open source; NIM requires NVIDIA AI Enterprise licence for support |
| Models | Open-weight instruction model (Llama/Mistral/Qwen-class), small classifier model, embedding model, reranker | **Licence review is mandatory** — some weights have acceptable-use or commercial restrictions. Get this reviewed before selection, not after |
| ASR | IndicWhisper / IndicConformer **[FACT F-16]** | MIT for IndicConformer; verify per checkpoint |
| TTS | Piper-class local | Open source |
| Orchestration | LangGraph + Temporal **[FACT F-08]** | Open source, self-hosted |
| Vector + relational | PostgreSQL + pgvector, or a dedicated vector store | Choose pgvector first: fewer systems, ACL joins in SQL |
| Object store | MinIO or equivalent, with object lock | For WORM audit copies |
| Policy | OPA / Cedar-style | Policy as code |
| Identity | Existing IdP + SPIRE (workload identity) **[FACT F-10]** | Open source |
| Observability | OpenTelemetry + Prometheus/Grafana + Loki, SIEM feed **[FACT F-14]** | Open source |
| Secrets | Vault or cloud KMS/HSM | HSM for root keys |
| Container platform | Kubernetes (or Docker Compose for a pilot) | Network policies |

### 1.4 Operational requirements — the part budgets forget

| Requirement | Why | Indicative effort **[ESTIMATE]** |
|---|---|---|
| **AI platform engineer** (inference, GPU, containers) | The enclave does not run itself | 1 FTE, from Phase 2 |
| **AI/agent engineer** (prompts, evals, orchestration) | Core build | 1–2 FTE |
| **Integration engineer** (ERP facade, events) | The hardest, least glamorous work | 1 FTE, shared with ERP team |
| **Security engineer** (part-time or contracted) | Threat model, injection testing, IAM | 0.5 FTE |
| **Data/knowledge engineer** (ingestion, quality, golden sets) | Determines whether answers are any good | 0.5–1 FTE |
| **Process owners per domain** (estimator, PPC head, quality head) | Label data, define golden answers, approve | 10–20% of 4–6 people |
| **AI risk owner + steering group** | Governance, autonomy promotion, incident decisions **[FACT F-05]** | 0.2 FTE + quarterly review |
| **Training and change management** | Adoption is the real failure mode | 1 person part-time, ongoing |
| **Vendor/contract management** | Model licences, cloud terms, renewals | 0.2 FTE |

**Honest staffing statement:** this programme fails more often from missing process ownership than from missing technology. If the estimator and PPC head cannot give 10% of their time to define and validate answers, do not start Phase 2.

### 1.5 Running cost structure **[ESTIMATE]**

| Cost line | Early phase (pilot) | Steady state (year 2) |
|---|---|---|
| Cloud inference (C1/C2 + evals) | ₹50k–₹2L / month | ₹1–5L / month, workload-dependent |
| On-prem power/cooling | — | ₹50k–₹2L / month depending on load and tariff |
| Hardware depreciation | — | Capital / 4 years |
| Software licences | Low (open source) | ₹5–20L / year (support, NIM/EH, observability) |
| People | 2–3 FTE | 4–5 FTE |
| Security tooling/audit | ₹2–5L / year | ₹10–30L / year |
| Model/API contingency | — | 15% buffer for price changes |

Cost per task is the metric to track: `(inference + infrastructure + amortised people) / completed tasks`. It is the only number that tells you whether the system is affordable at scale, and it should be on the CEO dashboard from month one.

---

## 2. Risks

### 2.1 Programme risks (the ones that kill projects)

| # | Risk | Likelihood | Impact | Mitigation | Early warning |
|---|---|---|---|---|---|
| R-1 | **ERP integration takes far longer than planned** because the ERP has no API and no documentation | High | High | Discovery in Phase 0 before any commitment; facade isolates impact; start read-only | Facade contract count stalls |
| R-2 | **No process owner time** → no golden sets → no measurable accuracy → no autonomy | High | High | Make owner time a Phase 1 exit condition | Golden sets <50 cases after 6 weeks |
| R-3 | **Data quality** (inconsistent part numbers, duplicate vendors, missing drawings) | High | High | Data-quality workstream from day one; entity resolution in the facade; do not "let the AI fix it" | Retrieval precision plateaus; duplicate comparables |
| R-4 | **Pilot succeeds, adoption fails** (people don't use it, or use it and then re-key into Excel) | Medium | High | Design around existing habits; replace one existing report exactly; measure weekly active use | Usage decays after week 4 |
| R-5 | **Confidentiality incident** (C4 leaked to cloud, or via chat app) | Medium | Very high | Tiering, DLP, no consumer messaging, insider monitoring; pre-agreed incident plan | Any policy override attempt |
| R-6 | **Trust collapse after one public error** (a wrong quote reaching a customer) | Medium | High | Draft-only, approval gates, numeric consistency checks, disclaimer on drafts | Rising correction rate |
| R-7 | **Cost overrun on inference** (big model used for everything) | Medium | Medium | Model routing, budgets, small-model-first | Cost per task rising month over month |
| R-8 | **Key-person dependency** (one engineer understands the stack) | Medium | High | Documentation, pair rotation, vendor support contract, reproducible deploys | Bus factor = 1 |
| R-9 | **Regulatory/contractual surprise** (customer forbids AI processing; DPDP obligations land) | Medium | High | Legal review now (Q-09, Q-13); design so cloud tier can be switched off | Contract review findings |
| R-10 | **Vendor lock-in** (agent platform or model provider becomes un-exitable) | Medium | Medium | Open-weight models, OpenAI-compatible abstraction, self-hosted runtime, exit clause | Cannot run without one provider |
| R-11 | **Scope creep to "AGI for the factory"** | High | High | One capability per phase, with exit criteria; a written "not now" list | New use cases outnumber closed ones |
| R-12 | **Governance theatre** (approvals rubber-stamped) | High | High | Measure approval latency; rotate approvers; sample audits | Median approval <10 s |

### 2.2 Technical risks specific to this design

| # | Risk | Mitigation |
|---|---|---|
| R-13 | Prompt injection through vendor PDFs/drawings | Untrusted-content handling, output validation, no write tools on reading agents, red-team suite |
| R-14 | Hallucinated numbers in a quote | Deterministic costing engine; numbers injected as template slots; consistency check |
| R-15 | Vision model misreads drawing callouts | Per-field confidence; DXF/STEP parsing for geometry; mandatory human confirmation on cost-driving fields |
| R-16 | ASR mis-hears a quantity or part number | "Heard → shown → confirmed" flow; no voice-only actions |
| R-17 | Retrieval returns a document the user should not see | ACL pushdown; fail closed; per-class indexes |
| R-18 | Agent loop burns budget overnight | Budgets, concurrency caps, spend alerts, kill switch |
| R-19 | Model update changes behaviour silently | Signed manifests, pinned versions, eval gate before promotion |
| R-20 | Audit log unrecoverable after an incident | Hash chain + WORM + offsite; quarterly restore verification |

---

## 3. Architecture alternatives — decision summary

Full comparison in `01-target-architecture.md` §4–5. Summary for decision:

| Option | Confidentiality | Capability | Cost | Speed | Verdict |
|---|---|---|---|---|---|
| **A. Full on-prem / air-gapped** | Strongest | Good, lagging frontier | Highest capital + staff | Slowest | Mandatory if contracts demand it; otherwise over-engineered for C2 work |
| **B. Hybrid, separated tiers** ← recommended hypothesis | Strong, testable boundary | Near-frontier where allowed | Medium–high | Medium | Best fit if D-1 permits C1/C2 cloud processing |
| **C. Cloud-first, India region** | Contract-dependent | Frontier | Lowest capital | Fastest | Acceptable for a pilot on non-sensitive data; weak long-term story for C4 |
| **O-02. Read-only copilot** | Strong | Limited | Low | Fastest | Good Phase 1, insufficient as a target |
| **O-04. Vendor agent platform** | Provider-dependent | High | Per-seat, scaling badly | Fastest | Accelerator for the UI layer only; keep the enclave |

**No option is "secure".** B is recommended because it makes the boundary *verifiable*, not because it is safe by assertion.

---

## 4. Autonomy ladder (the governance core)

Every capability gets a level. Levels are per-capability, not per-agent, and they expire.

| Level | Name | What the AI may do | Human role | Evidence required to enter |
|---|---|---|---|---|
| **L0** | Observe | Read and analyse; produce nothing that leaves the system | None (shadow mode) | Signed manifest; read-only tools; eval suite exists |
| **L1** | Advise | Produce drafts, memos, recommendations to a human; nothing committed | Reviews every output | Field accuracy ≥ threshold on golden set; zero harmful outputs in red-team |
| **L2** | Prepare & approve | Compose a complete action; a human approves the exact payload; system executes | Approves/rejects with reason code | ≥100 L1 cases, correction rate below target, approval latency healthy |
| **L3** | Act & notify | Execute bounded, reversible actions within policy; notify a human after | Audits samples, handles exceptions | ≥3 months at L2, zero critical incidents, rollback tested, CEO written authorization |
| **L4** | Autonomous in policy | Execute within defined policy limits, no per-action human | Monitors dashboards, sets policy | Separate board-level decision; insurance/contract review; periodic re-certification |

**Rules:**

1. **Promotion requires evidence + a named approver + an expiry date.** No permanent grants.
2. **Demotion is automatic on trigger:** a critical incident, a metric breach, or a model change without re-evaluation drops the capability one level until re-certified.
3. **L3/L4 never cover irreversible or contractual actions** (customer commitments, payments, statutory filings) in year one.
4. **The ladder is published internally.** People behave differently when they can see what the system is allowed to do.
5. **The CEO signs L3+.** That signature is the accountability mechanism, and it should feel heavy.

Proposed year-one assignment: everything at L0→L1; task routing/reminders at L2; nothing at L3 for commercial outputs.

---

## 5. Phased roadmap

Each phase has an exit criterion. **Do not start a phase before the previous exit criterion is met** — this is the single most effective scope control available.

### Phase 0 — Discovery and decision (4–6 weeks) — no AI build

| Work | Output |
|---|---|
| Answer Q-01..Q-08 (ERP and environment facts) | Completed fact sheet |
| Legal/contract review of data clauses, DPDP, CERT-In applicability **[FACT F-03/F-04]** | Written legal position |
| Data classification workshop (C1–C5) with real document samples | Classification policy v1 + labelled sample set |
| Select first capability and name its process owner | One-page capability charter |
| Threat model v1 | Threat register |
| Decision on D-1..D-5 | Signed decision record |

**Exit:** facts documented, legal position written, classification policy approved, first capability chartered, D-1 and D-2 signed.

### Phase 1 — Read-only ground truth (6–10 weeks)

| Work | Output |
|---|---|
| ERP read facade (M1): 5–8 read contracts | Working read-only API with audit |
| Document ingestion for one domain (e.g. quotations + drawings) with ACL + classification | Hybrid index with per-chunk ACL |
| Golden set: 100–150 historical cases, human-verified | Evaluation harness with baseline scores |
| Cockpit v0: ask a question, get an answer with citations | Usable read-only copilot |
| Red-team injection suite v1 | Pass/fail baseline |
| Kill switch, audit ledger v0, hash chain | Demonstrable tamper-evidence |

**Exit:** a real user answers a real question with citations in <60 s; zero C4 content reaches an unapproved endpoint in tests; golden-set baseline recorded.

### Phase 2 — First advisory capability (8–12 weeks)

Target: **enquiry-to-quotation assistant**, L1.

| Work | Output |
|---|---|
| Enquiry intake + drawing reader + comparable finder agents | Structured extraction with per-field confidence |
| Deterministic costing engine v1 (material, process, overhead, margin) | Versioned, reproducible cost calculation |
| Quote drafter bound to computed numbers | Draft with numeric slots, no free-text arithmetic |
| Approval service + decision ledger | Approval records with reason codes |
| Model routing by class + on-prem enclave (if D-1 = B/A) | Enforced tiering |

**Exit:** on 50 unseen historical enquiries, field accuracy and cost deviation within agreed tolerance; every draft number traceable; approvers using reason codes.

### Phase 3 — Narrow write path, L2 (8–12 weeks)

| Work | Output |
|---|---|
| `quotation.create_draft` command via facade, idempotent, approval-gated | Draft quotes created in ERP with audit |
| Reconciliation job (derived vs ERP) | Daily discrepancy report |
| Task coordinator + follow-up engine | Tasks created from decisions and ERP events |
| Voice notes (push-to-talk) with confirm-on-screen | Working voice capture for shop/office |
| Vendor analyst + PO expediter (L1/L2 messaging only) | Follow-up value visible |

**Exit:** zero duplicate records from retries; ≥80% of AI-drafted quotes accepted with edits; task closure evidence captured; no audit gaps.

### Phase 4 — CEO cockpit and cross-domain (12 weeks)

| Work | Output |
|---|---|
| CEO Agent with decision memos | Weekly programme briefing, blocker detection |
| Parallel fan-out investigations ("why is X late?") | Bounded multi-domain diagnosis |
| Report builder replacing one existing report exactly | A report the CEO actually reads |
| Quality / stores / PPC analysts | Trend and shortage intelligence |
| Drift monitoring, autonomy registry | Evidence for promotion decisions |

**Exit:** CEO uses the briefing instead of asking for it; three capabilities meet L3 evidence thresholds (but are not promoted yet).

### Phase 5 — Selective L3 and hardening (ongoing)

| Work | Output |
|---|---|
| Internal task routing and reminders at L3 (bounded, reversible) | Measured reduction in missed follow-ups |
| ISO/IEC 42001-aligned governance review **[FACT F-05]** | Documented AI management system |
| Third-party penetration test and adversarial AI assessment | Findings + remediation |
| DR drills, restore verification, incident tabletop | Tested recovery |
| Model/version upgrade process with eval gates | Repeatable upgrade path |

**Exit:** independent assessment passed; recovery drills documented; L3 promotion signed by CEO.

### Phase 6 — Expansion (only after 5)

Additional domains (accounts, HR reporting, deeper PPC), possible cloud burst for C2, possible fine-tuning on a curated dataset. **Not in scope until Phase 5 exits.**

### Timeline realism

Roughly **9–15 months** from Phase 0 start to end of Phase 4, for a company without an existing AI platform team, assuming ERP access cooperation and process-owner availability. Any plan promising the full ten-domain vision in under a year is either not doing the governance or not measuring quality.

---

## 6. Unresolved questions (the deliverable's "we don't know" list)

Grouped by who can answer them.

### 6.1 Only the ERP/IT team can answer **[UNKNOWN — blocking]**

| ID | Question | Why it blocks |
|---|---|---|
| Q-01 | Technology stack, framework, language, and who maintains it | Determines feasibility of every integration option |
| Q-02 | Database engine, version, hosting, size, read-replica feasibility | Determines M1 feasibility and query safety |
| Q-03 | Does any documented API/service layer exist? Auth model? | Determines M2 and the whole Phase 1 timeline |
| Q-04 | Module boundaries and true system of record per domain (costing, BOM, stock, quality) | Prevents building against the wrong source |
| Q-05 | Where drawings/documents live and how they link to records | Determines ingestion design |
| Q-06 | Is there an existing directory/SSO, or local ERP accounts? | Determines the identity architecture |
| Q-07 | Network topology: VLANs, firewall rules, server-room capacity, internet path | Determines zone design and enclave feasibility |

### 6.2 Only the CEO/business can answer

| ID | Question |
|---|---|
| Q-08 | Which contracts restrict processing location, sub-processors, or AI use? |
| Q-09 | Which single capability matters most in the next two quarters? (D-2) |
| Q-10 | Who is accountable for AI risk, and who signs autonomy promotions? (D-5) |
| Q-11 | What delegation limits apply (amounts, quantities, customer commitments)? |
| Q-12 | Which reports does the CEO actually read today, and in what format? |
| Q-13 | Is the workforce likely to accept an in-app task inbox, or will it need a chat tool? |
| Q-14 | What language mix do shop-floor and office staff use in written and spoken communication? |
| Q-15 | What is the budget envelope and the value deadline for the first capability? |

### 6.3 Require legal/security counsel

| ID | Question |
|---|---|
| Q-16 | Does DPDP apply to our processing of employee/customer data, and in what role (fiduciary/processor)? **[FACT F-03]** |
| Q-17 | Are we a covered entity under CERT-In Directions, and what is our incident-reporting runbook? **[FACT F-04]** |
| Q-18 | Do customer/programme contracts prohibit or condition AI processing, cloud processing, or recording of calls? |
| Q-19 | What retention periods are required vs permitted for decision records containing commercial data? |
| Q-20 | What security assurances (certifications, audit rights, segregation) do our defence customers require of sub-processors? |

### 6.4 Require measurement, not an answer

| ID | Question |
|---|---|
| Q-21 | Actual prompt sizes, interaction volumes and concurrency (needed for sizing) |
| Q-22 | Peak concurrent users and acceptable latency for each workflow |
| Q-23 | Real document volumes, scan quality, and OCR failure rate |
| Q-24 | Actual data-quality defect rates in part masters, vendor masters and BOMs |
| Q-25 | Baseline human effort and error rates for the first capability (to prove value) |

---

## 7. Minimum information request (what to send us / your architects next)

A concrete checklist. Items marked ★ are blocking; everything else can follow.

### 7.1 ERP and data

- ★ ERP architecture summary (1–2 pages): language/framework, database engine + version, hosting model (on-prem/cloud), deployment method, who maintains it.
- ★ Module list with one-line responsibility each; note where costing, BOM, stock and quality actually live.
- ★ Whether an API/service layer exists; if yes, its documentation and auth mechanism. If no, state that plainly — it is a legitimate answer and changes the plan.
- ★ Database size, largest tables, and whether a read replica or reporting views are possible.
- ★ Document storage: where drawings/specs/PDFs live, naming conventions, how they link to ERP records.
- Sample (anonymised) records: 5 enquiries, 5 quotations with cost breakdowns, 5 BOMs, 5 drawings, 5 POs, 5 NCRs.
- Data dictionary for the entities above (field names + meaning), even if informal.
- Known data-quality issues (duplicate vendors, part-number conventions, missing fields).

### 7.2 Environment

- ★ Network diagram: office, shop floor, servers, internet egress, any cloud tenant.
- ★ Server-room capacity: power, cooling, rack space available for a GPU node.
- ★ Existing identity provider, MFA coverage, endpoint management status.
- Existing backup/DR arrangements and last successful restore date.
- Existing SIEM/logging, if any; log retention in practice.

### 7.3 Business and governance

- ★ Data classification: do you already have one? Samples of what is C4 vs C3 in practice.
- ★ Contractual constraints on data location and third-party processing (from legal/contracts).
- ★ Named process owners for: sales/estimation, engineering, purchase, PPC, quality, stores, accounts.
- Target capability and success measure for the first 6 months.
- Budget envelope and procurement constraints (capex vs opex preference).
- Any existing AI usage (staff using ChatGPT etc.) and current acceptable-use policy.

### 7.4 What we will deliver once these arrive

A revised architecture with the integration design *actually specified* (contract list, adapter choice per module, event strategy), a measured hardware sizing based on your real prompt/document sizes, a costed Phase 1 plan with named owners, and an updated risk register. Until then, this report remains a pattern library and a decision framework — which is the honest maximum value available without access.

---

## 8. What this report did not do

- Did not inspect, test, or access ELITE AQMS-ERP in any form.
- Did not benchmark any model on your data; all performance figures are cited public references or estimates.
- Did not verify your legal position on DPDP, CERT-In, or customer contracts.
- Did not produce a security assurance, penetration test, or compliance certification.
- Did not select vendors, models, or hardware; it names candidates to be evaluated against your constraints.
- Did not write code or produce an implementation plan at task level.

Those are the next steps, and each requires information or authorization this exercise did not have.
