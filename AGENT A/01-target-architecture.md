# 01 — Target Architecture, Infrastructure Placement, ERP Integration

Covers assignment sections **A** (overall architecture), **G** (local vs cloud vs hybrid) and **E** (ERP integration without compromising ERP logic).

---

## 1. Architecture at a glance (A)

### 1.1 The three-plane model

The system is described as three planes. This separation is what allows confidentiality, audit and change control to be reasoned about independently.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PLANE 1 — EXPERIENCE                                                         │
│  CEO Cockpit (web)  │  Employee task inbox / chat  │  Voice notes  │  Reports │
│  Mobile view        │  Email digest               │  Approval queue          │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │  authenticated, per-user, no direct model access
┌───────────────────────────────▼──────────────────────────────────────────────┐
│  PLANE 2 — DECISION (the "AI operating layer")                                │
│                                                                              │
│   ┌─────────────────┐    ┌──────────────────────┐   ┌────────────────────┐   │
│   │  CEO Agent      │───▶│  Orchestrator        │──▶│  Specialist agents │   │
│   │  (coordinator)  │◀───│  (durable workflows) │◀──│  (10 domains)      │   │
│   └─────────────────┘    └──────────────────────┘   └────────────────────┘   │
│            │                        │                        │              │
│   ┌────────▼────────────────────────▼────────────────────────▼───────────┐  │
│   │  CONTROL SERVICES                                                    │  │
│   │  Policy engine (authorization) │ Approval service │ Autonomy registry │  │
│   │  Tool gateway (scoped access)  │ Model router     │ Eval & guardrails │  │
│   └────────┬────────────────────────┬────────────────────────┬────────────┘  │
│            │                        │                        │              │
│   ┌────────▼──────────┐  ┌──────────▼─────────┐  ┌───────────▼───────────┐   │
│   │ Knowledge services│  │ Memory services    │  │ Audit & telemetry     │   │
│   │ hybrid index, RAG │  │ decisions, cases   │  │ ledger, traces, SIEM  │   │
│   └───────────────────┘  └────────────────────┘  └───────────────────────┘   │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │  only path to data; every call named + scoped
┌───────────────────────────────▼──────────────────────────────────────────────┐
│  PLANE 3 — SYSTEMS OF RECORD & REALITY                                        │
│  ELITE AQMS-ERP (authority) │ Files/drawings │ Mail/calendar │ Shop-floor     │
│  Accounting package?        │ Vendor portals│ Instruments   │ Documents      │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Non-negotiable properties of this shape**

1. **Single choke point.** All data movement out of Plane 3 passes through the integration facade (Section 3). There is no second path, no direct database link from an agent container, no "temporary" CSV drop.
2. **Control services are not optional middleware.** Policy, approval, autonomy and routing sit *in the call path*. If they are unavailable, the call fails closed.
3. **Plane 2 holds no authority of its own.** It holds *delegated* authority, resolved per request, and it can be switched off without touching the ERP.

### 1.2 Component responsibilities

| Component | Responsibility | Explicitly NOT responsible for |
|---|---|---|
| **CEO Cockpit** | Objectives, briefings, decision memos, approval actions, programme risk view | Editing business records |
| **Orchestrator** | Durable task graphs, retries, timeouts, parallel fan-out, human waits, compensation | Business rules; authorization decisions |
| **CEO Agent** | Decomposition, prioritisation, blocker detection, memo synthesis, escalation | Direct tool execution on write paths |
| **Specialist agents** | Domain reasoning and drafting within one bounded scope each | Cross-domain data access; autonomy policy |
| **Tool gateway** | The only executable surface for agents: typed, versioned, allow-listed tools; mints scoped tokens; validates arguments; enforces rate/row/amount limits | Business logic; storing data of record |
| **Policy engine** | Decisions of the form "may principal P perform action X on resource R in context C" | Authentication |
| **Approval service** | Human decision records: who, what, evidence shown, quorum, deadline, outcome, reason | Being an optional notification |
| **Autonomy registry** | The authoritative table of which capability is at which autonomy level, approved by whom, with expiry | Runtime enforcement (policy engine enforces) |
| **Model router** | Maps data class + task class → approved model endpoint; blocks disallowed combinations | Content moderation (separate guard layer) |
| **Eval & guardrails** | Pre-deployment evaluation suites, online guard checks, injection detection, output schema validation | Guaranteeing correctness |
| **Knowledge services** | Ingestion, classification inheritance, chunking, hybrid index, retrieval with ACL pushdown | Deciding who may see a document (that is policy) |
| **Audit ledger** | Append-only, hash-chained record of every consequential event | Debug logging (separate, shorter retention) |

