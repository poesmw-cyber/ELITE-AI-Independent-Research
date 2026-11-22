# 03 — Security, Identity, Audit, Approval and Recovery

Covers assignment sections **F** (authentication, authorization, secure agent identity) and **I** (monitoring, audit trails, approvals, recovery), plus the threat model that justifies them.

---

## 1. The honest security position (read this before the controls)

**No architecture is 100% secure.** Anyone who says otherwise is selling something. What a good design does is change the economics and the consequences:

| Property | What it means concretely |
|---|---|
| **Costly to attack** | No single credential opens everything; no standing wildcard; no unauthenticated inference endpoint |
| **Detectable** | Every consequential action produces a correlated, tamper-evident record; anomalies alert |
| **Containable** | Compromise of one agent, one document or one user is bounded by data class, tool allow-list and network zone |
| **Recoverable** | Known-good state, tested restore, model and prompt rollback, and a kill switch that works |
| **Explainable afterwards** | You can reconstruct who/what/why for any decision, months later |

Two threat realities specific to this programme:

1. **The model is an untrusted component.** Treat its output exactly as you would input from the internet: parse it, validate it, constrain what it can cause. **[FACT F-01]** prompt injection (direct and indirect) is the top LLM risk precisely because it cannot be fully "fixed" in the model — only contained by architecture.
2. **Your documents are the attack surface.** A drawing note, a vendor PDF, an email signature, a spreadsheet cell, an OCR'd scanned page can carry instructions. The moment an agent reads external content and has tools, indirect injection becomes a live path. This is the single most under-estimated risk in enterprise agent projects.

---

## 2. Authentication, authorization and agent identity (F)

### 2.1 Three distinct identity problems

Most designs conflate these and then cannot answer "who did this?" in an audit.

| Identity | Question it answers | Mechanism |
|---|---|---|
| **Human identity** | Which person is present, on which device? | Corporate IdP, OIDC/SAML, MFA, device posture, session binding |
| **Workload identity** | Which service/agent process is this, cryptographically? | SPIFFE/SPIRE SVIDs or mTLS with short-lived certs **[FACT F-10]** |
| **Delegated authority** | On whose behalf, with what scope, for how long? | OAuth 2.1 + RFC 8693 token exchange with `act` claim; DPoP (RFC 9449) sender-constrained tokens **[FACT F-10]** |

The audit record must contain **all three**: user, agent, and the scope it acted under. A log line with only a service account name is useless for accountability.

### 2.2 Human authentication

| Control | Proposal |
|---|---|
| Primary auth | Corporate IdP (existing AD/Entra/Google/LDAP if present — see Q-07); OIDC; MFA mandatory for anything above C2 |
| Session | Short-lived access tokens (≤60 min), refresh with rotation, absolute and idle timeouts |
| Device | Managed devices for C3/C4 access; unmanaged devices restricted to read-only C1/C2 summaries |
| Step-up | Re-authentication (or hardware key) for: approving payments, exporting C4, changing autonomy levels, changing policy |
| Privileged access | Separate admin accounts, no admin rights in the AI layer from ordinary sessions, break-glass account with loud alerting |
| Departures | Immediate revocation path, tested; offboarding checklist includes agent tokens issued on behalf of that user |

### 2.3 Authorization: policy as code, decisions in the path

Authorization is a **policy decision** (`may P do X on R given C`), evaluated by a dedicated engine, not scattered `if` statements in agent prompts.

```
Request → authenticate (human + workload)
        → resolve entitlements (groups, programme access, clearance, delegation)
        → policy engine evaluates: principal, action, resource, data class, context
        → allow | deny | allow-with-obligations (e.g. "requires approval", "mask field F")
        → token minted: audience-bound, scope-limited, TTL ≤ task duration
        → tool gateway executes; audit record written BEFORE the effect
```

Rules that matter:

- **Deny by default.** A new tool or a new agent has no access until explicitly granted in its manifest.
- **No client-supplied identity or classification.** The user ID and data class come from the session and from the record, never from the request body or the prompt.
- **Obligations, not just allow/deny.** "Allowed but requires a second approver over ₹X" must be expressible in policy, not in a prompt instruction.
- **Policy is versioned and reviewed.** A policy change is a change-controlled event with an audit entry.
- **Fail closed.** Policy engine unavailable → deny. Latency budget must accommodate this (see §5.4).

### 2.4 Agent identity and least authority — the design

Each agent gets:

