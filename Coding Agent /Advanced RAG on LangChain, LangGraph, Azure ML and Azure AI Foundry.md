# Advanced RAG on LangChain, LangGraph, Azure ML and Azure AI Foundry

## 1. The insight: each technique fixes a different failure point

| Failure point | Symptom | Technique | Where it sits |
|---|---|---|---|
| Query is vague or depends on chat history | "What about its pricing?" retrieves nothing | **Query rewriting** (decontextualize) | Pre-retrieval |
| Single phrasing misses relevant docs | Low recall | **Multi-query / decomposition / HyDE** | Pre-retrieval |
| Semantic search misses exact terms (IDs, error codes), or keyword search misses meaning | Inconsistent recall | **Hybrid search + RRF fusion** | Retrieval |
| Right doc is in top-50 but not top-5 | Low precision | **Reranking** (cross-encoder) | Post-retrieval |
| Domain vocabulary is poorly embedded | Systematic misses | **Embedding fine-tuning** | Model level |
| Bad context still reaches the LLM | Hallucination | **Relevance grading, CRAG/self-RAG loops** | Control flow |

**Rule of thumb:** fix the cheapest layers first (hybrid, then rerank, then rewriting). Fine-tuning and HyDE come last, and only if evaluation shows a gap.

## 2. Role of each platform component

| Component | Role |
|---|---|
| **LangChain** | Building blocks: loaders, splitters, retrievers, prompt templates, model wrappers, output parsers |
| **LangGraph** | Runtime orchestration: stateful graph with conditional edges, loops, retries, checkpointing, human-in-the-loop |
| **Azure AI Foundry** (now branded Microsoft Foundry) | Model catalog and deployments (chat and embedding), Prompt Flow/evaluations, content safety, tracing, agent services |
| **Azure ML Studio** | Embedding/reranker fine-tuning jobs, datasets, model registry, managed online endpoints, MLOps pipelines |
| **Azure AI Search** | Hybrid index (BM25 + HNSW vector + filters), semantic ranker, security trimming |
| **Supporting** | APIM, Entra ID, Key Vault, Cosmos DB or Redis (memory/cache), Blob/ADLS, Azure Monitor and App Insights, Azure AI Content Safety |

## 3. High-Level Design (text diagram)

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║ PLANE A: OFFLINE INGESTION (batch/event-driven)                                  ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ [A1 Sources]──►[A2 Landing]──►[A3 Parse/Clean]──►[A4 Chunk]──►[A5 Enrich]        ║
║  SharePoint     Blob/ADLS      Doc Intelligence   Structure-    Metadata, ACLs,  ║
║  DBs, PDFs,     (raw zone)     OCR, tables,       aware +       summaries,       ║
║  APIs, Wiki     + event grid   dedupe, PII mask   parent/child  synthetic Qs     ║
║                                                                                  ║
║ ──►[A6 Embed]──────────►[A7 Index: Azure AI Search]──►[A8 Quality gate]          ║
║    Foundry/AML endpoint   BM25 + vector + filters       Index validation,        ║
║    (versioned model)      + semantic config             golden-query smoke test  ║
╚══════════════════════════════════════════════════════════════════════════════════╝
                │ (hard negatives, click logs, golden Q&A)
                ▼
