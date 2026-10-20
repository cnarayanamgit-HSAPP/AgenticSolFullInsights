# End-to-end workflow: coding agent building a high-scale Agentic AI platform (10 LOBs, 1M users/hour)

This builds on the Spring Boot backend flow from the previous answer. The core services (DDD, CQRS, Saga, Kafka, Resilience4j, Istio, Mongo, Entra ID, Prometheus/Grafana/Azure Monitor) stay the same, so only the **deltas for agentic AI and scale** are expanded here.

Pattern tags:
- **[A#]** how the coding agent itself works (same catalog as before)
- **[G#]** agentic AI runtime patterns (inside the product being built)
- **[S#]** scale and capacity patterns
- **[D#] [E#] [R#] [P#] [O#] [X#]** core-service patterns (domain, event-driven, resilience, platform, observability, security), as in the previous answer

## 0) Sizing: the agent must do this before designing

All numbers below are assumptions to validate with the business.

```
1M users/hour           ≈ 278 new users/sec
x 3 turns per user      ≈ 830 turns/sec average, ~2,500 turns/sec at 3x peak
x 1.5 LLM calls/turn    ≈ 1,250 LLM calls/sec average
x ~3k tokens/call       ≈ ~3.7M tokens/sec ≈ ~225M tokens/min (before optimization)
```

- **The LLM, not the Java services, is the bottleneck.** Quota (TPM/RPM), latency and cost drive the architecture.
- Levers the agent must design in: semantic cache, deterministic fast paths that skip the LLM, small-model routing, prompt caching, batching for async work, multi-region and multi-deployment capacity (provisioned throughput plus pay-as-you-go spillover), and per-LOB budgets.
- Output of this step is a `capacity-model.md` plus an ADR, approved at Gate 1.

## 1) Master workflow

```
┌──────────────────────────────────────────────────────────────────┐
│ START: Human goal / GitHub Issue                                 │
│ "Agentic AI platform for 10 LOBs, 1M users/hour, multi-region,   │
│  core services on AKS+Istio, Kafka, Mongo CQRS, Entra ID"        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 1. CONTEXT LOAD                                                  │
│ [A10] RAG repo map, golden templates (service, agent, tool, IaC) │
│ [A11] Memory: AGENTS.md, .github/copilot-instructions.md         │
│ [A13] Sandbox, least privilege, no prod data, budget caps        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. DISCOVERY (per LOB, parallel interviews)   [A12][A9]          │
│ [D1] Event Storming  [D2] Bounded contexts + Context Map         │
│ Per LOB capture: use cases, agent goals, tools/APIs, data        │
│ sources, autonomy level, HITL points, compliance, SLOs, budget   │
│ Classify each use case:                                          │
│   deterministic workflow | RAG Q&A | single agent | multi-agent  │
│   (use the simplest that works; agents only where needed)        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. SPEC + ARCHITECTURE + CONTRACTS (Planner agent)  [A14]        │
│ Capacity model [S1]  Tenancy model [S2]  Region topology [S3]    │
│ Contracts first: openapi.yaml, AsyncAPI + Schema Registry,       │
│ MCP tool schemas [G4], agent cards/A2A [G7], Mongo collections,  │
│ Saga definitions, eval datasets + pass thresholds [G12]          │
│ ADRs: cell architecture, LLM gateway, memory, orchestration      │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
              ◇ GATE 1: human approves spec/ADRs/capacity/cost? ◇ [A12]
                             │ Yes
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. TASK PLAN (tasks.md → GitHub Issues)                          │
│ [A4] Plan-and-Execute [A2] Routing [A3] Orchestrator-Workers     │
│ Order: contracts → platform+cells → core services → LLM gateway  │
│ → agent runtime → LOB agents → security → observability →        │
│ evals → load/chaos                                               │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼ [A9] Parallelization
 ┌─────────┬──────────┬───────────┬───────────┬───────────┬────────────┐
 ▼         ▼          ▼           ▼           ▼           ▼            ▼
5A Core   5B Platform 5C LLM      5D Agent    5E LOB      5F Security  5G Observ.
services  cells/Istio gateway +   runtime +   agents      + guardrails + evals
(DDD,     /AKS/IaC    scale       memory/RAG  (x10)       (Entra)
CQRS,Saga)            layer
 └─────────┴──────────┴───────────┴───────────┴───────────┴────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. VERIFY: unit, Testcontainers, contract, saga, AGENT EVALS,    │
│ red-team, load (1M/hr), chaos, cost tests                        │
│ [A8] Evaluator-Optimizer  [G12] Eval gates in CI                 │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
        ◇ All green (incl. eval thresholds + SLO + cost)? ◇
          No → route to owning agent → fix → re-verify [A2][A5]
                             │ Yes
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. REVIEW [A7] reviewer agent + Copilot code review + CodeQL     │
│ [A9] parallel reviewers: security, architecture, AI safety, cost │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
              ◇ GATE 2: human approves PR (+ AI risk review)? ◇ [A12]
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 8. CI/CD: GitHub Actions → build, test, SAST/SCA, evals, image,  │
│ sign/SBOM → ACR → [P6] GitOps (Argo CD/Flux) per cell/region     │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
              ◇ GATE 3: prod approval (per LOB, per region)? ◇ [A12]
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 9. PROGRESSIVE ROLLOUT                                           │
│ [P5] Istio canary + feature flags per LOB + shadow traffic       │
│ [G13] Prompt/model canary (A/B) with online evals                │
│ [R7] Auto rollback on SLO / quality / cost burn                  │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 10. FEEDBACK LOOP [O6]                                           │
│ traces, thumbs-down, escalations, drift → eval set, prompts,     │
│ backlog, AGENTS.md                                               │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                            END
```

## 2) Target runtime architecture (what the agent builds)

```
Users (web/mobile/API) → Azure Front Door/WAF (global, geo-routing)
        ▼
 Regional CELL (repeat N cells per region; each cell is self-contained) [S3]
 ┌───────────────────────────────────────────────────────────────────┐
 │ Istio Ingress Gateway (TLS, Entra JWT, rate limit per LOB/tenant) │
 │        ▼                                                          │
 │ Conversation/Session Service (stateless, HPA, WebSocket/SSE)      │
 │        ▼                                                          │
 │ ┌─ Agent Gateway / Router [G1] ───────────────────────────────┐   │
 │ │ intent → deterministic flow | RAG | single agent | multi-agent│  │
 │ └───────────┬─────────────────────────────────────────────────┘   │
 │             ▼                                                     │
 │ Agent Runtime Workers (Spring Boot / Spring AI or LangGraph4j)    │
 │   Supervisor [G2] → LOB Agents [G3] → Tools via MCP [G4]          │
 │   Durable execution state [G6], memory [G8], guardrails [G10]     │
 │             ▼                                                     │
 │ LLM Gateway [S4]: auth, quota/budget per LOB, semantic cache [S5],│
 │  model router [S6], failover [R1], token metering, PII redaction  │
 │             ▼                                                     │
 │   Azure OpenAI / Foundry deployments (PTU + PAYG, multi-region)   │
 │                                                                   │
 │ Core services (DDD, CQRS, Saga) ◄──► Kafka (events, commands)     │
 │   Order/Case/Claim/Account… per LOB bounded contexts              │
 │ Mongo (write + read models, vector index) | Redis (cache, sessions)│
 │ AI Search / Mongo vector search (RAG)   | Blob (docs)             │
 └───────────────────────────────────────────────────────────────────┘
 Control plane: Prompt/Agent registry, eval service, policy service,
 cost/FinOps, LOB onboarding, Grafana + Azure Monitor + Prometheus
```

## 3) Flow 5A: Core services (delta only)

```
Same flow as the previous answer:
 [D1-D16] DDD, hexagonal, CQRS + Mongo projections
 [E1-E12] Kafka, Outbox, Saga, idempotent consumers, retry topics, DLQ
 [R1-R8] Resilience4j, one retry layer, idempotency keys
 [P1-P8] AKS, Istio sidecar, probes, GitOps
 [X1-X6] Entra ID, Key Vault, defense in depth

Additions for 10 LOBs and 1M users/hour:
 [S2]  Bounded context per LOB; shared kernel kept tiny (identity, audit, billing)
 [S7]  Kafka sizing: partitions = target consumer parallelism x headroom;
       key = aggregateId; separate topics per LOB; quotas per LOB; tiered storage
 [S8]  Mongo: shard key chosen from the access pattern (e.g., tenantId+aggregateId);
       read replicas for queries; per-LOB collections or databases; TTL on session data
 [S9]  Agent actions that change state are COMMANDS through core services,
       never direct DB writes by the agent  (agent = untrusted client)
 [E3]  Long-running agent workflows reuse the Saga pattern with compensations
```

## 4) Flow 5C: LLM gateway and scale layer (the critical path)

```
[Task: "LLM Gateway"]
   ▼
[A14] TDD + load-test harness first (mock LLM with realistic latency distribution)
   ▼
Implement:
 [S4]  Gateway pattern: one internal OpenAI-compatible API in front of all models
       (swap providers/models without touching agents)
 [S5]  Semantic cache: embed query → Redis/vector lookup → hit returns cached answer
       (per-LOB, per-tenant isolation; TTL; never cache personalized/PII answers)
 [S6]  Model router: cheap/small model for classification, extraction, routing;
       large model only for hard reasoning (route by task type + confidence)
 [S10] Admission control + backpressure: priority queues per LOB,
       load shedding (429 + Retry-After), degrade gracefully
 [S11] Token-aware rate limiter and budget enforcement per LOB/tenant (TPM, daily $)
 [S12] Capacity pooling: PTU deployments for baseline, PAYG for burst,
       multi-region spillover; health-based endpoint selection
 [R1-R6] Circuit breaker per deployment, retry with jitter on 429/5xx (honor Retry-After),
         timeout (streaming first-token timeout + total), bulkhead per LOB,
         fallback chain: primary model → secondary model → smaller model → cached/static reply
 [S13] Streaming (SSE) to cut perceived latency; prompt caching for static prefixes
 [S14] Async/batch lane via Kafka for non-interactive work (summaries, back-office)
 [O1]  Meter tokens, cost, latency, cache hit rate, per LOB/agent/model
   ▼
Tests: load at 3x peak with LLM stub; 429 storms; region failover; budget exhaustion
   ▼
◇ Green? ◇ ─ No → [A5] fix loop → [A12] escalate
```

## 5) Flow 5D: Agent runtime

```
[Task: "Agent runtime"]
   ▼
[A10] retrieve golden agent template   [A14] write failing tests + eval cases first
   ▼
Implement agentic patterns:
 [G1]  Router / intent classifier: deterministic where possible (rules, small model)
 [G2]  Supervisor (orchestrator-workers): decomposes the goal, delegates to LOB agents
 [G3]  Specialist agents per LOB with narrow tools and prompts
        (fewer tools per agent = higher accuracy, lower cost)
 [G4]  Tool use via MCP servers: typed schemas, timeouts, idempotency keys,
        read-only vs write tools separated
 [G5]  ReAct loop with hard limits: max steps, max tokens, max cost, wall-clock timeout
 [G5b] Plan-and-Execute for multi-step tasks; reflection only where evals show a gain
 [G6]  Durable execution: agent state persisted per step (Mongo/Redis) so a pod crash
        resumes mid-task; long waits (human approval) are suspended, not held in memory
 [G7]  Agent-to-agent via A2A / Kafka commands; agent cards describe capabilities
 [G8]  Memory tiers: working (context window, summarized/compacted) |
        session (Redis) | long-term user/LOB facts (Mongo + vector, with consent + TTL)
 [G9]  RAG: chunking, hybrid search (BM25 + vector), reranking, per-LOB indexes,
        ACL-aware retrieval (filter by the caller's entitlements, not the agent's)
 [G10] Guardrails: input (prompt-injection, jailbreak, PII), output (groundedness,
        toxicity, policy), tool-call allow-list and argument validation
 [G11] Human-in-the-loop: approval step for high-risk actions
        (payments, deletions, external email); implemented as a Saga wait state
 [G14] Idempotent, compensatable tool actions: every write tool has an undo
   ▼
Stateless workers + externalized state → scale horizontally via HPA + KEDA
(Kafka lag, queue depth, active sessions), not CPU only [S15]
   ▼
Tests: loop-limit enforcement, crash/resume, tool failure, injection attempts,
       duplicate tool call, permission boundary
   ▼
◇ Green? ◇ ─ No → [A5] fix loop
```

## 6) Flow 5E: Onboarding each LOB (repeatable factory, x10)

```
[Template: LOB agent pack]  [P2] Factory
   ▼
For each LOB (parallelizable, [A9]):
  1. Domain model + events from Event Storming          [D1-D3]
  2. Tool catalog: wrap LOB APIs as MCP tools behind an ACL [D12]
  3. Knowledge sources → ingestion pipeline → LOB index  [G9]
  4. Agent definition (prompts, tools, guardrails, autonomy level, HITL rules)
  5. Eval suite: golden Q&A, tool-use trajectories, safety cases,
     with a per-LOB pass threshold
  6. Quotas: TPM, budget, rate limits, Kafka topics, Mongo namespaces
  7. Feature flag + canary plan + runbook + dashboards-as-code
   ▼
Registry entry (agent/prompt versions, owners, risk tier)  [G13]
   ▼
LOB-specific gate: LOB owner + risk/compliance sign-off, then staged rollout
```

## 7) Flow 5F: Security and responsible AI

```
 [X1-X3] Entra ID JWT at Istio and Spring; defense in depth
 [X4]  Agents act ON BEHALF OF the user (OBO/token exchange) → downstream authz
       sees the real user; service identities use Workload Identity
 [X7]  Least-privilege per agent: separate identity, scoped tools, per-LOB isolation
 [X5]  Secrets in Key Vault; none in prompts or logs
 [X6]  PII redaction before the LLM and in logs; data residency per region;
       no training on customer data; retention and consent for memory
 [G10] Prompt-injection defense: treat retrieved and tool content as untrusted data,
       separate instructions from data, validate tool arguments, deny-by-default egress
 [X8]  Audit trail: every prompt version, tool call, approval and decision,
       linked by traceId + correlationId (immutable store)
 [X9]  Azure AI Content Safety / policy filters; red-team suite in CI
 Tests: injection corpus, cross-tenant leakage tests, OBO permission tests
```

## 8) Flow 5G: Observability, evals and alerting

```
 Telemetry [O1]: OpenTelemetry traces across HTTP, Kafka, agent steps, tool calls, LLM calls
   (use GenAI semantic conventions: model, tokens in/out, latency, finish reason)
 Metrics (Prometheus via Micrometer): standard RED + 
   llm_tokens_total{lob,model}, llm_cost_usd, cache_hit_ratio, agent_steps_per_task,
   tool_error_rate, handoff_rate_to_human, guardrail_block_rate, saga_compensation_rate
 Storage/visualization: Azure Monitor managed Prometheus + Managed Grafana;
   Application Insights/Log Analytics (KQL) for traces and logs
 Dashboards as code [O3]: per-LOB view, per-cell view, cost/FinOps, quality, SLO burn
 Alerts [O4] (Grafana + Azure Monitor action groups, each linked to a runbook):
   latency/5xx burn rate, 429 rate from LLM, breaker open, consumer lag,
   DLQ depth, budget at 80%/100%, cache hit drop, quality score drop,
   guardrail spike (possible attack), agent loop-limit hits
 Online evals [G12]: sample production traces → LLM-judge + rules → quality score
   per LOB/agent/prompt version; offline evals gate every PR
 Alert on QUALITY and COST, not only uptime
```

## 9) Flow: Load, chaos and cost verification

```
 Load (k6/Gatling, distributed): ramp to 1M users/hr and 3x spike, per-LOB mix,
   realistic LLM latency model; measure p50/p95/p99, queue depth, shed rate,
   consumer lag, cache hit rate, cost per 1k conversations
 Soak test: 4-8 hours (memory leaks, connection pools, Kafka rebalances)
 Chaos (staging, Istio fault injection + AKS): kill pods, zone loss, region failover,
   LLM 429/5xx storms, slow tokens, Kafka broker loss, Mongo primary election,
   Redis loss (agents must degrade, not die)
 Cost test: per-LOB budget enforcement trips correctly
 Pass criteria come from the spec (SLOs + quality threshold + $/conversation)
   ▼
◇ Pass? ◇ ─ No → [A8] feed back to the owning agent → fix → re-run
```

## 10) Flow 8: GitHub Copilot and the agents in delivery

```
 Issue per task (acceptance criteria + eval cases)
   ▼
 Copilot coding agent or orchestrated agents [A3]
   (reads .github/copilot-instructions.md, AGENTS.md; works in sandbox only)
   ▼
 PR → GitHub Actions:
   build/unit → Testcontainers (Kafka, Mongo replica set, Redis) → Pact (HTTP + message)
   → ArchUnit/Spotless → Sonar → CodeQL, dependency review, secret scan, Trivy
   → AGENT EVALS (accuracy, tool-use, safety, cost-per-task regression gates)
   → image + SBOM + cosign → ACR
   ▼
 Copilot code review + reviewer agent [A7] + CODEOWNERS human review [Gate 2]
   ▼
 GitOps promote: dev → test → staging (load/chaos/evals) → prod cells [Gate 3]
   cell by cell, region by region, with auto rollback [R7]
```

## 11) Runtime example: one user turn

```
 1 User → Front Door → Istio Gateway (JWT, per-LOB rate limit)
 2 Session Service loads session (Redis), streams via SSE
 3 Agent Gateway classifies intent (small model or rules) [G1]
 4a FAQ hit → semantic cache returns answer, no LLM large call [S5]
 4b Simple RAG → retrieve (ACL-filtered) → small/mid model → guardrails → reply
 4c Task → Supervisor plans [G2] → LOB agent [G3] calls MCP tools [G4]
     → read tools directly; write tools = COMMAND to core service
 5 Core service: aggregate validates → Mongo TX (state + outbox) → Kafka [E2]
 6 Saga runs (e.g., reserve → pay → confirm; compensate on failure) [E3]
 7 High-risk step → HITL approval, workflow suspends (durable) [G11]
 8 Projection updates read model; agent/UI sees the result (eventual consistency) [D14]
 9 Output guardrails → stream answer → meter tokens/cost → trace complete
 * Failure at any hop: breaker → fallback model → cached/static reply → human handoff
```

## 12) Pattern catalog (new for this scope)

### G) Agentic AI runtime patterns
| ID | Pattern | Use |
|---|---|---|
| G1 | Intent router, deterministic-first | Avoid unnecessary LLM calls |
| G2 | Supervisor / orchestrator-workers | Multi-agent coordination |
| G3 | Specialist agent per LOB, narrow tools | Accuracy and isolation |
| G4 | Tool use via MCP, typed and idempotent | Safe action execution |
| G5 | ReAct and Plan-and-Execute with hard limits | Bounded loops |
| G6 | Durable execution / checkpointed state | Crash resume, long waits |
| G7 | Agent-to-agent (A2A, Kafka commands) | Cross-LOB collaboration |
| G8 | Tiered memory | Context control, personalization |
| G9 | RAG: hybrid search, rerank, ACL-aware | Grounded answers |
| G10 | Guardrails: input, output, tool allow-list | Safety, injection defense |
| G11 | Human-in-the-loop as a Saga wait state | High-risk approvals |
| G12 | Offline and online evals as CI/CD gates | Quality control |
| G13 | Prompt/agent registry, versioning, A/B canary | Safe change management |
| G14 | Compensatable tool actions | Undo for agent mistakes |

### S) Scale and capacity patterns
| ID | Pattern | Use |
|---|---|---|
| S1 | Capacity model first | Size TPM, partitions, pods, cost |
| S2 | Multi-tenancy by LOB, bounded contexts | Isolation, quotas |
| S3 | Cell-based architecture, multi-region | Blast-radius control |
| S4 | LLM gateway | Central quota, routing, metering |
| S5 | Semantic cache | Cut cost and latency |
| S6 | Model routing/cascade | Right-size model per task |
| S7 | Kafka partitioning and quotas | Throughput, ordering |
| S8 | Mongo sharding, read replicas | Data scale |
| S9 | Agent as untrusted client, commands via core services | Integrity |
| S10 | Admission control, load shedding, backpressure | Survive spikes |
| S11 | Token-aware rate limits and budgets | Cost control |
| S12 | PTU baseline + PAYG burst + multi-region spillover | LLM capacity |
| S13 | Streaming, prompt caching | Latency |
| S14 | Async/batch lane | Offload non-interactive work |
| S15 | Event-driven autoscaling (KEDA) | Scale on real load signals |

Reuse **[A1-A15]** from earlier, plus **[D], [E], [R], [P], [O], [X]** for core services. Add **[X7-X9]** for agent identity, audit and content safety.

## Selection rules for the agent

1. **Simplest thing that works**: deterministic flow, then RAG, then single agent, then multi-agent. Each upgrade needs an ADR and an eval showing the gain.
2. **Capacity model before architecture.** If the LLM quota and budget do not fit, redesign (cache, routing, async) before writing code.
3. **Agents never write to databases directly.** They issue commands through core services, which apply DDD invariants, outbox and saga.
4. **Every loop has hard limits** on steps, tokens, cost and time.
5. **Evals are the test suite for agent behavior.** No eval gate means no merge, and prompt or model changes are treated like code changes.
6. **One layer owns retries** (gateway, app or mesh), and honor `Retry-After` on 429s to avoid retry storms.
7. **Per-LOB isolation** of quotas, data, indexes, identities and kill switches, so one LOB cannot take down the others.
8. **Cell architecture** limits blast radius. Roll out cell by cell with auto rollback on SLO, quality or cost burn.
9. **Humans approve** spec, PR, production, and every high-risk agent action.
10. **Agent safety for the builder too**: Copilot and coding agents work only in dev/sandbox, and production changes go through GitOps.

If you share your Azure OpenAI/Foundry quota, the real per-LOB use cases and traffic mix, Kafka flavor (Confluent, Event Hubs, Strimzi), and Mongo hosting (Atlas, Cosmos DB for Mongo vCore), I can produce the capacity model, `AGENTS.md`, `.github/copilot-instructions.md`, the LOB agent-pack template and the ADR templates for this flow.