| Element | Design |
|---|---|
| **A distinct identity** | Its own SPIFFE ID / workload identity; no shared "ai-service" account across agents |
| **A manifest** (versioned, signed) | role, prompt version, tool allow-list, data-class ceiling, autonomy level, budgets, owner, eval suite |
| **No long-lived secrets** | Credentials minted per task; nothing in environment variables or code |
| **Delegated, down-scoped tokens** | RFC 8693 exchange from the *user's* session; scope = exactly the tools of this task; TTL minutes **[FACT F-10]** |
| **Actor attribution** | Token carries `sub` = user, `act` = agent, so downstream services log both |
| **Sender-constrained tokens** | DPoP so a stolen token is not replayable **[FACT F-10]** |
| **No token passthrough** | The gateway never forwards a caller's token to another service **[FACT F-09]** |
| **A kill switch** | Per-agent disable that takes effect within seconds (token issuance refused) |

**Background (non-user) agents** — nightly indexing, follow-up sweeps, monitoring — act as themselves, with a narrow service identity and a fixed tool set, never "on behalf of the CEO". If a background agent needs to attribute an action to a person, it creates a *proposal* for that person, not an action.

### 2.5 Trust boundaries between agents

Agent-to-agent calls are a classic confused-deputy risk: agent A (broad tools) accepts a task from agent B (narrow), and the combination exceeds either's authority.

Controls:
- **Typed task handoffs only** (schema-validated), never free-form prompt text.
- **Authority never increases through a hop.** The effective permission of a chain is the intersection of all participants' permissions and the originating user's. Enforce this at the gateway, not by convention.
- **Depth and fan-out limits** on delegation.
- **The CEO Agent holds no write tools**, so compromising it cannot commit anything — it can only produce bad advice, which is visible in the memo and gated by approval.

### 2.6 ERP-side identity