### 1.3 Data classification spine

Everything downstream — routing, retention, retrieval, autonomy, audit detail — keys off one classification scheme. Proposed:

| Class | Label | Examples | Permitted processing zone | Default autonomy ceiling |
|---|---|---|---|---|
| C1 | Public | Marketing material, published specs | Any approved model | L3 |
| C2 | Internal | Internal procedures, generic cost structures, org data | On-prem or India-region cloud | L2 |
| C3 | Confidential–Commercial | Customer pricing, margins, vendor rates, contracts, NDA'd drawings | On-prem enclave or dedicated India-region tenant with contractual no-training terms | L1–L2 |
| C4 | Restricted–Programme | Defence programme drawings, specifications, quantities, end-use, offsets | **On-prem enclave only, no internet egress** | L0–L1 |
| C5 | Secrets | Credentials, keys, signing material, HR/medical records | Never in a prompt, never in an index | n/a |

Two rules that make this work in practice:

- **Classification is inherited, not typed by hand.** A chunk, embedding, summary, cache entry, transcript or agent output inherits the maximum class of its inputs. Uplifting is automatic; downgrading requires an explicit, logged human action.
- **Downgrade is a workflow, not a permission.** Moving C3 content to a C2 pipeline requires a named approver and a recorded reason — because that is exactly the step an attacker will try to induce.

### 1.4 Deployment zones

```
 ZONE 0  Production/OT & ERP core          (existing; untouched network)
   │  read-only replica / facade queries only  ▲  commands via signed API
 ZONE 1  Confidential AI enclave (on-prem)  ── no internet egress ──
   │  GPU inference, vector store, object store, orchestrator, policy, audit
   │  one-way export of *policy-approved* artefacts only (DLP-scanned)
 ZONE 2  Corporate IT / user endpoints      ── internet via proxy ──
   │  Cockpit UI, IdP, mail, SIEM ingestion
 ZONE 3  Approved cloud (India region)      ── for C1/C2 only ──
      burst inference, evaluation lab, backup of non-restricted telemetry
```

The critical control is that **Zone 1 can reach Zone 0 (read) and Zone 2 (serve) but not Zone 3**. Any exception is a documented, time-boxed, DLP-mediated export with an approval record.

### 1.5 What the layer deliberately does not do

- It does not become a second ERP. It stores derived artefacts (embeddings, extracted fields, decision records, draft documents), never master data.
- It does not own the quotation, BOM or stock truth. Those stay where they are.
- It does not silently reconcile. A mismatch between AI-derived and ERP values is surfaced as a discrepancy task, not auto-corrected.
- It does not chat with untrusted external parties on your behalf.

---

## 2. Request lifecycle (how one task actually flows)

Worked example: *"Prepare a quotation for enquiry ENQ-2291 for Tata-style bracket assembly, 4,000 off."*

