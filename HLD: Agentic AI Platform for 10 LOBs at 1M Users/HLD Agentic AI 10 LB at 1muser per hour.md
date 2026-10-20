# HLD: Agentic AI Platform for 10 LOBs at 1M Users/Hour (Text Diagram)

## 1. End-to-End Request Flow (Top Level)

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                   USERS (1M/hr, promo spikes)                            │
│                          Web | Mobile | Chat | Voice | Partner Apps                      │
└───────────────────────────────────────────┬──────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼──────────────────────────────────────────────┐
│ [L1] EDGE & CHANNEL LAYER                                                                │
│  CDN / WAF / DDoS  ──►  Global Load Balancer  ──►  Channel API Gateway                   │
│  Patterns: GATEWAY | RATE LIMITING | IAM (OIDC/JWT) | BFF (per channel)                  │
└───────────────────────────────────────────┬──────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────▼──────────────────────────────────────────────┐
│ [L2] FAST PATH & TRIAGE (hot path, <300 ms)                                              │
│  ┌──────────────┐  ┌───────────────────┐  ┌─────────────────┐  ┌───────────────────┐     │
│  │ Input        │─►│ Intent Router     │─►│ Semantic Cache  │─►│ Tier 0 Resolver   │     │
│  │ Guardrails   │  │ (rules/small LLM) │  │ (Redis+vector)  │  │ (API/template)    │     │
│  └──────────────┘  └─────────┬─────────┘  └─────────────────┘  └───────────────────┘     │
│  Patterns: ROUTING | CACHE-ASIDE | GUARDRAIL | LOAD SHEDDING | PRIORITY QUEUE            │
└────────────────────────────────────────────┬─────────────────────────────────────────────┘
                              (cache miss / needs reasoning)
┌────────────────────────────────────────────▼─────────────────────────────────────────────┐
│ [L3] AGENT GATEWAY & ENTERPRISE SUPERVISOR                                               │
│  Enterprise Supervisor Agent ──► Agent Registry (discovery, risk tier, owner, version)   │
│  Patterns: ROUTING/HANDOFF | SERVICE DISCOVERY | CIRCUIT BREAKER | BULKHEAD (per LOB)    │
└───┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬──────┘
    │         │         │         │         │         │         │         │         │
┌───▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐  ...  (LOB-3 to LOB-9)  ...                ┌▼──────┐
│ LOB-1 │ │ LOB-2 │ │ LOB-3 │ │ LOB-4 │                                            │LOB-10 │
│ Pool  │ │ Pool  │ │ Pool  │ │ Pool  │                                            │ Pool  │
└───┬───┘ └───────┘ └───────┘ └───────┘                                            └───────┘
    │   (each LOB = isolated namespace, quota, node pool, queue)
    ▼
```

## 2. Inside One LOB Pool (Bounded Context)

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ LOB-n AGENT POOL (K8s namespace, own quota / HPA / KEDA)       Pattern: DDD BOUNDED CONTEXT│
│                                                                                          │
│   ┌───────────────────────┐                                                              │
│   │ LOB Supervisor Agent  │  Pattern: ORCHESTRATOR-WORKERS                               │
│   └───────────┬───────────┘                                                              │
│       ┌───────┴───────────┬──────────────────┬───────────────────┐                       │
│  ┌────▼─────┐      ┌──────▼──────┐    ┌──────▼──────┐     ┌──────▼──────────┐            │
│  │Specialist│      │ Specialist  │    │ Critic /    │     │ Shared Capability│           │
│  │ Agent A  │      │ Agent B     │    │ Verifier    │     │ Agents (KYC, Doc │           │
│  │(pipeline)│      │(parallel)   │    │(EVALUATOR-  │     │ extract, notify) │           │
│  └────┬─────┘      └──────┬──────┘    │ OPTIMIZER)  │     └──────┬──────────┘            │
│       │                   │           └──────┬──────┘            │                       │
│  ┌────▼───────────────────▼──────────────────▼───────────────────▼────┐                  │
│  │ SIDECAR PROXY (per pod): guardrails | tracing | policy | identity   │                  │
│  │ Pattern: SIDECAR | AGENT-AS-A-SERVICE | STATELESS PODS              │                  │
│  └──────────────────────────────┬──────────────────────────────────────┘                  │
│                                 │                                                         │
│  ┌──────────────────────────────▼──────────────────────────────────────┐                  │
│  │ HUMAN-IN-THE-LOOP GATE (High-risk: approvals, dual control)         │                  │
│  └─────────────────────────────────────────────────────────────────────┘                  │
└───────────────────────┬──────────────────────────────────────┬───────────────────────────┘
                        │ LLM calls                            │ Tool calls
                        ▼                                      ▼
```