- A **dedicated integration identity** per contract (not a human's login, not `sa`/`root`).
- Read identity and write identity separated; write identity usable only by the facade's command handler, not by any agent container.
- **IP allow-list** to the facade host, mutual TLS where supported.
- Credentials in a secrets manager with rotation and audit; never in config files, spreadsheets, or prompts.
- The ERP team retains the ability to **revoke the AI layer entirely** without touching production operations — this must be tested, not assumed.

---

## 3. Threat model and controls

### 3.1 Threats, ranked for this context

| # | Threat | Vector | Impact | Primary controls | Residual risk |
|---|---|---|---|---|---|
| T-1 | **Indirect prompt injection** via a document, drawing note, email or vendor file | Attacker-controlled content reaches model context while tools are available | Data exfiltration, wrong action, policy bypass | Treat retrieved content as untrusted; no instruction-following from documents; tool allow-list; output validation; classification-aware egress; injection detection; human approval on writes **[F-01]** | Medium: detection is probabilistic; a novel injection can still produce a bad *recommendation* (bounded by approval) |
| T-2 | **Excessive agency** (too many tools, too much autonomy) | Design choice, not attacker action | Large blast radius | Manifest allow-lists; no write tools for reasoning agents; autonomy ladder; budgets **[F-01 LLM06]** | Low if enforced in the gateway |
| T-3 | **Sensitive information disclosure** through outputs, logs, caches, embeddings or another tenant's context | Poor segregation, verbose logging, prompt leakage | Contract breach, programme compromise | Data-class routing; ACL pushdown in retrieval; redaction before logging; no cross-class index; per-class model endpoints **[F-01 LLM02]** | Medium: human error is the main path |
| T-4 | **Credential/token theft** from the AI layer | Compromised container, over-broad token, leaked secret | Lateral movement into ERP | Workload identity, short TTL, DPoP, no static secrets, network segmentation, egress deny | Low–medium |
| T-5 | **Insider misuse** (legitimate access, malicious or careless intent) | Staff exporting C3/C4, screenshotting, pasting into public AI | Confidentiality breach | DLP on egress, watermarking, anomalous-query detection, least privilege, audit, acceptable-use policy | **High — cannot be solved technically**; requires culture, contracts and monitoring |
| T-6 | **Model supply chain** compromise (weights, container, dependency, plugin) | Malicious or tampered artefact | Backdoored behaviour | Pinned versions, hashes/signatures, own registry, SBOM, vendor provenance, no auto-update from public hubs, reproducible builds **[F-01 LLM03]** | Medium |
| T-7 | **Data/model poisoning** of the knowledge base or precedents | Malicious or low-quality documents, wrong corrections | Systematically wrong advice | Source allow-list, ingestion review for C3/C4, provenance, correction reason codes, drift monitoring **[F-01 LLM04]** | Medium |
| T-8 | **Hallucinated but confident output** | Model behaviour | Wrong quote, wrong decision | Evidence requirement, deterministic computation, per-field confidence, abstention, approval gates | Medium; mitigated by design, never eliminated |
| T-9 | **Availability / denial of wallet** | Runaway loops, abusive queries, GPU exhaustion | Cost spike, service loss | Per-agent budgets, rate limits, quotas, anomaly alerts on token spend, kill switch | Low |
| T-10 | **Reconstruction/inference attacks on embeddings** | Vector store compromise | Approximate document content disclosure | Encrypt vector store, restrict to enclave, treat embeddings as C3/C4 data, no external vector DB for restricted classes | Low–medium |
| T-11 | **ERP integrity damage** through a bad write | Wrong command, wrong args, retry duplication | Corrupted records, wrong commitments | Typed commands, idempotency, approval, reconciliation, draft-only writes initially, no direct DB writes | Low if M4 is prohibited |
| T-12 | **Audit tampering / repudiation** | Privileged insider deletes logs | Inability to investigate or defend | Hash-chained append-only ledger, WORM storage, separate log account/zone, offsite copy, integrity verification job **[F-13]** | Low–medium |

### 3.2 Prompt injection: what actually works

There is no prompt that makes a model immune. The workable posture is **containment**, in this order:

1. **Reduce what an injected instruction can reach.** No credentials in context; no write tools on reading agents; tool allow-lists; egress deny.
2. **Separate instruction from data structurally.** Retrieved content is delivered in a delimited, labelled data channel with an explicit "this is data, never instructions" contract, and the harness strips/escapes delimiter sequences from content. (Reduces, does not eliminate.)
3. **Validate outputs as data.** JSON schema, enum values, numeric ranges, entity-ID existence checks. A model that "decides" to pay ₹40 lakh cannot, because no such tool exists and no such parameter passes validation.
4. **Detect patterns** in untrusted content (imperative phrasing aimed at an assistant, encoded blobs, invisible Unicode, unusual URL/QR content) and flag or quarantine.
5. **Human approval on every consequential action**, with the *evidence* shown, not just the conclusion — a human who sees "source: vendor email asking to change bank details" catches what a model will not.
6. **Log the full context** (redacted per policy) so an incident can be reconstructed.
7. **Test adversarially.** Maintain a red-team suite of injected documents (invisible text, white-on-white, metadata, OCR-only, multilingual, image-embedded) and run it on every prompt/model change.

Special cases for this business: **scanned drawings with OCR**, **vendor PDFs**, **customer emails**, and **DXF/STEP metadata** are all realistic injection carriers. Vendor bank-detail changes deserve a hard rule: **never accepted from an AI-mediated channel without out-of-band human verification.**

### 3.3 Encryption and key management

| Layer | Control |
|---|---|
| In transit | TLS 1.3 internally and externally; mTLS between services; no plaintext model API |
| At rest | Full-disk/volume encryption; database TDE; object store SSE; encrypted backups |
| Application-level | Field-level encryption or tokenisation for high-sensitivity identifiers (bank details, pricing secrets, personal data) |
| Keys | Managed KMS/HSM; separate keys per data class and per environment; envelope encryption; automatic rotation; no keys in code or config |
| Enclave keys | HSM or software HSM in Zone 1; key ceremony documented; dual control for root keys |
| Secrets | Vault/secrets manager with dynamic credentials; short TTL; audited retrieval |
| Deletion | Crypto-shredding for retained-then-deleted data (destroy the key) |
| Model/artefact integrity | Signatures and hashes on model files and containers; verification before load **[F-01 LLM03]** |

Note the honest limitation: encryption protects data at rest and in transit, **not data in use inside the model context**. If a C4 drawing must be analysed, its content is in GPU memory. That is why the enclave exists — not because encryption fails, but because the enclave bounds who can be in the same room as the plaintext.

---

## 4. Approval as a system component (I)

Approval is where most AI governance is claimed and not implemented. It must be a durable record with an owner, an SLA, an evidence bundle and a reason code — not a chat message saying "looks good".

### 4.1 The approval object

```yaml
approval_id, created_at, expires_at
action:            {tool: "quotation.create_draft", args_hash: "...", preview: {...}}
requested_by:      {agent: "quote-drafter@2.1", user: "u_1042"}
required_approvers: [{role: "estimator", quorum: 1}, {role: "commercial_head", quorum: 1, condition: "value > 25L"}]
evidence_bundle:   [memo, cost_breakdown, comparables, source_documents, confidence]
risk_flags:        [low_confidence_field: "thickness", novel_part: true, injection_suspected: false]
decision:          approved | rejected | changes_requested | expired
reason_code:       required enum  # e.g. price_strategy, technical_doubt, customer_terms
human_edits:       diff of what the human changed
signature:         {user, timestamp, device, auth_strength}
```

### 4.2 Rules

1. **Approval is bound to the exact payload** (`args_hash`). If the agent re-generates the draft, the approval is void and a new one is required. This prevents "approve then swap".
2. **Evidence is shown, not summarised away.** The approver sees sources and confidence, not just a conclusion.
3. **Quorum and conditions come from policy**, not from the agent's guess.
4. **Expiry escalates, never auto-approves.**
5. **Reason codes are mandatory** on rejection or edit — they are the highest-value learning signal you will get.
6. **Four-eyes for irreversible or high-value actions**; single approval acceptable for reversible drafts.
7. **Approval fatigue is a security bug.** If approvers rubber-stamp, the control is theatre. Measure approval latency and edit rates; if median review time is under ~10 seconds, the gate is not working and should be re-scoped.

---

## 5. Monitoring, audit and observability (I)

### 5.1 Three log classes, different purposes and retention

| Class | Contents | Storage | Retention |
|---|---|---|---|
| **Audit ledger** | Consequential events: approvals, writes, policy changes, autonomy changes, exports, logins to sensitive classes, key operations | Append-only, hash-chained, WORM copy **[FACT F-13]** | Long — propose 7 years for approvals/decisions, subject to legal review; minimum 180 days for ICT logs **[FACT F-04]** |
| **Operational telemetry** | Traces (agent, tool, model spans), latency, token usage, errors, queue depth | OTel collector → metrics/trace store (Zone 1) **[FACT F-14]** | 30–90 days hot, 1 year cold |
| **Content logs** | Prompts, retrieved chunks, outputs | Encrypted, access-restricted, redacted per policy | Short by default (30–90 days), longer only for incidents/investigations |

The distinction matters: you cannot put full prompts in a 7-year WORM archive (volume and confidentiality), and you cannot investigate an incident with only metrics. Separate stores, separate access, explicit linkage by trace/decision ID.

### 5.2 What every audit record must contain

`event_id` · `timestamp` (NTP-synchronised **[FACT F-04]**) · `actor_user` · `actor_agent` (name@version) · `delegated_scope` · `action` · `resource` · `data_class` · `policy_decision` (allow/deny + rule id) · `approval_id` (if any) · `outcome` · `args_hash` · `response_hash` · `prev_hash` · `row_hash`

Chain rule: `row_hash = SHA-256(prev_hash ‖ canonical(fields))`, with a periodic signed checkpoint, and an independent verification job. **[FACT F-13]** WORM alone proves non-overwrite; the chain proves non-alteration; the external checkpoint proves completeness to a third party.

### 5.3 Detection content (the alerts that actually matter)

| Alert | Signal | Response |
|---|---|---|
| Injection suspected | Guard model / pattern hit on retrieved content | Quarantine document, notify owner, block that document's use until reviewed |
| Unusual data-class routing | C4 content attempted to a non-C4 endpoint | Block (fail closed), page security owner |
| Mass read | One agent/user reading far above baseline rows or entities | Throttle, require justification, review |
| Approval anomaly | Approval in <5 s, or same approver for 100% of a capability | Governance review; possible rubber-stamping |
| Cost anomaly | Token spend per hour above baseline by a factor | Budget kill, investigate loop |
| Ledger integrity failure | Hash chain break | **Do not repair.** Preserve, escalate, treat as incident |
| ERP/derived divergence | Reconciliation mismatch | Block dependent automation; open data-quality task |
| Model/prompt change | Deviation from signed manifest | Block start until re-approved + re-evaluated |
| Tool error surge | Rising `unavailable` | Distinguish outage from "no data"; never let it become a false negative |

### 5.4 SLOs (proposed starting targets)

| Metric | Target | Note |
|---|---|---|
| Interactive p95 time-to-first-token | ≤ 2.5 s on-prem, ≤ 1.5 s cloud | Below this, users abandon the tool |
| Memo generation p95 | ≤ 90 s | Bound fan-out accordingly |
| Approval path availability | 99.5% monthly | If approvals are down, writes stop — acceptable, unlike silent bypass |
| Policy decision latency | p99 ≤ 25 ms | Must not become the bottleneck that tempts bypass |
| Audit write success | 100% (fail closed on audit failure) | No audit, no action |
| Ingestion freshness | ≤ 15 min from ERP event | Stale stock/price answers are dangerous |
| Eval suite pass rate | 100% of gates before deploy | A failing gate blocks deployment |

Availability philosophy: **the AI layer may fail; the ERP must not.** Every dependency is one-way. Degraded mode = AI off, humans work as they do today.

---

## 6. Recovery and continuity

### 6.1 Targets **[ESTIMATE — to be ratified]**

| Scenario | RPO | RTO | Mechanism |
|---|---|---|---|
| AI layer down | 0 (stateless) | 4 h | Redeploy from signed artefacts; ERP unaffected |
| Vector/knowledge store lost | 0 for sources; ≤24 h for derived | 12–24 h | Rebuild from ERP + document store (derived data is disposable by design) |
| Decision ledger lost | 0 | 4 h | WORM copy + offsite; restore and verify chain |
| GPU node failure | n/a | 1–3 days (hardware) | Fallback to smaller model / cloud for C1–C2; C3–C4 queue until repaired |
| Model regression discovered | n/a | 15 min | Pin previous model/prompt version; rollback is a config change, not a deploy |
| Ransomware in Zone 2 | ≤24 h | 1–3 days | Immutable backups, network isolation of Zone 1, tested restore |
| Suspected compromise of enclave | 0 | 1–5 days | Isolate, rotate all keys, rebuild from signed images, forensic copy |

### 6.2 Recovery practices that are usually skipped

- **Quarterly restore drills** with a written result, including a chain verification of the restored ledger.
- **Model/prompt rollback rehearsed** — if rollback takes a day, you will not roll back when you should.
- **Kill-switch test monthly**, timed, with the result recorded. A kill switch that has never been pulled is a hope.
- **Revocation test**: revoke one user and one agent; confirm token issuance stops within the target window.
- **"Delete the AI layer" test**: confirm ERP operations continue unaffected — this is the strongest proof that the architecture is additive, not entangled.
- **Tabletop exercise** on an indirect-injection incident: who is paged, who decides to shut down, who notifies the customer, who notifies CERT-In within 6 hours **[FACT F-04]**.

### 6.3 Incident response and the Indian reporting clock

**[FACT F-04]** requires reporting listed cyber incidents to CERT-In within **6 hours**, with 180-day log retention in Indian jurisdiction and NTP-synchronised clocks. **[FACT F-03]** adds breach intimation to the Data Protection Board and affected individuals once the DPDP obligations commence (13 May 2027), with a 72-hour detailed report under the Rules.

Design implications, stated plainly:

- A 6-hour clock means **you cannot wait for the AI vendor's support ticket**. Detection and first assessment must be in-house.
- Cloud providers with 72-hour-notification terms **cannot** satisfy a 6-hour reporting obligation alone; contractual notice must be tighter, or the architecture must make provider-side incidents non-material to your reporting duty.
- Logs that live only in the model provider's console are not your logs. Retain your own copies in Indian jurisdiction, and make sure the retention is real (not "up to 30 days").
- Legal counsel must confirm whether and how these apply to your entity and to AI-specific incidents; this report is not a legal opinion.

---

## 7. Security controls checklist (condensed, auditable)

| Area | Control | Status to be verified |
|---|---|---|
| Identity | MFA everywhere above C2; SSO; no shared accounts | ☐ |
| Workload identity | Per-agent cryptographic identity; no static secrets | ☐ |
| Authorization | Policy engine in path; deny by default; obligations supported | ☐ |
| Tokens | Audience-bound, short TTL, DPoP, no passthrough | ☐ |
| Data class | Inherited, server-computed, enforced in routing and retrieval | ☐ |
| Retrieval | ACL pushdown; fail closed; no cross-class index | ☐ |
| Prompt injection | Untrusted-content handling, output validation, red-team suite | ☐ |
| Tools | Allow-lists per agent; schema validation; budgets; no raw SQL | ☐ |
| Writes | Typed commands, idempotent, approval-gated, draft-first | ☐ |
| Encryption | Transit, rest, field-level, KMS/HSM, rotation, crypto-shred | ☐ |
| Audit | Hash chain, WORM copy, verification job, 180-day minimum | ☐ |
| Monitoring | SIEM ingest, alert content, on-call owner | ☐ |
| Approval | Bound to payload, quorum, expiry, reason codes, fatigue metric | ☐ |
| Autonomy | Registry with owner, evidence, expiry, review date | ☐ |
| Supply chain | Pinned/signed models and images, SBOM, own registry | ☐ |
| Recovery | Restore drill, rollback drill, kill-switch drill, ledger verification | ☐ |
| Governance | AI policy, risk register, ISO 42001-aligned review cadence | ☐ |
| People | Acceptable-use policy, training, insider-risk monitoring | ☐ |
| Legal | DPDP, CERT-In, contract data clauses reviewed by counsel | ☐ |

Nothing on this list is a one-time task. Each item needs an owner, a test, and a review date — otherwise it decays into a paragraph in a document like this one.