| Step | Plane | What happens | Control applied |
|---|---|---|---|
| 1 | 1 | User submits objective in Cockpit | SSO, MFA, session bound to device posture |
| 2 | 2 | CEO Agent builds a task graph: parse enquiry → fetch drawings → find comparable jobs → extract features → request costing → check capacity → draft quote → assemble memo | Governor limits: max 40 tool calls, max 25 min, max token budget |
| 3 | 2 | Policy engine evaluates each planned tool call before execution | Agent's manifest must include the tool; data class must permit it; amount thresholds checked |
| 4 | 2 | Tool gateway calls facade: `enquiry.get`, `drawing.list`, `jobcost.history.matches` | Read-only, row-limited, field-masked, audience-bound token |
| 5 | 2 | Knowledge services retrieve comparable quotations with ACL pushdown | Only records the requesting user may see |
| 6 | 2 | Extraction agent returns structured features **plus confidence and source citations** | Output schema validated; low confidence routed to human |
| 7 | 2 | **Deterministic costing engine** computes material, process, overhead, margin per approved rules | No LLM arithmetic; engine is versioned and reproducible |
| 8 | 2 | Drafting agent composes quote narrative from computed numbers only | Numbers injected as immutable template slots, not free text |
| 9 | 2 | Guard checks: schema, numeric consistency, classification, injection artefacts | Fails → re-draft once → then human |
| 10 | 2 | Approval service creates an approval request with full evidence bundle | L2: a human must approve before any ERP write |
| 11 | 1 | CEO/estimator reviews: numbers, sources, comparables, confidence, risks | Time-boxed SLA; escalation on expiry |
| 12 | 2/3 | On approval: `quotation.create_draft` command to ERP, idempotent, with approval ID | Reversible (draft only); full audit entry |
| 13 | 2 | Decision record written: inputs, options, recommendation, human edits, outcome | Becomes a *precedent*, not a rule (see `02` §3) |
| 14 | 2 | Later: won/lost + actual vs quoted cost feeds the learning loop | Requires human validation before influencing future suggestions |

The important observation: **most of the steps are not AI**. The AI does steps 2, 5, 6, 8, 9 and 12-partially. Steps 3, 4, 7, 10, 13 are deterministic governance machinery. That ratio is the design.

---

## 3. ERP integration strategy (E)

> We do not know the ERP's stack, schema, API surface or hosting. Everything below is a *pattern* with decision branches, not a claim about ELITE AQMS-ERP.

### 3.1 The governing rule: an anti-corruption layer

Between the AI layer and the ERP we place a **facade** (anti-corruption layer) that:

- exposes a **stable, versioned, intention-revealing contract** (`enquiry.get`, `bom.explode`, `jobcost.history.matches`, `quotation.create_draft`, `vendor.po.status`), never table shapes;
- translates to whatever the ERP actually offers (SQL view, stored procedure, REST/SOAP endpoint, file drop, screen automation as last resort);
- **owns all mapping** between ERP concepts and AI-layer concepts, so an ERP schema change breaks one adapter, not forty prompts;
- enforces limits (max rows, max amount, max frequency) per tool;
- records every call with principal, purpose, arguments hash, row count and response hash.

This is the "strangler-adjacent" approach: we wrap, we do not rewrite. The ERP keeps running exactly as it does today.

### 3.2 Four access modes, in increasing risk

| Mode | Mechanism | Use it for | Guardrails | Risk |
|---|---|---|---|---|
| **M1 Read replica** (preferred for analytics) | Read-only DB replica or materialised views, queried by the facade | Reporting, comparables, dashboards, backfills | Dedicated read-only account, statement timeout, row caps, no `SELECT *`, no ad-hoc SQL from agents | Low–medium: replica may lag; sensitive columns present |
| **M2 Facade API over existing endpoints** | ERP's own APIs / stored procedures wrapped | Live reads and narrow writes | Versioned contract, idempotency keys, schema validation | Medium: depends on API stability |
| **M3 Business events** | Transactional outbox in ERP DB, or log-based CDC (Debezium), onto a broker | Keeping derived state (search index, caches, task state) fresh; blocker detection | Publish business events where possible, not raw row diffs; idempotent consumers; schema registry | Medium: CDC leaks schema and can double-fire one business action; outbox needs a write into the ERP DB — **a decision, not a default** |
| **M4 Direct DB write / screen automation** | Writes into ERP tables, or RPA against ERP screens | **Not recommended** | If unavoidable: single service account, change-log table, nightly reconciliation, kill switch | High: bypasses ERP business logic, breaks invariants, unpatchable |

