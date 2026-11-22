# ELITE AI – Technical Blueprint Proposal
## AI-Powered Company Operating System for Indian Defence & Industrial Sheet-Metal Manufacturing

**Document Version:** 1.0  
**Date:** 2026-11-22  
**Classification:** CONFIDENTIAL – Internal Architecture Research  
**Status:** DRAFT – For CEO Review & Approval

---

## Executive Summary

This document presents an independent architecture research report for **ELITE AI** – a private AI-powered company operating system for an Indian defence and sheet-metal manufacturing company. The system envisions a **Digital CEO Agent** coordinating specialist AI agents across the full manufacturing value chain: Sales, Engineering, BOM, Costing, Procurement, PPC, Quality, Stores, Accounts, and Employee Management.

**Core Philosophy:** Observe → Analyse → Recommend → Human Approval → Record → Validate → Reuse. Autonomy granted gradually based on demonstrated reliability.

**Key Constraint:** No access to existing ERP (ELITE AQMS-ERP) source code. All integration recommendations are architecture-level patterns requiring technical discovery before implementation.

---

## Table of Contents

1. [Recommended Overall Architecture](#1-recommended-overall-architecture)
2. [Central CEO Agent & Specialist Agent Design](#2-central-ceo-agent--specialist-agent-design)
3. [Agent Orchestration & Parallel Task Execution](#3-agent-orchestration--parallel-task-execution)
4. [Shared Company Knowledge & Controlled Learning](#4-shared-company-knowledge--controlled-learning)
5. [ERP Integration Strategy](#5-erp-integration-strategy)
6. [Authentication, Authorization & Secure Agent Identity](#6-authentication-authorization--secure-agent-identity)
7. [Local vs Cloud vs Hybrid AI Infrastructure](#7-local-vs-cloud-vs-hybrid-ai-infrastructure)
8. [Voice Interaction, Employee Communication & Task Follow-ups](#8-voice-interaction-employee-communication--task-follow-ups)
9. [Monitoring, Audit Trails, Approvals & Recovery](#9-monitoring-audit-trails-approvals--recovery)
10. [Estimated Infrastructure & Operational Requirements](#10-estimated-infrastructure--operational-requirements)
11. [Architecture Alternatives & Trade-offs](#11-architecture-alternatives--trade-offs)
12. [Security Risks & Mitigations](#12-security-risks--mitigations)
13. [Unresolved Questions Requiring CEO Input](#13-unresolved-questions-requiring-ceo-input)
14. [Phased Implementation Roadmap](#14-phased-implementation-roadmap)
15. [Minimum Technical Information Needed to Proceed](#15-minimum-technical-information-needed-to-proceed)

---

## 1. Recommended Overall Architecture

### 1.1 Architecture Pattern: Hierarchical Supervisor + Centralized Checkpointed State

Based on research across LangGraph, AutoGen, CrewAI, Semantic Kernel, and Microsoft Magentic-One, the **only production-hardened pattern** meeting defence/manufacturing requirements is:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DIGITAL CEO (Supervisor)                     │
│  • Strategic objective decomposition                            │
│  • Specialist routing based on shared state                    │
│  • Result synthesis & executive directives                      │
│  • Human-in-the-loop escalation gates                           │
└──────────────────────┬──────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
┌───────────────┐ ┌───────────┐ ┌───────────┐ ┌───────────────┐
│  Sales &      │ │ Engineering│ │ Procure-  │ │ Quality &     │
│  Enquiry      │ │ & BOM      │ │ ment      │ │ Inspection    │
│  Agent        │ │ Agent      │ │ Agent     │ │ Agent         │
└───────┬───────┘ └─────┬─────┘ └─────┬─────┘ └───────┬───────┘
        │               │             │               │
        └───────────────┴─────────────┴───────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  SHARED STATE + CHECKPOINTS │
         │  (PostgreSQL + pgvector)    │
         └─────────────────────────────┘
```

**Why This Pattern:**
- ✅ Deterministic, auditable execution (LangGraph StateGraph)
- ✅ Failure isolation with graceful degradation
- ✅ Classification-aware routing (messages tagged; agents cleared accordingly)
- ✅ Immutable audit trails with cryptographic signing
- ✅ Air-gap compatibility (zero external API dependencies)
- ✅ Human-in-the-loop authority at safety gates

**Technology Stack:**
| Layer | Choice | Justification |
|-------|--------|---------------|
| Orchestration | **LangGraph** | Graph-based, checkpoints, OpenTelemetry, production-hardened |
| Durability | **Temporal** | Workflow-as-code, automatic retries, replay, saga support |
| State Store | **PostgreSQL + pgvector** | ACID, encryption-at-rest, vector search for RAG |
| Message Bus | **NATS JetStream** | Durability, replay, subject-based security, air-gap ready |
| Isolation | **gVisor / Kata Containers** | Hardware-level isolation, defence-grade |
| Observability | **OpenTelemetry + Grafana + Loki** | Vendor-neutral, FIPS-compatible |
| Secrets | **HashiCorp Vault / AWS KMS** | FIPS 140-2 compliant |
| Policy Engine | **OPA (Rego)** | Policy-as-code, auditable, classification-aware |

### 1.2 Architecture Pillars

| Pillar | Implementation |
|--------|----------------|
| **Communication** | Structured message envelopes (correlation ID, classification tag, priority, retry policy, TTL). Shared state > message passing for auditability. |
| **State Management** | Centralized LangGraph state with PostgreSQL checkpointing; per-agent namespaced state sections; classification-tagged state fields. |
| **Task Delegation** | CEO Planner → Task Graph → Parallel Specialist Execution → Aggregator → Directive. Skills Registry for dynamic specialist selection. |
| **Failure Isolation** | Agent-level circuit breakers; process-level gVisor/Kata containers; workflow-level Saga patterns; orchestration-level Temporal retries. |
| **Observability** | OpenTelemetry distributed tracing; LangGraph time-travel debugging; structured JSON logging; LangSmith evaluation; Grafana/Loki metrics. |

---

## 2. Central CEO Agent & Specialist Agent Design

### 2.1 CEO Agent Responsibilities

| Capability | Description |
|------------|-------------|
| **Strategic Decomposition** | Break high-level objectives (e.g., "Win Order X") into specialist tasks |
| **Dynamic Routing** | Route to specialists based on shared state, workload, and classification |
| **Synthesis** | Consolidate specialist outputs into executive recommendations |
| **Blocker Detection** | Identify cross-functional dependencies and resource conflicts |
| **Escalation** | Route decisions exceeding autonomy threshold to Human CEO |
| **Learning Capture** | Record approved corrections as contextual training material |

### 2.2 Specialist Agent Portfolio

| Agent | Domain | Key Functions | Classification Access |
|-------|--------|---------------|----------------------|
| **Sales Enquiry Agent** | Customer RFQs, pipeline tracking, competitor analysis | Parse enquiries, match capabilities, draft responses | Internal / Confidential |
| **Engineering & BOM Agent** | Drawing analysis, BOM intelligence, routings | CAD parsing, BOM explosion, make/buy analysis | Confidential / Secret |
| **Costing & Quotation Agent** | Material, labour, overhead, margin calculations | Multi-scenario costing, sensitivity analysis, quote generation | Confidential |
| **Vendor & Procurement Agent** | Supplier qualification, PO management, delivery tracking | Vendor scoring, auto-PO, exception handling | Internal / Confidential |
| **PPC & Manufacturing Agent** | Scheduling, capacity planning, shop floor coordination | Finite scheduling, work order release, progress tracking | Confidential |
| **Quality & Inspection Agent** | Inspection plans, NCR management, FAI, PPAP | Statistical process control, root cause analysis | Confidential / Secret |
| **Stores & Inventory Agent** | Stock levels, reorder points, shelf-life, traceability | MRP integration, lot tracking, obsolescence alerts | Internal |
| **Accounts & Commercial Agent** | Invoicing, payments, GST, LC/BG management | Cash flow forecasting, credit risk, compliance | Confidential |
| **Employee Task & Reporting Agent** | Task assignment, progress tracking, KPI dashboards | Daily standup automation, escalation alerts, management reports | Internal |

### 2.3 Agent Design Principles

1. **Single Responsibility** – Each agent owns one business domain
2. **Stateless Execution** – State lives in shared checkpointed store, not agent memory
3. **Typed Interfaces** – Pydantic schemas as contracts between CEO and specialists
4. **Capability-Based Permissions** – Tools/functions granted via OPA policies, not implicit
5. **Deterministic Replay** – Pin model versions, set seeds, log all inputs/outputs

---

## 3. Agent Orchestration & Parallel Task Execution

### 3.1 Hybrid Orchestration Architecture

Two-layer approach required for manufacturing workflows:

```
┌─────────────────────────────────────────────────────────────────┐
│  TEMPORAL — Durable Workflow Backbone (24/7, exactly-once)     │
│  • Work Order lifecycle (days/weeks)                            │
│  • Supply chain coordination (procurement → logistics → recv)   │
│  • Saga/compensation for rollbacks                              │
├─────────────────────────────────────────────────────────────────┤
│  LANGGRAPH — Agent Coordination Layer                           │
│  • Production cell agents: CNC, QA, Robot, AGV, Scheduler      │
│  • Planning agents: Demand → Capacity → Schedule                │
│  • Human-in-the-loop quality gates via `interrupt()`           │
├─────────────────────────────────────────────────────────────────┤
│  PREFECT/DAGSTER — Data & ML Pipeline Layer                     │
│  • OEE/Quality analytics from IoT/SCADA                        │
│  • Predictive maintenance ML pipelines                         │
│  • Supply chain forecasting                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Parallel Execution Patterns for Manufacturing

| Pattern | Framework | Manufacturing Use Case |
|---------|-----------|------------------------|
| **Fan-out/Fan-in** | LangGraph, Prefect | Parallel QC checks on 50 parts → aggregate pass/fail |
| **Pipeline/Stage** | CrewAI Flows, Temporal | Sequential: CNC → Deburr → Inspect → Coat → Package |
| **DAG Dependencies** | Temporal, Dagster | Assembly: wait for (subassembly-A AND subassembly-B) → final-assembly |
| **Dynamic Task Gen** | Prefect `task.map()`, LangGraph subgraphs | Maintenance agent spawns (inspect × 3, repair × 2, clean) on fault |

### 3.3 Critical Manufacturing Requirements Checklist

| Requirement | LangGraph + Temporal | AutoGen | CrewAI | Prefect | Airflow |
|-------------|---------------------|---------|--------|---------|---------|
| Exactly-once processing | ✅ | ❌ | ❌ | ⚠️ | ❌ |
| Workflow resumability | ✅ | ❌ | ✅ | ✅ | ❌ |
| Human approval gates | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| FDA/ISO audit trail | ✅ | ❌ | ⚠️ | ✅ | ✅ |
| Sub-second coordination | ✅ | ✅ | ✅ | ⚠️ | ❌ |

---

## 4. Shared Company Knowledge & Controlled Learning

### 4.1 Hybrid Knowledge Architecture: "Company Brain"

| Layer | Technology | Role |
|-------|------------|------|
| **Structural** | **Neo4j / FalkorDB** | BOM structures, routing sheets, manufacturing process relationships, org hierarchy |
| **Semantic** | **Qdrant / Weaviate** | Unstructured manuals, quality reports, CAD notes, project communications |
| **Orchestration** | **LangChain / LlamaIndex** | Agentic RAG mediating between Graph and Vector lookups |

**GraphRAG Advantage:** Manufacturing entities (Part A `ASSEMBLY_OF` Part B `FOLLOWS` Standard ISO9001) traversed as relationships, not semantic matching — critical for complex BOM auditing.

### 4.2 Controlled Learning Pipeline (Observe → Validate → Reuse)

```
┌─────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌─────────────────┐
│  Ingestion  │───▶│ AI Peer Review   │───▶│ Human Approval │───▶│ Versioned Vault │
│  & Staging  │    │ vs Standards     │    │ (HITL Gate)    │    │ (Rollbackable)  │
└─────────────┘    └──────────────────┘    └──────────────┘    └─────────────────┘
       ▲                                                                │
       │                         ┌────────────────────────────────────┘
       └─────────────────────────┤
                                 ▼
                        ┌──────────────────┐
                        │ PEFT/LoRA        │
                        │ Fine-tuning on   │
                        │ Validated Slices │
                        └──────────────────┘
```

**Validation Gates:**
1. **AI Peer Review** – New content checked against AS9100/NADCAP standards + historical failure logs
2. **Human-in-the-Loop** – Expert operators approve/reject based on agent critique
3. **Versioned Knowledge Vaults** – Approved knowledge promoted; previous versions archived for rollback
4. **Catastrophic Forgetting Mitigation** – LoRA adapters on validated slices; replay buffers for foundational standards

### 4.3 Context Management & Access Control

- **Hierarchical Retrieval**: Global (policies) → Domain (engineering specs) → Task (active BOM)
- **RBAC at Query Planner Level** – Planner verifies role against Knowledge Graph metadata before fetching
- **Compartmentalization** – Access by project/classification (e.g., AS9100 cert required for quality specs)

### 4.4 Manufacturing Domain Schema Initialization

Explicit entity definitions for:
- `BOM`, `Routing`, `Tooling`, `QualitySpec`, `Standard`
- Event-driven updates: Quality Non-Conformance → auto-flag related knowledge as "under review"

---

## 5. ERP Integration Strategy

### 5.1 Core Principle: Non-Invasive Integration

**Do not assume unrestricted database access.** Integration via controlled, auditable interfaces:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   ELITE AQMS-   │     │  INTEGRATION     │     │    ELITE AI     │
│     ERP         │     │     LAYER        │     │   AGENTS        │
│                 │     │                  │     │                 │
│  • Read Replica │────▶│  • CDC (Debezium)│────▶│  • Read Models  │
│  • API Gateway  │     │  • API Gateway   │     │  • Write via    │
│  • Event Bus    │     │  • DB Proxy      │     │    API Only     │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

### 5.2 Integration Patterns by Use Case

| Use Case | Pattern | Security Control |
|----------|---------|------------------|
| **Real-time queries** (stock, WIP) | API Gateway + mTLS + OAuth2/JWT + rate limiting | Read-only API keys per agent |
| **Bulk analytics** (costing, forecasting) | CDC → Read Replica → Columnar store (ClickHouse) | No direct ERP DB access |
| **Transaction initiation** (PO, WO release) | API Gateway → ERP workflow engine → async callback | Human approval gate mandatory |
| **Event streaming** (status changes) | Kafka/NATS with Schema Registry + ACLs | Topic-level classification tags |
| **Master data sync** (vendors, items) | Scheduled ETL with checksum validation | Write via ERP API only |

### 5.3 ERP Technical Discovery Required

Before implementation, **must obtain**:
- ERP framework, language, database schema
- Existing API surface (REST/GraphQL/SOAP)
- Authentication mechanism (SSO, LDAP, custom)
- Change management / deployment process
- Performance baselines (query latency, throughput)
- Customization extension points

---

## 6. Authentication, Authorization & Secure Agent Identity

### 6.1 Identity Model

| Entity | Identity Type | Credential | Rotation |
|--------|--------------|------------|----------|
| **Human Users** | Enterprise ID (AD/Azure AD) | FIDO2 / Cert + MFA | 90 days |
| **CEO Agent** | Service Identity (SPIFFE ID) | mTLS cert (Vault PKI) | 24 hours |
| **Specialist Agents** | Service Identity (SPIFFE ID) | mTLS cert (Vault PKI) | 24 hours |
| **ERP Integration** | Service Account | JWT + mTLS | 30 days |
| **External Systems** | Federated Identity | OIDC / SAML | Per contract |

### 6.2 Authorization Framework: Attribute-Based Access Control (ABAC)

```
Policy Decision Point (OPA) evaluates:
┌─────────────────────────────────────────────────────────────┐
│  SUBJECT: agent=ceo, clearance=secret, project=missile-x   │
│  RESOURCE:   bom=airframe, classification=secret           │
│  ACTION:     read, write                                   │
│  CONTEXT:    time=business-hours, location=secure-zone     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
            ALLOW / DENY / OBLIGATIONS
            (e.g., "require HITL for write")
```

**Classification-Aware Routing:**
- Every message tagged: `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `SECRET`
- Agents only receive messages at/below their clearance
- Cross-classification flows require explicit downgrader/CDG

### 6.3 Agent-to-Agent Security

- **mTLS everywhere** – Istio Ambient mesh with SPIFFE workload identity
- **Capability Tokens** – Short-lived JWTs encoding permitted tools/functions
- **Tool Use Restrictions** – OPA policies define which agents can call which ERP APIs
- **Prompt Injection Defense** – Input sanitization + instruction hierarchy + output validation (see Section 12)

---

## 7. Local vs Cloud vs Hybrid AI Infrastructure

### 7.1 Deployment Options Comparison

| Factor | **Fully On-Premise** | **Hybrid (Recommended)** | **Cloud-First** |
|--------|---------------------|-------------------------|-----------------|
| **Data Sovereignty** | ✅ Absolute | ✅ Classified on-prem | ❌ Cross-border risk |
| **Model Access** | Open-weight only | Best of both | ✅ Frontier models |
| **CapEx** | High (₹2-5 Cr) | Medium (₹50L-2Cr) | Low (OpEx) |
| **OpEx** | Power, cooling, staff | Split | ✅ Predictable |
| **Scalability** | Fixed capacity | Burst to cloud | ✅ Elastic |
| **Compliance** | Easiest for Secret | Complex but auditable | Unclassified only |
| **Latency** | Sub-ms (local) | Hybrid routing | Variable |

### 7.2 Recommended: Hybrid Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ CLASSIFIED ENCLAVE (On-Premise / Sovereign Cloud)              │
│ • Secret/Confidential inference & training                     │
│ • Llama 3.1 70B / Qwen 2.5 72B / Mistral Large 2               │
│ • vLLM / TGI on H100/A100 cluster                              │
│ • Air-gapped, STQC-certified, DRDO-aligned                     │
├─────────────────────────────────────────────────────────────────┤
│ CROSS-DOMAIN GUARD (Owl CDG-5000 / Forcepoint)                 │
│ • Protocol break + content inspection                           │
│ • IL5/IL6 certified                                             │
├─────────────────────────────────────────────────────────────────┤
│ GOVCLOUD BURST (Azure India / AWS GovCloud / GCP Assured)      │
│ • Unclassified workloads: GPT-4o, Gemini 1.5, Claude           │
│ • Elastic training on p5.48xlarge / HyperPod                   │
│ • Azure Stack Hub / GDC Air-gapped for disconnected ops        │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 On-Premise Model Recommendations

| Model | Size | Strength | Hardware (Int4) |
|-------|------|----------|-----------------|
| **Llama 3.1/3.2** | 8B, 70B, 405B | Industry standard, tooling | 8B→8GB, 70B→48GB, 405B→240GB+ |
| **Qwen 2.5** | 7B-72B | Best small-model perf, multilingual | 7B→6GB, 72B→48GB |
| **Mistral Large 2** | 123B | Reasoning, efficiency | 48GB+ |
| **Command R+** | 104B | RAG & tool-use optimized | 48GB+ |
| **Phi-3/4** | 3.5B-14B | Edge/constrained | 3.5B→3GB |

**Inference Engine:** vLLM (production) / Ollama (prototyping)  
**Fine-tuning:** LoRA/QLoRA on validated knowledge slices only

### 7.4 Indian Cloud Provider Matrix

| Provider | MeitY CSP | STQC | GenAI Service | Best For |
|----------|-----------|------|---------------|----------|
| **Azure India** | ✅ | ✅ | **OpenAI GPT-4o (GA)** | Unclassified GenAI, Hybrid |
| **AWS Mumbai** | ✅ | ✅ | Bedrock (Claude, Llama) | ML Platform, SageMaker |
| **GCP Mumbai** | ✅ | ✅ | Vertex AI (Gemini 1.5) | ML Training, Cost Optimization |
| **Tata/CtrlS/Sify** | ✅ | ✅ | Limited/Partner | **Classified**, Air-gap, Cost |
| **Jio Cloud** | ✅ | ✅ | Emerging | 5G Edge, Shop Floor IoT |

**Cost Advantage:** Indian sovereign providers 20-40% lower compute, zero egress fees.

---

## 8. Voice Interaction, Employee Communication & Task Follow-ups

### 8.1 Voice Architecture

| Component | Technology | Notes |
|-----------|------------|-------|
| **ASR (Speech-to-Text)** | Whisper (local) / Azure Speech / Google Chirp | Hindi/English/Marathi support |
| **TTS (Text-to-Speech)** | Piper (local) / Azure Neural TTS / ElevenLabs | Voice cloning for familiar tones |
| **Wake Word** | Porcupine / custom on-device | Privacy-preserving |
| **VAD** | Silero VAD (local) | Low-latency turn detection |

**Deployment:** On-premise ASR/TTS for classified areas; cloud for general office.

### 8.2 Employee Communication Channels

| Channel | Use Case | Integration |
|---------|----------|-------------|
| **Voice Assistant** | Hands-free shop floor queries, safety logs | Wearable / Tablet / Kiosk |
| **Chat (Teams/Slack/Custom)** | Task assignment, approvals, alerts | Bot framework + adaptive cards |
| **Mobile App** | Field staff, supervisors, managers | Push notifications, offline queue |
| **Display Boards** | Andon, OEE, schedule visibility | MQTT → Digital signage |

### 8.3 Task Follow-up System

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│  Assignment │───▶│  Tracking    │───▶│  Escalation     │
│  (CEO Agent)│    │  (Employee   │    │  (Auto + Human) │
└─────────────┘    │   Agent)     │    └─────────────────┘
                   └──────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         Reminder   Auto-      Manager
         (T+4h)     reassign   Alert (T+24h)
                  (T+12h)
```

- **Natural Language Task Creation** – "Assign QC check on Batch 104 to Ramesh by 2 PM"
- **Progress Nudges** – Context-aware, not spammy (respects shift schedules)
- **Completion Verification** – Photo/voice/signature capture for critical tasks
- **Audit Trail** – Immutable log of all assignments, acknowledgments, completions

---

## 9. Monitoring, Audit Trails, Approvals & Recovery

### 9.1 Four-Layer Observability

| Layer | Tool | Purpose |
|-------|------|---------|
| **Infrastructure** | Grafana + Prometheus + Loki | Node health, GPU utilization, network |
| **Platform** | OpenTelemetry + Jaeger | Distributed traces, service dependencies |
| **Agent/Application** | LangSmith + Custom Dashboards | Agent decisions, token usage, latency, quality |
| **Business** | Custom KPI Dashboards | Order cycle time, quote win rate, quality PPM |

### 9.2 Immutable Audit Trail Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│  Every Decision │───▶│  Structured Log  │───▶│  WORM Storage       │
│  (Agent/Human)  │    │  (JSON + Sig)    │    │  (S3 Glacier Vault  │
└─────────────────┘    └──────────────────┘    │  / Azure Immutable) │
                                               └──────────┬──────────┘
                                                          │
                              ┌───────────────────────────┘
                              ▼
                    ┌─────────────────────┐
                    │  Merkle Tree + HSM  │
                    │  Signing (daily)    │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
             SIEM Ingestion        Compliance Evidence
             (Sentinel/Splunk)     (CMMC/ITAR/DFARS)
```

**Log Contents:** Timestamp, Actor (human/agent), Classification, Input Hash, Decision, Rationale, Approver, Model Version, Seed.

### 9.3 Human Approval Gates (HITL)

| Trigger | Gate Type | Escalation |
|---------|-----------|------------|
| **Safety-critical** (kinetic, pressure vessel) | Mandatory dual-approval | Plant Manager + Quality Head |
| **Financial > ₹10L** | Single approval + audit | CFO notification |
| **Vendor qualification** | Committee review | Procurement Lead |
| **Schedule change > 20%** | Supervisor approval | PPC Manager |
| **Quality concession** | MRB (Material Review Board) | Quality Head + Customer Rep |

### 9.4 Recovery & Resilience

| Scenario | RTO | RPO | Mechanism |
|----------|-----|-----|-----------|
| **Agent crash** | < 30s | 0 | Temporal replay from last checkpoint |
| **Database failure** | < 5 min | < 1s | PostgreSQL streaming replica + pgBackRest |
| **Inference cluster loss** | < 15 min | 0 | Multi-AZ vLLM + model cache sync |
| **Ransomware/Encryption** | < 4 hr | < 1 hr | Immutable backups + air-gapped vault |
| **Full site loss** | < 24 hr | < 1 hr | DR site (sovereign cloud) with encrypted replicas |

---

## 10. Estimated Infrastructure & Operational Requirements

### 10.1 Hardware Estimate (3-Year Horizon)

| Component | Phase 1 (Pilot) | Phase 2 (Scale) | Phase 3 (Full) |
|-----------|-----------------|-----------------|----------------|
| **GPU Cluster (On-Prem)** | 4× H100 80GB | 16× H100 80GB | 32× H100/H200 |
| **CPU Servers** | 4× Dual EPYC | 12× Dual EPYC | 24× Dual EPYC |
| **Storage (NVMe)** | 100 TB | 500 TB | 1.5 PB |
| **Network** | 100 GbE backbone | 200 GbE + 400 GbE | 400 GbE + 800 GbE |
| **Sovereign Cloud Burst** | ₹15L/yr | ₹60L/yr | ₹1.5Cr/yr |

### 10.2 Operational Team (Phase 3)

| Role | Count | Notes |
|------|-------|-------|
| **MLOps / Platform Engineers** | 4 | Kubernetes, vLLM, Temporal, monitoring |
| **AI/ML Engineers** | 3 | RAG, fine-tuning, evaluation, guardrails |
| **Security Engineers** | 2 | Zero-trust, OPA, Vault, compliance automation |
| **Data Engineers** | 2 | CDC, Knowledge Graph, pipelines |
| **Domain SMEs (Manufacturing)** | 3 | Embedded in agent teams for validation |
| **Total** | **14** | Plus shared DevSecOps, IT support |

### 10.3 3-Year TCO Estimate

| Category | Year 1 | Year 2 | Year 3 | Total |
|----------|--------|--------|--------|-------|
| **Hardware (CapEx)** | ₹3.5 Cr | ₹1.5 Cr | ₹1.0 Cr | ₹6.0 Cr |
| **Cloud Burst (OpEx)** | ₹0.3 Cr | ₹0.6 Cr | ₹1.2 Cr | ₹2.1 Cr |
| **Personnel** | ₹2.5 Cr | ₹3.0 Cr | ₹3.5 Cr | ₹9.0 Cr |
| **Software/Licenses** | ₹0.5 Cr | ₹0.5 Cr | ₹0.5 Cr | ₹1.5 Cr |
| **Facilities/Power** | ₹0.4 Cr | ₹0.6 Cr | ₹0.8 Cr | ₹1.8 Cr |
| **Total** | **₹7.2 Cr** | **₹6.2 Cr** | **₹7.0 Cr** | **₹20.4 Cr** |

*(~$2.4M USD at ₹85/USD)*

---

## 11. Architecture Alternatives & Trade-offs

### 11.1 Alternative 1: Peer-to-Peer Agent Mesh (AutoGen GroupChat)

| Pros | Cons |
|------|------|
| Flexible, emergent problem-solving | ❌ Non-deterministic, unauditable |
| Good for brainstorming | ❌ Debugging nightmare |
| No central bottleneck | ❌ No policy enforcement |
| | ❌ Unsuitable for defence compliance |

**Verdict:** Reject for production. Use only for internal ideation.

### 11.2 Alternative 2: CrewAI Hierarchical Process

| Pros | Cons |
|------|------|
| Explicit role assignment | ❌ Less mature checkpointing |
| Built-in human feedback | ❌ Limited exactly-once guarantees |
| Good task decomposition | ❌ Smaller ecosystem |

**Verdict:** Viable for Phase 1 prototype; migrate to LangGraph+Temporal for production.

### 11.3 Alternative 3: Semantic Kernel Planner-Executor

| Pros | Cons |
|------|------|
| Enterprise plugin model | ❌ .NET-centric |
| Strong skill registry | ❌ Less agent-native |
| Microsoft enterprise support | ❌ Limited manufacturing references |

**Verdict:** Consider if existing .NET/ERP stack; otherwise LangGraph preferred.

### 11.4 Alternative 4: Fully Cloud (Azure OpenAI + Azure AI Studio)

| Pros | Cons |
|------|------|
| Fastest time-to-value | ❌ Classified data prohibited |
| Managed infrastructure | ❌ Vendor lock-in (high egress) |
| Frontier models | ❌ Sovereignty risk |

**Verdict:** Use only for unclassified workloads in hybrid model.

---

## 12. Security Risks & Mitigations

### 12.1 Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **Prompt Injection** | High | Critical | 4-layer defense (sanitization, instruction hierarchy, tool restrictions, output validation) |
| **Data Exfiltration** | Medium | Critical | Classification-aware routing, air-gap, CDG, DLP on all egress |
| **Model Poisoning** | Low | High | Signed model artifacts, SLSA L3 provenance, reproducible builds |
| **Agent Hijacking** | Medium | High | mTLS + SPIFFE, capability tokens, runtime eBPF monitoring (Tetragon) |
| **Supply Chain Attack** | Medium | High | SBOM for all containers, sigstore signing, private registries |
| **Insider Threat** | Medium | High | ABAC + UEBA, immutable audit, dual-approval for sensitive actions |
| **Availability (Ransomware)** | Medium | Critical | Immutable backups, air-gapped vault, tested DR |
| **Compliance Violation** | Low | Critical | Policy-as-code (OPA), automated evidence, continuous ATO |

### 12.2 Prompt Injection Defense (4 Layers)

1. **Input Sanitization** – Regex + ML classifier for injection patterns; delimiter-based parsing
2. **Instruction Hierarchy** – System prompt > Developer prompt > User prompt (XML-delimited)
3. **Tool Restrictions** – Capability-based tokens; no raw SQL/Shell; HITL for write operations
4. **Output Validation** – Schema validation; PII/classification scanning; anomaly detection

### 12.3 Compliance Mapping

| Standard | Applicable Controls |
|----------|---------------------|
| **CMMC 2.0 Level 3** | AC.L2-3.1.1-3.1.22, AU.L2-3.3.1-3.3.9, SC.L2-3.13.1-3.13.16 |
| **NIST 800-171** | 3.1-3.14 families (Access Control, Audit, Config Mgmt, IA, IR, MA, MP, PE, PL, PS, RA, SC, SI) |
| **ITAR** | Data residency, encryption, access control, audit, visitor control |
| **DFARS 252.204-7012** | Adequate security, cyber incident reporting, cloud requirements |
| **NIST AI RMF** | GOVERN 1-4, MAP 1-4, MEASURE 1-4, MANAGE 1-4 |
| **DPDP Act 2023** | Consent, purpose limitation, data minimization, security safeguards |

---

## 13. Unresolved Questions Requiring CEO Input

| # | Question | Decision Needed |
|---|----------|-----------------|
| **1** | What is the **exact classification breakdown** of data across domains? (e.g., % Secret vs Confidential vs Internal) | Drives enclave sizing, CDG requirements |
| **2** | **Autonomy threshold** – Which decisions can agents make autonomously vs. require human approval? | Defines HITL gate placement |
| **3** | **ERP Technical Details** – Framework, DB, API surface, auth, customization points? | Required for integration design |
| **4** | **Existing Infrastructure** – Current cloud, on-prem, network, security tools? | Avoids duplication, leverages existing |
| **5** | **Voice Language Priority** – Hindi, English, Marathi, others? Regional accents? | Drives ASR/TTS model selection |
| **6** | **Pilot Scope** – Which 2-3 domains for Phase 1? (Recommend: Sales Enquiry + Engineering/BOM + Quality) | Focuses MVP |
| **7** | **Budget Approval** – CapEx for on-premise GPU cluster vs. sovereign cloud rental? | Determines Phase 1 hardware |
| **8** | **Team Composition** – Internal hiring vs. partner vs. hybrid? | Affects timeline and knowledge retention |
| **9** | **Regulatory Landscape** – Specific DRDO/DGAQA/MoD requirements beyond CMMC/ITAR? | May add controls |
| **10** | **Success Metrics** – KPIs for Phase 1 (e.g., quote turnaround -30%, NCR detection +50%)? | Defines "done" |

---

## 14. Phased Implementation Roadmap

### Phase 1: Foundation & Pilot (Months 1-6)

| Milestone | Deliverable | Success Criteria |
|-----------|-------------|------------------|
| **1.1** | Technical Discovery | ERP API spec, DB schema, auth, infra inventory documented |
| **1.2** | Secure Enclave Setup | Air-gapped Kubernetes + vLLM + PostgreSQL + Neo4j + Vault operational |
| **1.3** | Core Orchestration | LangGraph + Temporal + NATS deployed; checkpointing verified |
| **1.4** | Pilot Agents (3) | Sales Enquiry, Engineering/BOM, Quality agents functional |
| **1.5** | Knowledge Graph Seed | BOM, Routings, Standards loaded; GraphRAG queries working |
| **1.6** | HITL Gates | Approval workflows for quote >₹5L, quality concession operational |
| **1.7** | Audit & Observability | OpenTelemetry + LangSmith + Grafana + Immutable logging live |
| **1.8** | Pilot Evaluation | 30-day production shadow; metrics vs. baseline documented |

**Go/No-Go Gate:** CEO review of pilot metrics, security audit, team readiness.

---

### Phase 2: Scale & Harden (Months 7-14)

| Milestone | Deliverable |
|-----------|-------------|
| **2.1** | Remaining 6 specialist agents deployed |
| **2.2** | Hybrid cloud burst configured (Azure India GovCloud) |
| **2.3** | Cross-domain guard (CDG) installed & certified |
| **2.4** | Full ERP integration (CDC, API gateway, event streaming) |
| **2.5** | Voice interface (ASR/TTS) for shop floor pilot |
| **2.6** | Employee Task Agent + mobile app beta |
| **2.7** | Automated compliance evidence generation (CMMC/ITAR) |
| **2.8** | Disaster recovery drill (full site failover) |

---

### Phase 3: Autonomy & Optimization (Months 15-24)

| Milestone | Deliverable |
|-----------|-------------|
| **3.1** | Controlled learning pipeline operational (validated knowledge → LoRA) |
| **3.2** | Gradual autonomy expansion per reliability metrics |
| **3.3** | Predictive agents (maintenance, demand, quality) |
| **3.4** | Full voice/chat coverage (all shifts, all languages) |
| **3.5** | Executive dashboard for Human CEO (real-time org view) |
| **3.6** | Continuous ATO (Authority to Operate) automation |
| **3.7** | Knowledge distillation for edge deployment (tablets, wearables) |
| **3.8** | Architecture review & next-gen planning |

---

## 15. Minimum Technical Information Needed to Proceed

Before any implementation, the following **must** be obtained:

| Category | Specific Information |
|----------|---------------------|
| **ERP Technical** | Framework, language, DB (vendor/schema), API spec (OpenAPI/Swagger), Auth mechanism, Deployment topology, Customization framework |
| **Infrastructure** | Current cloud accounts, on-prem server specs, network diagram, firewall zones, VPN/MPLS, Monitoring tools, Backup/DR setup |
| **Security** | Existing certifications (ISO 27001, CMMC, STQC), Identity provider (AD/Azure AD/Okta), HSM/PKI, DLP, SIEM, Incident response plan |
| **Data** | Data classification policy, Data retention schedule, PII/ITAR data map, Current data flows (ERP → MES → CAD → PLM) |
| **Organizational** | Org chart with roles/clearances, Shift schedules, Escalation matrix, Existing SOPs for quote/quality/procurement |
| **Constraints** | Budget (CapEx/OpEx), Timeline hard dates, Regulatory mandates, Vendor preferences, Team skills inventory |

---

## Appendix A: Research Sources

### Agent Architecture
- LangGraph: https://docs.langchain.com/oss/python/langgraph/
- AutoGen 0.4: https://microsoft.github.io/autogen/
- CrewAI: https://docs.crewai.com/
- Semantic Kernel: https://learn.microsoft.com/semantic-kernel/
- Magentic-One: https://www.microsoft.com/en-us/research/project/magentic-one/

### On-Premise AI
- Hugging Face Open LLM Leaderboard: https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard
- NVIDIA TensorRT-LLM: https://github.com/NVIDIA/TensorRT-LLM
- DPDP Act 2023: https://www.meity.gov.in/content/digital-personal-data-protection-act-2023
- llama.cpp: https://github.com/ggerganov/llama.cpp
- Ollama: https://ollama.com/

### Orchestration
- Temporal: https://docs.temporal.io/
- LangGraph Checkpointing: https://langchain-ai.github.io/langgraph/how-tos/persistence/
- Prefect: https://docs.prefect.io/
- Dagster: https://dagster.io/

### Knowledge & Learning
- Neo4j Manufacturing: https://neo4j.com/use-cases/manufacturing/
- LlamaIndex GraphRAG: https://docs.llamaindex.ai/en/stable/optimizing/graph_rag/
- PEFT/LoRA: https://huggingface.co/docs/peft/index

### Indian Cloud & Compliance
- MeitY CSP Empanelment: https://www.meity.gov.in/content/cloud-services-empanelment
- STQC: https://www.stqc.gov.in/
- DPDP Act 2023: https://www.meity.gov.in/content/digital-personal-data-protection-act-2023

### Secure Architecture
- NIST AI RMF: https://www.nist.gov/itl/ai-risk-management-framework
- CMMC 2.0: https://www.acq.osd.mil/cmmc/
- MITRE ATLAS: https://atlas.mitre.org/
- Intel SGX / AMD SEV-SNP / AWS Nitro: Vendor documentation

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **CEO Agent** | Central supervisory agent coordinating specialist agents |
| **Specialist Agent** | Domain-specific agent (Sales, Engineering, Quality, etc.) |
| **HITL** | Human-in-the-Loop – mandatory human approval gate |
| **CDC** | Change Data Capture – database log replication |
| **CDG** | Cross-Domain Guard – hardware security appliance for classification crossing |
| **SPIFFE** | Secure Production Identity Framework For Everyone |
| **OPA** | Open Policy Agent – policy-as-code engine |
| **WORM** | Write Once Read Many – immutable storage |
| **SLSA** | Supply Chain Levels for Software Artifacts |
| **SBOM** | Software Bill of Materials |
| **ATO** | Authority to Operate |
| **MRB** | Material Review Board |
| **FAI** | First Article Inspection |
| **PPAP** | Production Part Approval Process |

---

**END OF DOCUMENT**

---

*This blueprint is a research deliverable, not an implementation specification. All architecture decisions require CEO approval and technical validation against the actual ERP environment before development begins.*