╔══════════════════════════════════════════════════════════════════════════════════╗
║ PLANE B: MODEL IMPROVEMENT LOOP (Azure ML Studio)                                ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ [B1 Training data]─►[B2 Fine-tune]─►[B3 Offline eval]─►[B4 Model Registry]       ║
║  synthetic Q/chunk    embedder +     recall@k, MRR,     versioned, tagged        ║
║  pairs + hard         reranker       nDCG vs baseline   champion/challenger      ║
║  negatives            (AML pipeline)                                             ║
║                                                         │                        ║
║ [B5 Managed Online Endpoint, blue/green]◄───────────────┘                        ║
║ [B6 Re-embed + reindex on model change (new index, alias swap)]──► back to A6/A7 ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║ PLANE C: ONLINE SERVING (LangGraph state machine)                                ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ User ─► [C1 APIM + Entra ID]─►[C2 Input guardrails]─►[C3 Load memory/session]    ║
║          authN/Z, throttle     Content Safety,         Cosmos/Redis, summarize   ║
║                                prompt-injection        old turns                 ║
║                                      │                                           ║
║                               [C4 Intent Router]                                 ║
║                      ┌───────────────┼────────────────┐                          ║
║                  chit-chat       RAG path         tool/SQL/agent path            ║
║                  (direct LLM)       │                                            ║
║                                [C5 Query Rewrite] history-aware, decontextualize ║
║                                     │                                            ║
║                                [C6 Query Expansion]                              ║
║                                 multi-query | decomposition | step-back | HyDE   ║
║                                     │ (N queries, parallel fan-out)              ║
║              ┌──────────────────────┼──────────────────────┐                     ║
║        [C7a BM25 keyword]    [C7b Vector (fine-tuned    [C7c Metadata/           ║
║                                   embedding)]            filter + ACL trim]      ║
║              └──────────────────────┼──────────────────────┘                     ║
║                             [C8 RRF Fusion + Dedupe]  (top ~50)                  ║
║                                     │                                            ║
║                             [C9 Reranker] cross-encoder / semantic ranker (top ~5)║
║                                     │                                            ║
║                             [C10 Context Builder]                                ║
║                              parent expansion, compression, ordering, citations ║
║                                     │                                            ║
║                             [C11 Relevance Grader] ──insufficient──┐             ║
║                                     │ ok                          ▼             ║
║                                     │                  [C11b Corrective loop]    ║
║                                     │                  rewrite again / web /     ║
║                                     │                  fallback (max N retries)  ║
║                             [C12 Grounded Generation] Foundry-deployed LLM       ║
║                                     │                                            ║
║                             [C13 Groundedness + Safety check] ──fail──► retry/refuse║
║                                     │ pass                                       ║
║                             [C14 Response + citations + confidence]              ║
║                                     │                                            ║
║                             [C15 Persist memory + cache + feedback hook]         ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║ PLANE D: LLMOPS / OBSERVABILITY / GOVERNANCE (cross-cutting)                     ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ Tracing (OpenTelemetry → App Insights/Foundry tracing) │ Eval (Foundry evals)    ║
║ RAGAS-style metrics │ Prompt versioning │ CI/CD (GitHub Actions/Azure DevOps)    ║
║ Cost/latency dashboards │ Feedback → Plane B │ Key Vault, Private Link, RBAC     ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

## 4. Step-by-step: design pattern and best practices

### Plane A: Ingestion

| Step | Design pattern | Best practices |
|---|---|---|
| **A1 Sources** | **Adapter / Connector** pattern | One connector per source with a common schema. Use incremental sync (change tokens), not full reloads. Capture source ACLs at ingestion. |
| **A2 Landing** | **Medallion architecture** (raw, curated, indexed) | Immutable raw zone. Event Grid triggers processing. Keep lineage (source URI, hash, version). |
| **A3 Parse/Clean** | **Pipeline (pipes and filters)** | Azure Document Intelligence for layout, tables and OCR. Preserve headings and tables as structure. Dedupe by content hash. Mask PII before indexing if policy requires. |
| **A4 Chunk** | **Strategy pattern** (swappable chunkers) | Structure-aware chunking (headings, sections) over fixed size. About 300–800 tokens with 10–15% overlap. Use **parent-child chunks**: embed small, return large. |
| **A5 Enrich** | **Decorator / Enrichment pipeline** | Add metadata (doc type, date, product, tenant, ACL group). Add chunk summaries and **synthetic questions per chunk** (doc2query), a cheap alternative to HyDE at index time. |
| **A6 Embed** | **Model versioning / Strategy** | Batch with retries and backoff. Store the embedding model version on every record. Never mix embedding versions in one index. |
| **A7 Index** | **CQRS** (separate write and read paths), **Alias swap** | Azure AI Search with vector (HNSW) and keyword fields plus filterable metadata and ACL fields. Use index aliases for zero-downtime reindexing. Tune HNSW (m, efConstruction) via evaluation. |
| **A8 Quality gate** | **Quality gate / Canary** | Run golden queries after each ingestion. Fail the pipeline if recall drops below threshold. |

