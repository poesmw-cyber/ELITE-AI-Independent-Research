# 02 — CEO Agent, Specialist Agents, Orchestration, Knowledge and Voice

Covers assignment sections **B** (CEO and specialist agent design), **C** (orchestration and parallel execution), **D** (shared knowledge and controlled learning), **H** (voice, employee communication, task follow-ups).

---

## 1. Agent design principles (B)

Before listing agents, the constraints that apply to all of them. These are what keep a "multi-agent system" from becoming an unauditable swarm.

| # | Rule | Rationale |
|---|---|---|
| AG-1 | **An agent is a configuration, not a program.** Each agent is defined by a versioned manifest: role, prompt template, allowed tools, data classes, output schema, autonomy level, evaluation suite, owner. | An agent you cannot describe in a file is an agent you cannot audit, roll back or promote. |
| AG-2 | **One agent, one domain, one output contract.** No agent spans sales and accounts "for convenience". | Cross-domain agents drift into god-objects with excessive agency (**[FACT F-01]** LLM06). |
| AG-3 | **Agents propose; services commit.** The only components that can change business state are typed tools behind the gateway, on an approved command. | Separates reasoning failure from data corruption. |
| AG-4 | **Every agent has a tool allow-list and a hard budget** (calls, tokens, time, rows, money). | Bounded blast radius; predictable cost. |
| AG-5 | **Agents never see credentials.** They request *actions*; the gateway holds authorization. | Prompt injection cannot exfiltrate what was never in the prompt. |
| AG-6 | **Outputs are schema-validated**, and validation failure is a first-class outcome (retry once, then escalate). | Free-text output into a downstream system is the root of most agent incidents. |
| AG-7 | **Agents are stateless per task**; context comes from explicit, access-controlled retrieval. | Prevents "memory" becoming uncontrolled data pooling across clearances. |
| AG-8 | **Two agents never negotiate free-form.** Handoffs use typed task objects. | Stops prompt injection propagating agent-to-agent (**[FACT F-01]**). |

### 1.1 The CEO Agent — what it is, and three things it must not be

**It is** a coordinator with four jobs:

1. **Intake and framing** — turn the CEO's objective ("why is D-17 late?", "price this enquiry", "where are we exposed this month?") into a task graph with explicit questions, sources and success criteria.
2. **Dispatch and supervision** — start specialist tasks (in parallel where independent), enforce budgets and deadlines, watch for contradictions between specialists.
3. **Synthesis** — produce a *decision memo*: the question, the answer, the evidence (with record IDs), the confidence, the options with trade-offs, the cost/schedule implications, and what would change the answer.
4. **Escalation and memory** — raise blockers, request approvals, and record what was decided and why.

**It must not be:**

- **An operator.** It never holds a write tool. It cannot "just fix" a PO. This single restriction removes the largest category of catastrophic failure and is worth the loss of convenience.
- **An oracle.** Every claim in a memo carries a source (ERP record, document page, or explicit "inference"). Unsourced confident prose from a CEO agent is worse than no memo, because it is persuasive.
- **A scheduler of everything.** It does not run the factory. It watches, flags and asks.

**Decision memo — required structure** (this is the contract that makes it useful rather than impressive):

```yaml
question:            "<the decision to be made>"
answer:              "<one paragraph, plain language>"
confidence:          high | medium | low   # with the reason for the rating
evidence:
  - {source_type: erp_record|document|message, ref: "...", what_it_shows: "..."}
  - {source_type: inference, ref: "agent:<name>@<version>", what_it_shows: "..."}
options:
  - {option: "A", upside: "...", downside: "...", cost_impact: "...", schedule_impact: "..."}
recommendation:      "<option + why>"
what_would_change_it: "<the fact that flips the recommendation>"
assumptions:         ["..."]
risks:               ["..."]
approvals_required:  ["..."]
```

### 1.2 Specialist agents — proposed roster

Ten domains, each decomposed into *narrow* agents with distinct output contracts. This is deliberately more agents than the ten business areas, because each narrow agent has a testable output. A "Sales Agent" that does everything is untestable; "Enquiry Intake" is testable.