**Our recommendation:** M1 + M2 for Phase 1–3. M3 only after the ERP team confirms there is a safe place to publish events (a dedicated outbox table is a schema change and requires their sign-off). **M4 prohibited** for AI-initiated actions; acceptable only for a short, human-supervised migration utility with reconciliation.

**Why not "just let the agent query the database":** three concrete reasons, all of which apply regardless of your ERP's quality.

1. **Business rules live in code, not tables.** An ORDER table has statuses with meanings, invariants, triggers and downstream effects. Reading a status is not knowing what it means; writing one bypasses the logic that makes it valid.
2. **Authorization is not row-level in most ERPs.** A DB credential that can read one enquiry can usually read all of them. You cannot express "this estimator, this programme, this customer" in a connection string.
3. **Prompt injection becomes data exfiltration.** If a model can emit SQL, a malicious drawing note or vendor email can attempt to make it emit SQL. With a typed tool surface, the worst case is a badly-chosen *allowed* call.

### 3.3 The integration contracts you will need (provisional)

To be refined once Q-01..Q-06 are answered. Indicative only.

| Domain | Read contract | Write contract (narrow, idempotent, approval-gated) |
|---|---|---|
| Sales / enquiries | `enquiry.get`, `enquiry.search`, `enquiry.attachments` | `enquiry.note.append`, `quotation.create_draft` |
| Engineering / drawings | `drawing.list`, `drawing.metadata`, `drawing.fetch_derived` (PDF/PNG preview only) | none initially |
| BOM | `bom.get`, `bom.explode` (deterministic, ERP-side) | `bom.suggest` → human applies |
| Costing | `cost.history.by_part`, `cost.rate_cards` | none (costing engine is separate, versioned) |
| Vendor / purchase | `vendor.performance`, `po.status`, `rfq.list` | `followup.create_task` (task store, not ERP) |
| PPC / manufacturing | `workorder.status`, `schedule.capacity`, `delay.reasons` | `escalation.create` |
| Quality | `inspection.results`, `ncr.list`, `capex.trend` | `ncr.draft_note` |
| Stores | `stock.on_hand`, `stock.movements`, `shortage.list` | none |
| Accounts | `invoice.status`, `receivable.aging` (aggregated, masked) | none |
| HR / tasks | `employee.directory` (minimum fields), `task.*` (our own store) | `task.assign`, `task.remind` (our store) |

Note the asymmetry: **read breadth, write narrowness**. That is deliberate and should be a permanent property.

### 3.4 Keeping the ERP's logic intact — six concrete mechanisms

1. **No schema coupling.** The facade maps fields; prompts never embed column names.
2. **No business-rule re-implementation on the AI side for anything that commits.** If the ERP validates a BOM, the AI proposes; the ERP validates.
3. **Derived, never authoritative.** Search indexes, embeddings and caches are rebuildable from the ERP at any time. If the AI layer is deleted tonight, the ERP is unaffected.
4. **Idempotency everywhere.** Every write carries a client-generated key; retries cannot double-create. This matters because agent retries are normal, not exceptional.
5. **Reconciliation as a first-class job.** Daily comparison of derived state vs ERP truth, with discrepancy reporting. Silent drift is the failure mode that destroys trust.
6. **Sandbox first.** A restored copy of the ERP (anonymised where personal data is involved) is the only place integration changes are tested. Never test write paths in production.

### 3.5 Integration anti-patterns to refuse, explicitly

| Anti-pattern | Why it is refused |
|---|---|
| Agents given a read-write DB login | Unbounded blast radius, no per-action authorization, no audit granularity |
| LLM generating SQL against production | Prompt injection becomes database access; results unauditable |
| Screen-scraping/RPA as the primary integration | Brittle, invisible to ERP audit, breaks on UI change, cannot be authorized per field |
| Dual writes (write ERP + write our DB) without outbox/idempotency | Guaranteed inconsistency under failure |
| "Sync everything" nightly full extract | Confidentiality sprawl and no freshness; extract only what a contract needs |
| Agent-to-agent free-form messaging with no schema | Unbounded prompt-injection propagation; unauditable |

### 3.6 What we must learn about the ERP before designing integration (summary)