### Plane B: Model improvement (Azure ML Studio)

| Step | Design pattern | Best practices |
|---|---|---|
| **B1 Training data** | **Data flywheel** | Build (query, positive chunk, hard negative) triples from synthetic LLM-generated questions, user feedback and click logs. Always hold out a test set and a human-reviewed subset. |
| **B2 Fine-tune** | **Pipeline-as-code (AML pipelines)** | Fine-tune the embedder (contrastive or MultipleNegativesRanking loss) and optionally a cross-encoder reranker. Start from a strong base model and track runs with MLflow. |
| **B3 Offline eval** | **Champion/Challenger** | Measure recall@k, MRR and nDCG against the current production model. Promote only on a meaningful gain without regressions on general queries. |
| **B4 Registry** | **Model registry + semantic versioning** | Register with lineage (data version, code commit, metrics). Require approval gates. |
| **B5 Endpoint** | **Blue/Green deployment** | AML managed online endpoints, autoscaling, private networking. Shadow traffic before cutover. |
| **B6 Re-embed** | **Immutable index + alias swap** | A new embedding model means a new index. Validate, then flip the alias. Keep the old index for rollback. |

### Plane C: Online serving (LangGraph)

| Step | Design pattern | Best practices |
|---|---|---|
| **C1 Gateway** | **API Gateway + Zero Trust** | APIM for auth, quotas and rate limiting. Entra ID and managed identities (no keys in code). Private endpoints. |
| **C2 Input guardrails** | **Guardrail / Chain of Responsibility** | Content Safety, prompt-injection and jailbreak detection (Prompt Shields), PII detection, input length limits. Treat retrieved content as untrusted too. |
| **C3 Memory** | **Memory / Repository pattern** | Session state in Cosmos DB or Redis. Summarize old turns to control tokens. Use LangGraph checkpointers for resumable, auditable runs. |
| **C4 Intent router** | **Router / Strategy** | A cheap, small model or classifier decides: chit-chat, RAG, SQL/tool, or refuse. Saves cost and latency by skipping retrieval when unneeded. |
| **C5 Query rewrite** | **Pre-retrieval transform (Decorator)** | Condense history plus the question into a standalone query. Use a small fast model with a low temperature. Preserve entities and identifiers verbatim. Log original and rewritten queries. |
| **C6 Expansion** | **Fan-out / Scatter-Gather** | Generate 3–5 query variants (multi-query). Use **decomposition** for multi-hop questions and **step-back** for conceptual ones. Apply **HyDE selectively** (only for vague conceptual queries, via the router). Run variants in parallel. Cap N to control cost. |
| **C7a/b/c Retrieval** | **Parallel retrievers + Ensemble / Composite** | Run BM25 and vector in parallel (Azure AI Search does this natively in a hybrid query). Apply **security trimming** with ACL filters at query time. Pre-filter by metadata (tenant, date, product) to narrow the search space. |
| **C8 Fusion** | **Reciprocal Rank Fusion (RRF)** | RRF is score-scale agnostic, so no normalization problems. Deduplicate across query variants. Keep about 30–50 candidates for reranking. |
| **C9 Rerank** | **Two-stage retrieval (recall then precision)** | Cross-encoder (fine-tuned in AML) or Azure semantic ranker. Keep the top 3–8. Set a score threshold so weak matches are dropped. Rerank latency scales with candidates, so batch it. |
| **C10 Context builder** | **Builder + Compression** | Expand to parent chunks, order by relevance (put the best at the start and end to mitigate "lost in the middle"), and compress or trim to a token budget. Attach source IDs for citations. |
| **C11 Relevance grader** | **Corrective RAG (CRAG) / Self-RAG, Circuit Breaker** | An LLM or classifier grades whether the context can answer the question. On failure, loop to rewrite or broaden the search, and **cap retries at 1–2**. Fall back to "I don't know" or escalate to a human. |
| **C12 Generation** | **Prompt Template + Structured Output** | Strict system prompt: answer only from context, cite sources, admit uncertainty. Use JSON/Pydantic output schemas. Use Foundry model deployments with fallback models and retries. Use temperature near 0 for factual tasks. |
| **C13 Output validation** | **Guardrail / Verifier (LLM-as-judge)** | Groundedness detection, claim-to-citation check, content safety and PII redaction. On failure, regenerate once or refuse safely. |
| **C14 Response** | **DTO / Response envelope** | Return the answer, citations, confidence and trace ID. Support streaming for perceived latency. |
| **C15 Persist and feedback** | **Event sourcing + Cache-aside** | Store the turn in memory, add **semantic caching** (Redis) for repeated queries, and capture thumbs up/down with the trace ID to feed Plane B. |