| # | Domain | Agent | Input | Output contract (typed) | Tools (allow-list) | Initial autonomy |
|---|---|---|---|---|---|---|
| 1 | Sales | **Enquiry Intake** | Email/PDF/portal enquiry | `EnquiryExtract` (customer, part, qty, due, specs, attachments, confidence) | `enquiry.search`, `customer.lookup`, `document.parse` | L1 |
| 2 | Sales | **Comparable Finder** | `EnquiryExtract` | `ComparableSet` (similar jobs, why similar, actuals) | `jobcost.history`, `bom.get`, knowledge search | L1 |
| 3 | Sales | **Follow-up Chaser** | Open enquiry/pipeline list | `FollowupPlan` (who, when, channel, draft message) | `task.*`, `message.draft` | L2 (messages only after approval) |
| 4 | Engineering | **Drawing Reader** | Drawing PDF/DXF + part context | `DrawingFeatures` (material, thickness, bends, holes, tolerances, finish, confidence **per field**) | `document.parse`, `drawing.fetch_derived`, vision model (on-prem for C3/C4) | L1 |
| 5 | Engineering | **BOM Checker** | ERP BOM + drawing features | `BomDiscrepancy[]` (missing, extra, qty mismatch, unit mismatch) | `bom.get`, `bom.explode` (ERP-computed) | L1 |
| 6 | Costing | **Cost Preparer** | features + route + rates | `CostInputSet` → **deterministic engine** → `CostBreakdown` | `cost.rate_cards`, `cost.history`, costing engine | L1 |
| 7 | Costing | **Quote Drafter** | `CostBreakdown` + terms | `QuoteDraft` (narrative with numeric slots bound) | `template.render`, `terms.get` | L1 |
| 8 | Purchase | **Vendor Analyst** | Vendor history, RFQs | `VendorScorecard` (OTIF, quality, price trend, risk flags) | `vendor.performance`, `po.status` | L1 |
| 9 | Purchase | **PO Expediter** | Open POs vs need dates | `ExpediteList` + draft messages | `po.status`, `task.*` | L2 |
| 10 | PPC | **Delay Analyst** | Work orders, capacity, delays | `DelayDiagnosis` (root cause candidates with evidence) | `workorder.status`, `schedule.capacity`, `delay.reasons` | L1 |
| 11 | PPC | **Capacity Planner** | Order book + capacity | `CapacityGap` (bottleneck, weeks, options) | `schedule.capacity`, `workorder.status` | L0 |
| 12 | Quality | **NCR Analyst** | NCRs, inspection data | `QualityTrend` (Pareto, recurrence, likely causes) | `inspection.results`, `ncr.list` | L1 |
| 13 | Stores | **Shortage Analyst** | Stock, BOM demand, open POs | `ShortageForecast` (part, when, impact, mitigation) | `stock.on_hand`, `stock.movements`, `po.status` | L1 |
| 14 | Accounts | **Receivables Watcher** | Invoice/aging data | `CollectionPriority` (account, amount, days, action) | `receivable.aging` (masked), `invoice.status` | L1 |
| 15 | HR/Tasks | **Task Coordinator** | Decisions, actions, deadlines | `TaskSet` (owner, due, dependency, escalation rule) | `task.*` (our store), `employee.directory` | L2 |
| 16 | Management | **Report Builder** | Cross-domain queries | `Briefing` (weekly/monthly, all numbers sourced) | read tools + knowledge | L2 |

**Coverage note — what we deliberately left out of v1:** order acceptance, credit decisions, payment release, vendor selection with commitment, and anything that creates a contractual obligation. Those are human decisions with AI-prepared evidence, indefinitely, until a separate decision says otherwise.

### 1.3 The sheet-metal reality check (important, and often skipped)

Manufacturing AI pilots fail on arithmetic and geometry, not on language. Three specifics for a sheet-metal business:

1. **LLMs must not calculate.** Bend allowance, K-factor, developed length, material utilisation/nesting yield, scrap, heat-treatment and surface-finish costs, plating batch minimums, tooling amortisation, and GST are deterministic. Use formulas from your own rate cards and process sheets, implemented as a versioned engine. The model's job is to *populate the inputs from a drawing* and *explain the result to a human*.
2. **Drawing interpretation must be field-level confident.** A vision model reading a title block, material callout, thickness and tolerance can be right on four fields and wrong on the fifth. Return **per-field confidence**, and force human confirmation on any field whose value drives more than a defined share of cost (thickness and material usually do).
3. **Geometry needs real tools, not vision.** Flat-pattern extraction, hole counts and cut lengths come from DXF/STEP parsing and CAD kernels, not from a model looking at a PDF. A model looking at a raster drawing should be treated as a *reading aid with citations to the page region*, never as a measurement instrument.

Stated plainly: **a vision model cannot be trusted to read a GD&T frame or a bend note reliably today**, and pretending otherwise is how a quotation gets a 30% error that nobody catches until the job runs. The design above makes the model's uncertainty visible.

---

## 2. Orchestration and parallel execution (C)

### 2.1 Layer choice

| Layer | Role | Candidate technology | Why |
|---|---|---|---|
| **Task graph / agent runtime** | Express agent steps, state, branches, interrupts | LangGraph-style graph runtime (self-hosted) | **[FACT F-08]** durable checkpoints, `interrupt()` for human approval, time-travel replay |
| **Durable workflow** | Survive process death, long waits (days), timers, retries across services | Temporal (or equivalent) | **[FACT F-08]** checkpoints persist state, not execution; a dead process must be re-driven by something that owns the run |
| **Queue / bus** | Independent work fan-out, backpressure, event distribution | Kafka (or NATS/RabbitMQ) | Durable, replayable, ordered per key |
| **Scheduler** | Recurring jobs (nightly indexing, morning briefings, follow-up sweeps) | Cron/K8s CronJob or the workflow engine | Single, observable place for time-based work |
| **Policy / approval** | In-path decisions and human gates | OPA-style policy + approval service | Separates "what may happen" from "what the graph wants" |

Two viable shapes:

- **O-cheap:** LangGraph checkpoints + Postgres + a simple job runner. Good to prove the pattern; breaks down on multi-day waits and crash recovery.
- **O-durable (recommended):** LangGraph for agent logic, Temporal for execution durability, Kafka only where genuinely asynchronous events are needed. More moving parts, far fewer 3 a.m. mysteries.

### 2.2 Orchestration patterns to use, and the ones to avoid

| Pattern | Use it for | Caution |
|---|---|---|
| **Sequential chain** | Extraction → costing → drafting | Clear, cheap, debuggable; the default |
| **Parallel fan-out / fan-in** | Query five domains at once (stock, POs, capacity, quality, past jobs) then synthesise | Cap fan-out (e.g. 8); enforce a per-branch timeout; reconcile contradictions explicitly rather than averaging them |
| **Router / dispatcher** | Send a task to the right specialist | Router errors are silent and expensive — evaluate the router separately |
| **Supervisor loop** (CEO agent re-plans after each result) | Open-ended investigation ("why is margin falling?") | Bound iterations (e.g. 6); every loop costs money and drifts |
| **Map-reduce over documents** | "Summarise all 40 NCRs" | Chunk boundaries change answers; keep per-document citations |
| **Human-in-the-loop interrupt** | Any approval, any low confidence, any write | Must resume from persisted state, not a held-open process **[FACT F-08]** |
| **Compensation / saga** | Multi-step writes that can partially fail | Most ERP writes should not be "compensated"; better to make them one call or none |
| ❌ Free-form agent-to-agent chat | — | Injection propagation, unbounded cost, unauditable |
| ❌ Recursive self-spawning agents | — | Runaway cost and no clear owner of an outcome |
| ❌ "Let the CEO agent call all tools directly" | — | Defeats the allow-list and the audit boundary |

### 2.3 A concrete fan-out: "Why is order SO-4471 late?"