## 3. Model Gateway & Inference Layer

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ [L4] MODEL GATEWAY (horizontally scaled)                                                 │
│  Token-aware RATE LIMITING | Per-LOB QUOTA | PRIORITY QUEUE | PREFIX CACHE | SEMANTIC    │
│  CACHE | Output Guardrails | Cost metering (FinOps)                                       │
│  Patterns: GATEWAY | CIRCUIT BREAKER | RETRY+JITTER | HEDGED REQUESTS | FALLBACK LADDER  │
│                                                                                          │
│       ┌───────────────┬───────────────────┬──────────────────┬───────────────────┐       │
│       ▼               ▼                   ▼                  ▼                   │       │
│  ┌─────────┐    ┌───────────┐      ┌─────────────┐    ┌───────────────┐           │       │
│  │ Tier 1  │    │ Tier 2    │      │ Tier 3      │    │ Secondary     │◄──FAILOVER┘       │
│  │ Small   │    │ Mid model │      │ Large/      │    │ Provider /    │                   │
│  │ (self-  │    │ (reserved │      │ Reasoning   │    │ Region        │                   │
│  │ hosted  │    │ capacity) │      │ (<10% traffic)│  │               │                   │
│  │ GPU/K8s)│    └───────────┘      └─────────────┘    └───────────────┘                   │
│  └─────────┘     Fallback: Large → Mid → Small → Cache → Deterministic → Human            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## 4. Tool, Integration & Core Services Layer

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ [L5] TOOL / MCP GATEWAY                                                                  │
│  Task-level tools (BFF style) | Tool allow-list | Authz (delegated user context)         │
│  Patterns: ANTI-CORRUPTION LAYER | IDEMPOTENCY KEYS | RATE LIMIT per agent identity      │
│            | BULKHEAD (agent traffic vs customer traffic) | PARALLEL FAN-OUT/FAN-IN      │
└───────────────┬───────────────────────────────────────────┬──────────────────────────────┘
                │ READS (Query side)                        │ WRITES (Command side)
                ▼                                           ▼
┌──────────────────────────────────┐        ┌───────────────────────────────────────────┐
│ CQRS READ MODELS                 │        │ COMMAND HANDLERS (validated, authorized)   │
│  Read replicas | Redis cache     │        │   │                                        │
│  Vector DB (sharded, hybrid)     │        │   ▼                                        │
│  RAG indexes (event-fed)         │        │ SAGA / WORKFLOW ENGINE (compensation)      │
│  Pattern: CQRS | CACHE-ASIDE     │        │   │  (agents trigger, never own the txn)   │
└──────────────────────────────────┘        │   ▼                                        │
                                            │ OUTBOX ──► Kafka (EVENT-DRIVEN)            │
                                            └───────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ [L6] CORE SERVICES ON KUBERNETES (per LOB, DDD bounded contexts)                         │