Detailed in `04-roadmap-costs-risks-questions.md` §7. The six blocking ones:

- **Q-01** Technology stack, language, framework, and whether it is maintainable by a team we can talk to.
- **Q-02** Database engine, version, hosting, size, and whether read replicas are feasible.
- **Q-03** Whether *any* documented API/service layer exists, and its authentication model.
- **Q-04** Module boundaries and the true system of record per domain (is costing in ERP? in Excel? both?).
- **Q-05** Document storage: where drawings and PDFs live, and how they are linked to records.
- **Q-06** Identity: is there an existing directory/SSO, or is ERP login local accounts?

---

## 4. Local vs cloud vs hybrid AI infrastructure (G)

### 4.1 The three options, honestly compared

| Dimension | **Option A — Fully on-prem / air-gapped** | **Option B — Hybrid, separated tiers** | **Option C — Cloud-first, India region** |
|---|---|---|---|
| Shape | GPU cluster in Zone 1; no egress; models imported by media | Zone 1 enclave for C3/C4; Zone 3 India-region for C1/C2 burst and evals | All processing in an India-region tenant, VPC-isolated, private endpoints |
| Confidentiality | Strongest achievable control: data physically cannot leave | Strong *if* the tier boundary is real and tested; the boundary is the risk | Depends entirely on contract, tenancy and provider configuration |
| Model capability | Best open-weight models, weeks-to-months behind frontier; you own quantisation trade-offs | Near-frontier for non-restricted work; open-weight for restricted | Frontier, continuously updated |
| Model refresh | Manual, gated, slow (media transfer + validation) | Mixed: fast in cloud tier, slow in enclave | Fast |
| Capital cost | Highest upfront (GPUs, power, cooling, redundancy) | Medium–high (small enclave + cloud opex) | Lowest upfront |
| Operating cost | Power, space, MLOps staff, hardware refresh every 3–4 years | Both opex and staff | Predictable per-token opex; can spike |
| Skills required | GPU ops, Kubernetes, inference tuning, security ops | Both on-prem and cloud ops | Cloud + AI engineering |
| Auditability | Fully under your control; easiest to evidence | Split; needs correlated audit across zones | Provider-dependent; you get logs, not custody |
| Time to first value | Slowest (procurement lead times) | Medium | Fastest |
| Failure modes | Hardware/thermal/power; single-site risk; key-person risk | Boundary leakage; configuration drift between tiers; complexity | Outage, quota, price/model change, contract change |
| Defence-contract friendliness | Easiest to defend to a customer's security questionnaire | Defensible if segregation is documented and tested | Hardest to defend for C4; likely unacceptable for some programmes |

### 4.2 Why we recommend Option B as the *default hypothesis* (not a decision)

- It matches the actual shape of the data: a small fraction of your work is truly restricted; most is ordinary commercial. Forcing everything into air-gap buys confidentiality you may not need for C2 and pays for it in capability and staff.
- It creates a **testable boundary**. "Restricted data never enters the cloud tier" is a property you can verify with network policy, DLP and an audit test — unlike "the cloud provider is trustworthy".
- It lets the enclave stay small: a single 8-GPU node can serve C3/C4 workloads for a company of this size, while evaluation and burst work run elsewhere.
- It degrades gracefully: if contracts later forbid all cloud processing, Zone 3 is switched off and the enclave remains — no re-architecture.

**When to choose A instead:** if contractual review concludes that *any* transmission of customer drawings or programme data to a third-party processor is prohibited, or the customer's security questionnaire requires physical segregation. Then A is not a preference, it is the only compliant option — and the cost/benefit in §4.4 applies.

**When C might be acceptable:** if, after classification, C3/C4 volume is genuinely tiny and the business accepts that those specific workflows stay manual or on a single workstation-class local model. This is the cheapest path and the weakest confidentiality story; suitable for a pilot, not for the target state.

### 4.3 The enclave, concretely