```
CEO Agent
 ├─▶ [parallel] Delay Analyst      → work order status, operation delays
 ├─▶ [parallel] Stores Analyst     → material shortage? which part, when
 ├─▶ [parallel] Vendor Analyst     → PO delays from suppliers
 ├─▶ [parallel] Quality Analyst    → rework / NCR hold
 └─▶ [parallel] Capacity Planner   → machine contention
        │  (join, 90 s budget, per-branch timeout 45 s)
        ▼
  Contradiction check: "Stores says no shortage; Delay says material wait"
        ▼
  Follow-up (sequential, capped): fetch the specific material issue records
        ▼
  Decision memo with 3 root-cause candidates, evidence, confidence, actions
        ▼
  Approval: L2 → "send expedite mail to vendor X" (human approves text + recipient)
```

Parallelism here is **bounded, typed and joined deterministically**. That is what makes it auditable. Unbounded parallel agents "collaborating" is not a design, it is a cost incident waiting for a reason.

### 2.4 Budgets, timeouts and idempotency (the unglamorous essentials)

| Control | Proposed default | Why |
|---|---|---|
| Max tool calls per task | 40 | Stops loops; measurable |
| Max wall-clock per task | 25 min interactive, 6 h batch | Prevents zombie runs |
| Max tokens per task | Per-task budget in the manifest | Cost predictability |
| Per-tool timeout | 10 s read, 30 s compute, 60 s report | Fast failure beats slow ambiguity |
| Max rows per read | 500 (configurable per tool) | Prevents full-table exfiltration through a "summary" |
| Retry policy | 2 retries, exponential backoff, only for transient classes | Avoids duplicate side effects |
| Idempotency | Client key on every write; dedupe store | Agent retries are normal, not exceptional |
| Concurrency limit per agent | 3–5 | Protects downstream ERP |
| Kill switch | Per-agent, per-task-type, global | Must be one click, tested monthly |

**[FACT F-08]** matters here: checkpointing resumes at super-step boundaries, so a node re-runs from its start. Any side effect inside a node must be idempotent or moved to a task that records completion.

### 2.5 Orchestration failure taxonomy (design for these, not for the happy path)

| Failure | Symptom | Designed response |
|---|---|---|
| Model returns plausible but wrong tool args | Tool rejects on schema; or wrong entity fetched | Validate against schema + entity existence check; require exact IDs, never names |
| Contradictory specialist outputs | Two branches disagree | Explicit contradiction node; surface both, never average |
| Silent tool degradation | Empty result treated as "no data" | Distinguish `not_found`, `unauthorized`, `unavailable`; `unavailable` never becomes a negative answer |
| Runaway loop | Rising cost, repeated calls | Hard call budget; loop detector on (tool, args) hashes |
| Interrupt lost | Approval never resumes | Approvals are records with owners and SLAs; expired = escalate, never auto-approve |
| Partial write | ERP changed but task failed | One command per business action; reconciliation job |

---

## 3. Shared company knowledge and controlled learning (D)

### 3.1 Four distinct stores (do not merge them)

The most common design failure is a single "vector database of everything", which destroys access control and makes learning unauditable. Keep four stores with different lifecycles:

| Store | Contents | Mutable? | Access control | Retention |
|---|---|---|---|---|
| **Document index** | Chunks + embeddings + metadata of drawings, specs, procedures, contracts, emails | Rebuilt on source change | Inherited ACL per chunk, enforced at query | Mirrors source policy |
| **Structured knowledge** | Rate cards, process routings, machine capabilities, material specs, customer terms | Versioned, approved | Per-domain roles | Permanent, versioned |
| **Decision ledger** | Every recommendation, approval, rejection, human edit, outcome | Append-only | Read: role-scoped; write: system only | Long (see `03` §3) |
| **Working memory** | Per-task scratch, conversation state | Ephemeral | Task-scoped | 30–90 days |

The distinction matters because "learning" means different things in each:
- Document index: refreshed, not learned.
- Structured knowledge: **edited by humans**, with approval.
- Decision ledger: **mined for precedents**, never auto-promoted to rules.
- Working memory: discarded.

### 3.2 Retrieval design