│  Microservices | Service Mesh (mTLS) | Strangler facade over legacy | Systems of Record   │
│  Patterns: DDD | STRANGLER | SERVICE MESH | CIRCUIT BREAKER | DATABASE-PER-SERVICE       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5. Async & State Backbone

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ [L7] STATE & EVENT BACKBONE                                                              │
│  Redis Cluster (session/memory, TTL)    Durable Workflow Engine (long-running agents)    │
│  Kafka (events, KEDA triggers)          EVENT LOG: agent decision log (prompt, tools,    │
│  Async task queue + push/webhook         outputs, approvals) ──► audit & replay          │
│  Patterns: EVENT-DRIVEN | EVENT LOG/SOURCING | OUTBOX | ASYNC REQUEST-REPLY | STATELESS  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## 6. Cross-Cutting Platform Layer (EA-owned "Paved Road")

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ [L8] PLATFORM & GOVERNANCE (applies to all layers above)                                 │
│                                                                                          │
│ ┌─────────────┐ ┌──────────────┐ ┌───────────────┐ ┌──────────────┐ ┌─────────────────┐  │
│ │ IAM / Non-  │ │ Observability│ │ Evaluation &  │ │ AgentOps     │ │ FinOps          │  │
│ │ human ID,   │ │ Traces, TTFT,│ │ Red-team gate │ │ CI/CD, canary│ │ Cost per conv/  │  │
│ │ secrets,    │ │ tokens/sec,  │ │ Golden sets   │ │ rollback,    │ │ LOB chargeback  │  │
│ │ policy (OPA)│ │ cache hit    │ │ Drift monitor │ │ versioning   │ │ budget alerts   │  │
│ └─────────────┘ └──────────────┘ └───────────────┘ └──────────────┘ └─────────────────┘  │
│ ┌─────────────┐ ┌──────────────┐ ┌───────────────┐ ┌──────────────┐                      │
│ │ Agent       │ │ Memory/Data  │ │ Risk Tiering &│ │ Architecture │                      │
│ │ Registry    │ │ Governance   │ │ Control Matrix│ │ Review Board │                      │
│ └─────────────┘ └──────────────┘ └───────────────┘ └──────────────┘                      │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## 7. Kubernetes Deployment Topology

```
┌──────────────────────────── Region A (Active) ─────────────────── Region B (Active/Standby) ─┐
│  Zone 1 | Zone 2 | Zone 3  (topology spread, PDBs)                  (mirrored, failover)      │
│                                                                                              │
│  Node Pool 1: Gateways (Edge, Agent, Model, MCP)     HPA: RPS / concurrency                  │
│  Node Pool 2: Agent Runtime (per-LOB namespaces)     HPA + KEDA: queue depth, Kafka lag,     │
│                                                       tokens/sec, SCHEDULED PRE-SCALE (promo)│
│  Node Pool 3: GPU (Tier-1 self-hosted inference)     Scale on queue depth, continuous batch  │
│  Node Pool 4: Core Services (LOB microservices)      HPA (existing)                          │
│  Node Pool 5: Data (Redis, Vector DB, Kafka)         StatefulSets / managed services         │
│                                                                                              │
│  Isolation: namespace + ResourceQuota + NetworkPolicy per LOB  (BULKHEAD)                    │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 8. Pattern-to-Block Index

| Block | Patterns Applied |
|---|---|
| L1 Edge | Gateway, Rate Limiting, IAM, BFF |
| L2 Fast Path | Routing, Cache-Aside, Guardrail, Load Shedding, Priority Queue |
| L3 Supervisor | Routing/Handoff, Registry/Discovery, Circuit Breaker, Bulkhead |
| LOB Pool | DDD Bounded Context, Orchestrator-Workers, Pipeline, Parallelization, Evaluator-Optimizer, Sidecar, Stateless, HITL |
| L4 Model Gateway | Gateway, Token Rate Limit, Circuit Breaker, Retry+Jitter, Hedging, Fallback Ladder, Model Routing |
| L5 Tool Gateway | Anti-Corruption Layer, Idempotency, BFF, Bulkhead, Fan-out/Fan-in |
| Read/Write paths | CQRS, Saga, Outbox, Event-Driven |
| L6 Core | DDD, Strangler, Service Mesh, DB-per-Service |
| L7 Backbone | Event Log/Sourcing, Async Request-Reply, Durable Workflow |
| L8 Platform | Non-human IAM, Observability, Evaluation Gate, AgentOps, FinOps, Governance |

## 9. Latency Budget (Streamed Path)

```
Edge/Gateway 50ms → Guardrail+Router 150ms → Cache lookup 50ms → Retrieval 150ms
→ Model Gateway queue 50ms → LLM first token 800ms ≈ TTFT ~1.2-1.5s (then streaming)
Tier 0 path (cache/API): < 300 ms end-to-end
```

If you want, I can convert this into a Mermaid or draw.io-ready version, or add a sequence diagram for one specific journey (e.g., a promotion purchase flow with Saga and human approval).
