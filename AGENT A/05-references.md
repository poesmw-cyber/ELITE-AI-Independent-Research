# 05 — References

Public sources used, with the specific claim each supports. All accessed November 2026. Where a source is a vendor document describing that vendor's own product, it is treated as authoritative only about that product, not as independent evidence of superiority.

## Security and AI risk

| Ref | Source | Used for |
|---|---|---|
| S-1 | OWASP Top 10 for LLM Applications 2025 — https://genai.owasp.org/llm-top-10/ and PDF: https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf | F-01: prompt injection (LLM01), sensitive information disclosure (LLM02), supply chain (LLM03), data/model poisoning (LLM04), excessive agency (LLM06) and their definitions |
| S-2 | OWASP LLM06:2025 Excessive Agency — https://genai.owasp.org/llmrisk/LLM06:2025-Excessive-Agency | Excessive functionality / permissions / autonomy framing |
| S-3 | OWASP LLM01:2025 Prompt Injection — https://genai.owasp.org/llmrisk/llm01-prompt-injection/ | Direct vs indirect injection; document and multimodal carriers |
| S-4 | NIST AI 600-1, Generative AI Profile (July 2024) — https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf | F-02: GOVERN/MAP/MEASURE/MANAGE structure and GAI risk profile |
| S-5 | NIST AI RMF Core — https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Core_And_Profiles/5-sec-core | Governance vocabulary for the measurement section |
| S-6 | MITRE ATLAS techniques AML.T0051.000/.001 (LLM prompt injection, direct/indirect), AML.T0054 | Cited via OWASP as the adversarial taxonomy |
| S-7 | ISO/IEC 42001:2023 — https://www.iso.org/standard/42001 and https://www.iso.org/insights/iso-42001-explained | F-05: AI management system, clauses 4–10, Annex A controls, impact assessment |
| S-8 | ISO/IEC 27001:2022 alignment commentary (e.g. https://casrai.org/guides/iso-42001-vs-iso-27001) | Shared harmonised structure enabling integrated audit |

## Indian regulatory context

| Ref | Source | Used for |
|---|---|---|
| I-1 | DPDP Rules 2025, MeitY — https://www.meity.gov.in/documents/act-and-policies/digital-personal-data-protection-rules-2025-gDOxUjMtQWa ; text: https://www.dpdpa.in/dpdpa_rules_2025/dpdpa_rules_english_2.pdf | F-03: Rule 6 minimum safeguards (encryption/obfuscation/masking/tokenisation, access control, logs, continuity, one-year retention) |
| I-2 | Commencement timeline analysis — https://indiadatalaw.org/status/ and Lok Sabha answer AU3943 (12.08.2026) | F-03: phased commencement, principal obligations from 13 May 2027 |
| I-3 | CERT-In Directions No. 20(3)/2022 dated 28 April 2022 — https://www.cert-in.org.in/PDF/CERT-In_Directions_70B_28.04.2022.pdf | F-04: 6-hour incident reporting, 180-day log retention in Indian jurisdiction, NTP sync |
| I-4 | CERT-In FAQ (May 2022) — https://www.cert-in.org.in/PDF/FAQs_on_CyberSecurityDirections_May2022.pdf | Scope of covered entities, clock synchronisation interpretation, log types |
| I-5 | Legal commentary on the Directions (Trilegal, Khaitan) | Practical reading of the 6-hour obligation and platform duties |

## Inference and infrastructure

| Ref | Source | Used for |
|---|---|---|
| P-1 | vLLM documentation — https://docs.vllm.ai/ ; serving: https://docs.vllm.ai/en/latest/serving/online_serving/openai_compatible_server ; parallelism: https://docs.vllm.ai/en/latest/serving/parallelism_scaling/ | F-06: OpenAI-compatible server, PagedAttention, tensor/pipeline parallelism, quantisation, API-key endpoint coverage warning, Linux-only |
| P-2 | vLLM GPU installation requirements — https://docs.vllm.ai/en/stable/getting_started/installation/gpu | Compute capability requirements; no native Windows support |
| P-3 | NVIDIA NIM air-gap deployment — https://docs.nvidia.com/nim/large-language-models/latest/deploy-air-gap.html ; enterprise-grade page | F-07: two-phase air-gap staging, no registry access at runtime, no API keys in the air-gapped phase |
| P-4 | NVIDIA NIM LLM reference architecture, sizing — https://docs.nvidia.com/enterprise-reference-architectures/nim-llm-runai-vanilla-kubernetes/latest/sizing-guidelines.html | Calibration figure: ~137 concurrent users on one H100 NVL for Llama-3.1-8B at ~1,000 ms TTFT (used as an order-of-magnitude reference, not a sizing claim for our workload) |
| P-5 | NVIDIA NIM architecture — https://docs.nvidia.com/nim/large-language-models/latest/reference/architecture.html | NIM wrapping vLLM/SGLang behind a proxied OpenAI-compatible API |

## Orchestration and integration

| Ref | Source | Used for |
|---|---|---|
| O-1 | LangGraph overview — https://docs.langchain.com/oss/python/langgraph | F-08: durable execution, persistence, human-in-the-loop, memory, streaming |
| O-2 | LangGraph checkpoints and durability modes — https://docs.langchain.com/oss/python/langgraph/checkpointers | Super-step checkpoint boundaries; re-execution semantics |
| O-3 | LangGraph interrupts — https://docs.langchain.com/oss/python/langgraph/interrupts ; blog: https://langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt | Pause/edit/resume; persistence as a first-class citizen |
| O-4 | LangGraph time travel — https://docs.langchain.com/oss/python/langgraph/use-time-travel | Replay and fork from checkpoints |
| O-5 | Temporal, LangGraph plugin and durable execution commentary — https://temporal.io/blog/temporal-langgraph-plugin-durable-execution | Checkpoints preserve state, not execution; external orchestrator needed for crash recovery |
| O-6 | AWS Prescriptive Guidance, transactional outbox — https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html | F-11: dual-write problem and outbox solution |
| O-7 | Debezium outbox event router — https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html | F-11: CDC-based outbox publication |
| O-8 | Change-data-capture vs application events discussion (Red Hat architecture centre; field guides) | F-11: table-change vs business-event trade-off, idempotent consumers |

## Identity and authorization

| Ref | Source | Used for |
|---|---|---|
| A-1 | RFC 8693, OAuth 2.0 Token Exchange | F-10: delegation, down-scoping, `act` claim |
| A-2 | RFC 9449, DPoP | Sender-constrained tokens |
| A-3 | SPIFFE specifications — https://spiffe.io/docs/latest/spiffe-specs/spiffe-id and workload API | F-10: workload identity, SVIDs, trust domains |
| A-4 | Microsoft Entra Agent ID OBO flow — https://learn.microsoft.com/en-us/entra/agent-id/agent-on-behalf-of-oauth-flow | F-10: production agent identity implementation on RFC 8693 |
| A-5 | Microsoft Entra workload identity federation with SPIFFE/SPIRE — https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-spiffe-spire | F-10: eliminating stored secrets |
| A-6 | AWS Bedrock AgentCore on-behalf-of token exchange — https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-behalf-of-token-exchange.html | F-10: RFC 8693 / RFC 7523 agent delegation |
| A-7 | Google Cloud Agent Identity overview — https://docs.cloud.google.com/iam/docs/agent-identity-overview | F-10: SPIFFE-based agent identity |
| A-8 | Model Context Protocol specification, authorization — https://modelcontextprotocol.io/specification/latest/basic/authorization | F-09: OAuth 2.1, RFC 9728 protected resource metadata, RFC 8707 audience binding |
| A-9 | MCP security best practices — https://modelcontextprotocol.io/docs/.../security_best_practices | F-09: token passthrough prohibition, confused deputy |
| A-10 | MCP 2026-07-28 revision notes — https://blog.modelcontextprotocol.io/posts/2026-07-28/ | Current revision, stateless requests, enterprise managed authorization extension |

## Retrieval, knowledge and learning

| Ref | Source | Used for |
|---|---|---|
| K-1 | pgvector — https://github.com/pgvector/pgvector | HNSW/IVFFlat indexing, recall trade-offs |
| K-2 | Azure Architecture Center, RAG information retrieval — https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-information-retrieval | F-12: hybrid queries, reciprocal rank fusion, weighting |
| K-3 | Supabase, RAG with permissions — https://supabase.com/docs/guides/ai/rag-with-permissions | Row-level security applied to vector search |
| K-4 | AWS Security Blog, authorizing access to data with RAG — https://aws.amazon.com/blogs/security/authorizing-access-to-data-with-rag-implementations/ | F-12: authorization at retrieval time against the authoritative source |
| K-5 | Fail-closed retrieval guidance — https://hammerspace.com/rag-access-control-building-an-authorization-control-plane-from-native-file-permissions-to-fail-closed-retrieval | F-12: filter inside both lexical and dense queries; fail-closed rules |
| K-6 | Microsoft Foundry RAG concepts — https://learn.microsoft.com/en-us/azure/foundry/concepts/retrieval-augmented-generation | Treating retrieved content as untrusted input; document-level access control |

## Audit, observability

| Ref | Source | Used for |
|---|---|---|
| L-1 | S3 Object Lock — https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html | F-13: compliance vs governance mode; no identity can shorten compliance-mode retention |
| L-2 | Tamper-evident log design analysis — https://zatona.dev/blog/tamper-evident-audit-logs | F-13: four levels (retention, hash chain, Merkle + signed checkpoints, external anchor) and what each proves |
| L-3 | OWASP APTS auditability guidance (hash-chained append-only logs) — https://owasp.org/APTS/standard/5_Auditability/Implementation_Guide.html | F-13: SHA-256 chaining, periodic verification |
| L-4 | OpenTelemetry GenAI semantic conventions — https://github.com/open-telemetry/semantic-conventions-genai and https://opentelemetry.io/blog/2026/genai-observability | F-14: invoke_agent / execute_tool / chat spans, attribute names, Development status |
| L-5 | AWS security reference architecture, log archive | Separation of duties, centralised immutable log account |

## Cloud AI in India

| Ref | Source | Used for |
|---|---|---|
| C-1 | AWS, Claude models in India via geographic cross-Region inference — https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-expands-claude-model-availability-to-india-cross-region-inference | F-15: India geographic inference across ap-south-1/ap-south-2 |
| C-2 | AWS, OpenAI models on Bedrock for in-country inferencing in India — https://aws.amazon.com/blogs/machine-learning/introducing-openai-models-on-amazon-bedrock-for-in-country-inferencing-in-india | F-15: India-only routing for selected models |
| C-3 | Amazon Bedrock regional availability — https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints-region-availability.html | F-15: per-region endpoint support differences |
| C-4 | Google Cloud Vertex AI locations — https://docs.cloud.google.com/vertex-ai/docs/general/locations | F-15: asia-south1 (Mumbai), asia-south2 (Delhi) |
| C-5 | Google Cloud generative AI data residency — https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency | F-15: per-model regional processing support |
| C-6 | Azure data residency — https://azure.microsoft.com/en-us/global-infrastructure/data-residency | F-15: India geo commitments, and the caveat that certain services store outside region |
| C-7 | OpenAI data residency — https://developers.openai.com/api/docs/guides/data-residency | F-15: India regional processing for selected models/endpoints |

## Voice and Indic language

| Ref | Source | Used for |
|---|---|---|
| V-1 | AI4Bharat Vistaar / IndicWhisper — https://github.com/AI4Bharat/vistaar ; Interspeech 2023 paper | F-16: fine-tuning on ~10.7k hours across 12 languages, WER benchmarks |
| V-2 | AI4Bharat IndicConformer — https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual ; https://github.com/AI4Bharat/IndicConformerASR | F-16: 600M multilingual ASR, 22 scheduled languages, MIT licence |
| V-3 | Voice of India benchmark (Interspeech 2026) — https://www.isca-archive.org/interspeech_2026/bhogale26_interspeech.pdf | F-16: many systems exceed 20% WER on real-world Indian speech; no system consistent across languages |
| V-4 | Project Vaani open models — https://vaani.iisc.ac.in/models | Whisper fine-tunes across size ladder for Indian languages |
| V-5 | Code-switching ASR study (VITB-HEBiC) — ScienceDirect S0045790624009030 | Hindi-English code-switching accuracy context |

## Notes on evidence quality

- **Strongest:** standards bodies (NIST, ISO/IEC), RFCs, Indian government notifications and CERT-In directions, first-party technical documentation for specific products.
- **Weaker:** vendor blog posts about their own product's advantages; benchmark leaderboards without reproduction details; "users per GPU" claims.
- **Deliberately avoided:** unverifiable claims about "secure AI", marketing comparisons between model providers, and any figure presented without a stated assumption.
- **Time-sensitivity:** cloud model availability, regional support matrices, model versions and pricing change monthly. Every item in section C and P should be re-verified at procurement time, not trusted from this document.