| Decision | Proposal | Reason |
|---|---|---|
| Chunking | Structure-aware (headings, table rows, drawing regions), 300–800 tokens with overlap; keep parent section reference | Fixed-size chunking destroys table and drawing context |
| Index | **Hybrid**: BM25/lexical + dense vectors, fused by reciprocal rank **[FACT F-12]** | Part numbers, drawing numbers, vendor codes and tolerances are exact-match problems; semantics alone misses them |
| Reranking | Cross-encoder rerank top 30 → top 6 | Cheap accuracy gain |
| Access control | **ACL filter inside both the lexical and vector query, before fusion and reranking** **[FACT F-12]** | Post-filtering leaks via counts, snippets, timing; and misses permitted content |
| Versioning | Every chunk carries `source_version`, `ingested_at`, `classification` | Answers must be reproducible as of a date |
| Citations | Chunk ID + page/region returned to the model *and* to the UI | Enables verification; the memo contract requires it |
| Fail-closed | Identity/ACL service unreachable → return nothing | Safety over availability for C3/C4 |
| Freshness | Event-driven reindex (from facade events) + nightly reconciliation | Stale answers about stock or price are actively harmful |

Retrieval is where leak risk concentrates. Two rules worth stating as policy:

- **The prompt is not a security boundary.** "Do not reveal documents about X" in a system prompt is a suggestion to a stochastic system. Only the query filter is a boundary.
- **Summaries, snippets, caches and conversation history inherit the ACL.** A cached answer built from C4 sources is C4, even if it "feels" generic.

### 3.3 The learning loop, stated precisely

Your stated loop: **Observe → Analyse → Recommend → Human Approval → Record → Validate → Reuse.** Mapped to mechanism:

| Stage | Mechanism | Artefact | Owner |
|---|---|---|---|
| Observe | Facade reads, document ingestion, task outcomes, human actions | Raw events | System |
| Analyse | Agent reasoning over retrieved context + comparables | Draft recommendation with citations | Agent |
| Recommend | Decision memo / draft output | Memo, draft quote, proposed action | Agent |
| **Human approval** | Approval service records decision + edits + reason code | `ApprovalRecord` | Named human |
| Record | Decision ledger entry: inputs, options, output, edits, approver, timestamp | `DecisionCase` | System (immutable) |
| Validate | Outcome observed later (won/lost, actual cost, NCR closed, delay resolved) | `OutcomeRecord` linked to case | System + human confirmation |
| Reuse | Case becomes retrievable precedent, **scoped and weighted**, never a global rule | `Precedent` with scope tags | System, with human-visible scope |

### 3.4 Anti-over-generalisation: the "precedent, not rule" doctrine

This is the part of the brief that is easiest to say and hardest to implement. Five concrete controls:

1. **Scope tags are mandatory.** Every learned precedent carries: customer, programme, part family, material, process route, machine, time window, and the specific condition that made the decision right. A precedent without scope is not retrievable.
2. **Precedents are evidence, not instructions.** They appear in the prompt as *"similar past case, approved by X, outcome Y"* — never as *"always do Z"*. The model is instructed to weigh and cite them, and to state when it is deviating.
3. **Promotion is a separate, rare act.** A precedent can be promoted to a *rule of thumb* only after: N≥5 similar cases, consistent outcomes, a named reviewer, and a written scope. Promotions are versioned entries in structured knowledge with an expiry date and a review date.
4. **Contradiction is preserved.** When a later approval contradicts an earlier precedent, both remain; the ledger records the conflict and the reason. Do not "update" history to look consistent — that destroys the signal that the business has changed.
5. **No automatic fine-tuning on approvals.** Training a model on its own approved outputs amplifies existing bias and makes behaviour non-reproducible. If fine-tuning is ever used, it is on a curated, versioned dataset with a held-out evaluation, and it produces a new model version with a rollback. (And: fine-tuning is not needed to start — retrieval of precedents gives most of the benefit at a fraction of the governance cost.)

### 3.5 Evaluation: how you know a capability has earned autonomy

Autonomy should be an evidence-based promotion, not a feeling. The measurement kit:

| Metric | Definition | Use |
|---|---|---|
| **Field accuracy** | % of extracted fields matching human-confirmed values, per field | Drives per-field confidence thresholds |
| **Numeric error** | Absolute % deviation of computed cost vs approved cost | Gate for costing capability |
| **Recommendation acceptance** | % of memos where the human chose the AI's recommendation (or a variant) | Value signal, not a quality signal on its own |
| **Correction rate** | Edits per draft, by type | Where to improve; a rising rate is a regression alarm |
| **Abstention quality** | Of cases the AI flagged low-confidence, how many were actually wrong | Trust calibration — the most under-measured metric |
| **Harmful-output rate** | Schema violations, policy violations, injected content acted on | Must be zero for promotion |
| **Cost per task / latency** | Tokens, rupees, wall-clock | Operational viability |
| **Drift** | Change in the above over time and after model/data changes | Triggers re-evaluation |

Golden sets: build 100–300 real historical cases per capability, with human-verified answers, held by the business (not the vendor). Re-run on every prompt, model or retrieval change. **[FACT F-02]** NIST's MEASURE function is the right vocabulary for this; ISO/IEC 42001 **[FACT F-05]** supplies the management-system wrapper.

**The uncomfortable but necessary test:** before promoting any capability to L2, run a *blinded* comparison where a human does the same task independently, and compare errors by type and cost. If the AI's errors are cheaper to catch than the human's, promote. If not, don't.

### 3.6 What must never be learned

| Never | Why |
|---|---|
| Customer-specific price concessions as a general pricing rule | Commercially catastrophic if applied to another customer |
| A single engineer's drawing interpretation as a standard | One-off judgement, not policy |
| Deviations approved under time pressure | Encodes the exception as the norm |
| Anything marked C4 into a shared index | Cross-programme leak |
| Personal data about employees into a retrieval corpus | **[FACT F-03]** DPDP purpose limitation; also an insider-risk problem |
| Corrections made without a reason code | Unusable as a signal, unlearnable in a safe way |

---

## 4. Voice, employee communication and task follow-ups (H)

### 4.1 Why voice matters here, and why it is the riskiest channel

Factory and shop-floor reality: supervisors will not type a structured form. Voice notes are how work actually gets reported. But voice is also the channel with the worst accuracy, the least structure, and the highest chance of capturing confidential content in a semi-public place.

**Design position:** voice is an **input capture and confirmation** channel, not a command channel, and not an autonomous-action channel.

### 4.2 Speech stack (on-prem capable)

| Function | Options | Notes |
|---|---|---|
| **ASR (speech→text)** | AI4Bharat IndicWhisper / IndicConformer (600M, 22 languages, MIT licence), Whisper large-v3 variants, Vaani fine-tunes | **[FACT F-16]** publicly released, runnable on-prem. Expect WER well above 20% on noisy telephone/workshop audio — plan for confirmation, not transcription certainty |
| **Diarisation / VAD** | Standard open tooling | Needed for "who said what" in a huddle recording — treat with caution |
| **TTS (text→speech)** | Piper-class local TTS, or a commercial India-region service for non-restricted content | Local TTS avoids sending text out; quality is adequate for short confirmations |
| **Language** | Hindi/Marathi/English code-switching is the norm | Test on your own recordings early; generic benchmarks overstate accuracy |
| **Whisper hallucination risk** | Known failure mode on silence/noise | Reject transcripts with low confidence or implausible length; never act on a transcript that fails a sanity gate |

**Practical implication:** design every voice flow around **"heard → shown → confirmed"**. The user sees the extracted intent and entities, corrects them by tapping, then confirms. This converts an unreliable channel into a reliable one, and it produces labelled data for improving the ASR.

### 4.3 Communication channels — and the one we recommend against