| Layer | Proposal | Why |
|---|---|---|
| Compute | 1 node, 8× datacentre GPU (H100/H200-class or L40S-class depending on model size), NVMe local scratch, 10/25 GbE to Zone 0 | Single-node avoids multi-node interconnect complexity and keeps restricted data on one controlled box |
| Inference | vLLM behind an authenticating reverse proxy; or NVIDIA NIM if air-gapped packaging and support are wanted | **[FACT F-06/F-07]** vLLM's own API key does not cover all endpoints; NIM documents air-gap staging |
| Models | One 30–70B-class instruction model (reasoning, drafting), one small 3–8B model (classification, extraction, routing), one embedding model, one reranker, one ASR model | Small model does the volume; big model does the thinking. Running everything on the big model is the most common cost mistake |
| Storage | Encrypted block storage for models/vectors; WORM object store for audit and signed artefacts; no cloud sync | Audit evidence must be immutable and local |
| Network | Dedicated VLAN, no default route to internet, NTP from approved internal source, egress only via DLP-mediated export | **[FACT F-04]** clock sync matters for log correlation |
| Orchestration | Kubernetes (or a single-node container runtime if simplicity wins) with network policies; separate namespaces per data class | Blast-radius control |
| Identity | Workload identity for every service (SPIFFE/SPIRE or mTLS with short-lived certs); no static service passwords | See `03` §1 |
| Physical | Locked rack/cage, access log, UPS + generator, temperature monitoring | A GPU node that overheats is an availability incident, not an AI problem |

### 4.4 Indicative hardware sizing (worked example) — **[ESTIMATE]**

Assumptions: 250 total employees, 60 regular AI users, 15 concurrent peak, average 6k input + 700 output tokens per interaction, plus background jobs (nightly extraction, indexing) . These are guesses until measured (Q-19).

| Workload | Model class | Indicative footprint |
|---|---|---|
| Routing / classification / field extraction | 3–8B, quantised | 1 GPU (or CPU for low volume) |
| Reasoning, drafting, memo synthesis | 30–70B, FP8/INT8 | 2–8 GPUs depending on size and latency target |
| Embeddings + reranking | small encoders | 1 GPU shared, or CPU |
| Speech-to-text | Whisper-class / IndicConformer | 1 GPU (batch-tolerant) |
| Headroom for failover and maintenance | — | +1 node or N+1 GPU |

Reference point for calibration: NVIDIA's published reference architecture reports roughly **137 concurrent users** on a **single H100 NVL** for Llama-3.1-8B at ~1,000 ms time-to-first-token in a documented configuration **[FACT]**. That is an 8B model on one GPU — a 70B model needs an order of magnitude more hardware for the same concurrency. Do not size from vendor "users per GPU" claims without measuring your own prompt lengths.

Practical sizing rule: **measure, then buy.** Run a two-week benchmark on rented India-region GPUs or a vendor loaner with your own prompts and documents. Cost of the benchmark **[ESTIMATE]** is small compared with buying the wrong node.

### 4.5 Model routing policy (the mechanism that makes tiering real)

```
request → classify data (inherited class) + task type
        → look up routing table (policy file, signed, versioned)
        → allowed endpoint?  ── no ──▶ refuse (fail closed) + log + notify owner
                             └ yes ──▶ call, with per-endpoint quotas and no-retention flags
```

Routing table sketch:

| Data class | Reasoning model | Embeddings | Allowed endpoints |
|---|---|---|---|
| C1 | best available | any approved | cloud or on-prem |
| C2 | best available | any approved | cloud (India region) or on-prem |
| C3 | on-prem large | on-prem | on-prem only |
| C4 | on-prem large | on-prem | on-prem only, no egress |
| C5 | — | — | never |

Enforcement detail that is easy to get wrong: the **class is computed server-side from the retrieved records**, never supplied by the client or the prompt. Otherwise a user (or an injected instruction) can label a C4 drawing as C1 and route it to the cloud.

### 4.6 What "approved cloud AI" must mean contractually

Technical data residency is necessary but not sufficient. Before any C2/C3 data touches a cloud model, obtain in writing:

- **No training on our data**, with the specific product/feature names covered (not a marketing page).
- **No human review of prompts/outputs** without explicit, per-incident consent, or a documented abuse-review process with notice.
- **Retention window** and deletion capability for prompts, outputs, logs, caches and abuse-monitoring copies.
- **Sub-processor list** and change notification.
- **Data location** commitments covering *processing* as well as storage (these differ; some features route cross-region).
- **Breach notification** timelines compatible with **[FACT F-04]** 6-hour CERT-In reporting — a provider that notifies in 72 hours cannot support a 6-hour obligation on its own.
- **Exit**: bulk export and deletion, and the ability to run without the provider.

Note the honest caveat: **[FACT F-15]** India-region processing exists for selected models and features, not universally. Feature-level gaps (hosted tools, some modalities) are common. Verify per feature, per model, at contract time — and re-verify at renewal, because these matrices change.

### 4.7 Cost comparison logic (not numbers yet)

The decision should be made on **three-year total cost of ownership at equal capability**, not sticker price:

`TCO = hardware + power/cooling + space + staff + licences + integration + refresh − avoided cloud spend`

Two effects usually dominate and are usually omitted:

- **Staff.** An on-prem GPU enclave needs someone who can debug a NCCL hang and patch a CVE in a container stack. If that person does not exist, the enclave's real cost is a hire.
- **Utilisation.** A GPU node bought for peak is idle most of the day. Cloud is expensive per token and cheap per idle hour; on-prem is the reverse. Hybrid exists precisely to arbitrage this.

See `04-roadmap-costs-risks-questions.md` §2 for indicative ranges.

---

## 5. Architecture alternatives (recorded, not decided)

| ID | Alternative | Summary | Strong when | Weak when |
|---|---|---|---|---|
| O-01 | **Governed decision layer + ERP facade** (this proposal) | AI advises; ERP remains authority; tiered models | Always — this is the baseline | Never rejected outright; scope varies |
| O-02 | **Read-only analytics copilot only** | No writes at all, no task automation; pure Q&A over a reporting replica | Very early stage, low trust, tight budget | Value plateaus; users still copy-paste into ERP |
| O-03 | **Workflow-automation-first** (rules + RPA + small models) | Deterministic automation for follow-ups and data entry; AI only for extraction | High-volume repetitive tasks with stable rules | Fails on judgement-heavy work (costing, drawing interpretation) |
| O-04 | **Vendor agent platform** (buy the stack) | Commercial agent platform with connectors | Speed, support contract, small team | Lock-in, data-residency gaps, limited control of audit internals, per-seat cost at scale |
| O-05 | **Full air-gapped autonomy** | Everything local, maximum isolation, broad autonomy | Contractually mandated isolation | Capability lag, cost, MLOps burden, slow improvement |
| O-06 | **Human-in-the-loop marketplace of narrow tools** | Many small single-purpose assistants, no central CEO agent | Organisational resistance to a "CEO agent" | Fragmentation, duplicated context, no cross-domain insight |

The recommended path is **O-02 → O-01 with narrow scope → broader O-01**, using O-03 patterns inside individual agents where the work is genuinely deterministic. O-04 is a legitimate accelerator for the experience layer if the enclave remains ours. O-05 becomes mandatory if D-1 resolves to "no cloud".

---

## 6. Open architecture questions raised by this section

| ID | Question |
|---|---|
| Q-01..Q-06 | ERP facts (see §3.6) |
| Q-07 | Is there an existing identity provider (AD/Entra/Google/LDAP) that can be the single source of human identity? |
| Q-08 | What is the network reality between shop floor, office and any cloud tenant — is a dedicated VLAN and a one-way export path feasible? |
| Q-09 | Which customer/programme contracts contain data-handling clauses that restrict processing location or sub-processors? (Legal review needed; we cannot answer this.) |
| Q-10 | Is there an existing document management or PLM system, or are drawings on file shares? |
| Q-11 | What is the realistic power/cooling/space budget for an on-prem GPU node, and is the server room rated for it? |
| Q-12 | Who is accountable for AI risk (D-5), and who signs autonomy promotions? |