### Plane D: LLMOps and governance

| Concern | Pattern | Best practices |
|---|---|---|
| **Tracing** | **Distributed tracing** (OpenTelemetry) | Trace every graph node with inputs, outputs, latency, tokens and cost. Use App Insights and Foundry tracing, plus LangSmith if you choose. |
| **Evaluation** | **Continuous evaluation / Test pyramid** | Separate **retrieval metrics** (recall@k, MRR, nDCG, context precision) from **generation metrics** (faithfulness, answer relevance). Run on a golden set in CI and sample production traffic. |
| **CI/CD** | **GitOps / IaC** | Bicep/Terraform for infra. Version prompts, graphs, chunkers and index schemas. Gate releases on eval thresholds. |
| **Security** | **Defense in depth** | Private Link, VNet integration, Key Vault, RBAC, data residency, audit logs. Per-user ACL trimming is mandatory for enterprise data. |
| **Cost and latency** | **Tiered models / Budgeting** | Small models for routing, rewriting and grading; larger models only for the final answer. Cache, batch, and set token budgets per node. |

## 5. LangGraph design essentials

```
State (TypedDict/Pydantic):
  question, chat_history, rewritten_query, expanded_queries[],
  retrieved_docs[], reranked_docs[], context, answer,
  grade, retry_count, trace_id, user_acl

Nodes:  guard_in → route → rewrite → expand → retrieve(parallel) → fuse
        → rerank → build_context → grade → generate → verify → respond

Conditional edges:
  route   → {direct_llm | rag | tools | refuse}
  grade   → {generate | rewrite (retry_count < 2) | fallback}
  verify  → {respond | generate (once) | refuse}

Cross-cutting: checkpointer (resume/audit), per-node timeouts,
               retries with backoff, human-in-the-loop for low confidence
```

Use LangChain for the components inside nodes and LangGraph for control flow. Keep nodes small, pure and independently testable.

## 6. Best practices checklist

1. **Measure before adding complexity.** Build a golden set of 100–300 real queries first.
2. **Hybrid + rerank is the highest-ROI baseline.** It usually beats HyDE and fine-tuning per unit of effort.
3. **Retrieval and generation are evaluated separately.** Most "hallucinations" are retrieval failures.
4. **Fine-tune embeddings only after** data quality, chunking and hybrid search are solid, and only with 1k+ quality pairs.
5. **Treat the index as a versioned artifact.** Embedding model, chunking strategy and schema are all part of the version.
6. **Enforce ACLs at retrieval time**, not in the prompt.
7. **Bound every loop** (retries, query count, token budget).
8. **Use small models for control tasks** (route, rewrite, grade) and reserve large models for answers.
9. **Always return citations** and a "can't answer" path.
10. **Close the feedback loop** from production traces back to training data and evaluation sets.

## 7. Suggested rollout order

| Phase | Deliver |
|---|---|
| **1** | Ingestion, structure-aware chunking, Azure AI Search hybrid, basic LangChain RAG, tracing, golden eval set |
| **2** | Semantic ranker or cross-encoder rerank, history-aware rewriting, ACL trimming, guardrails |
| **3** | LangGraph orchestration: router, multi-query, relevance grader, corrective loop, groundedness check |
| **4** | Azure ML fine-tuning loop for embedder and reranker, blue/green model deployment, reindex automation |
| **5** | Semantic caching, selective HyDE, decomposition for multi-hop, continuous eval and cost optimization |

If you share your data types (PDFs, tables, code, multilingual), scale and latency target, I can tailor the chunking, model choices and graph topology, or turn this into a deployable LangGraph skeleton.