| Channel | Use | Constraint |
|---|---|---|
| **In-app task inbox** (web/mobile PWA) | Primary: assignments, confirmations, approvals, voice notes | Best control; requires adoption effort |
| **Self-hosted team chat** (e.g. Matrix/Element, Mattermost, on-prem) | Secondary: notifications, quick confirmations, group updates | Keeps content inside Zone 1; auditable |
| **Email** | Outbound drafts, digests, external follow-ups | Draft-only for external parties; approval required |
| **SMS / IVR** | Deadline reminders for staff without smartphones | Content must be minimal (no confidential detail) |
| **Public consumer messaging apps (WhatsApp etc.)** | **Not recommended for anything above C2** | Content leaves your control; no reliable audit; account compromise = data breach; business API terms change; **[FACT F-03]** fiduciary obligations follow the data |
| **Always-on room microphones** | Not recommended initially | Continuous capture of confidential speech is a large, hard-to-govern liability |

That last row is a genuine recommendation, not caution for its own sake: an always-listening device in a quotation or programme discussion room is a recording of your most sensitive commercial conversations, held by a system whose access control you are still building. Start with push-to-talk.

### 4.4 Task follow-up engine (the most immediately useful agent, and the least risky)

A follow-up system is where AI earns trust fastest, because the value is obvious and the failure is cheap ("the reminder was wrong" vs "the PO was wrong").

**The task object** (owned by us, not the ERP):

```yaml
task_id, title, description
origin:            {type: approval|decision|erp_event|manual|agent, ref: "..."}
owner:             employee_id            # human, never an agent
accountable:       employee_id            # one person, always
due_at, created_at, priority
dependencies:      [task_id]
evidence:          [links to records/documents]
escalation_rule:   {after_hours: 24, to: manager|ceo, max_levels: 3}
status:            open|blocked|done|cancelled|expired
closure_evidence:  required? (e.g. "photo of inspection report")
```

**Follow-up behaviour — the rules that keep it from becoming noise:**

1. **One owner per task.** Shared ownership is how tasks die.
2. **Escalation is a ladder with limits** (e.g. reminder → owner + manager at 24 h → CEO at 72 h, then stop). Escalating forever trains people to ignore the system.
3. **Reminders carry the evidence** ("PO 4412 to Precision Tools, needed for SO-4471 on 14 Dec, no GRN yet") — never "you have a pending task".
4. **Closure requires evidence where it matters.** "Done" with no artefact is a data quality problem, not a completion.
5. **Quiet hours and one digest per day per person.** Notification fatigue is the number one cause of adoption failure in these systems.
6. **The AI never assigns accountability to itself.** Agents are *actors*, humans are *accountable*. This distinction is what makes an audit trail meaningful.
7. **The task engine is not the ERP.** ERP remains the record for the underlying business object; the task is a coordination artefact linked to it.

**What "AI" adds over a normal task tracker:** it *creates* tasks from unstructured input (a meeting note, a voice message, an email, a detected delay), it *drafts* the message, it *detects* blockers by cross-referencing ERP state, and it *summarises* what is stuck and why. It does not need autonomy to be valuable here — and it should not have it.

### 4.5 Meeting and call handling

| Use | Approach | Guardrail |
|---|---|---|
| Internal review meeting notes | Record (with consent), transcribe, extract decisions/actions, human approves the action list | Consent recorded; retention short; C3/C4 meetings on-prem only |
| Customer call summarisation | Only with disclosure and consent; or manual notes | Recording laws and contract terms vary; legal review needed (Q-13) |
| Shop-floor voice notes | Push-to-talk, 30–60 s, structured extraction, confirm on screen | No always-on capture |
| Executive voice queries | Read-only answers, spoken summary of a memo | Never executes actions by voice alone |

---

## 5. Open questions raised by this section

| ID | Question |
|---|---|
| Q-13 | What consent and recording practices are lawful and acceptable for internal/customer calls, and do customer contracts prohibit recording? (Legal) |
| Q-14 | Who are the named accountable humans per domain, and what are the delegation limits (amounts, quantities, dates)? |
| Q-15 | Which single capability will be the first to be evaluated, and who owns its golden set of historical cases? |
| Q-16 | What language mix do shop-floor and office staff actually use in voice notes, and can we obtain 20–50 sample recordings for ASR evaluation? |
| Q-17 | Is there an existing task/action tracker that must be the task store of record, rather than building one? |
| Q-18 | Which three reports does the CEO actually read today, in what format, and at what cadence? (The first Report Builder output should replace one of them exactly.) |
