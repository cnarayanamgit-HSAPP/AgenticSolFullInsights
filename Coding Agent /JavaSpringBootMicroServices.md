# End-to-end workflow with design patterns: coding agent building Spring Boot backend microservices

Pattern tags used inline:

- **[A#]** Agent patterns: how the coding agent works
- **[D#]** Domain and data patterns: DDD, CQRS, Mongo
- **[E#]** Event-driven and integration patterns: Kafka, Saga, Outbox
- **[R#]** Resilience patterns: Resilience4j and Istio
- **[P#]** Platform and delivery patterns: Kubernetes, Istio, CI/CD
- **[O#]** Observability patterns
- **[X#]** Security patterns: Entra ID

## Assumptions

- Java 21 and Spring Boot 3.x, with Maven or Gradle.
- Mongo is the **read side** (and optionally the event store). The write side may be Mongo too, but with separate collections.
- Kafka carries domain events. AKS runs the services, with Istio providing the sidecar mesh.
- GitHub Copilot is used in two ways. The **Copilot coding agent** takes issues and opens PRs, and **Copilot code review** plus `.github/copilot-instructions.md` enforce rules. Your own orchestrated agents can do the heavy lifting alongside it.

## 1) Master workflow

```
┌──────────────────────────────────────────────────────────────────┐
│ START: Human goal / GitHub Issue                                 │
│ "Build Order, Payment, Inventory services (DDD, Kafka, Saga,     │
│  CQRS+Mongo) on AKS with Istio, Entra ID, Grafana/Azure Monitor" │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 1. CONTEXT LOAD                                                  │
│ [A10] RAG repo map + golden service template                     │
│ [A11] Memory: AGENTS.md, .github/copilot-instructions.md         │
│ [A13] Sandbox, least privilege, no prod secrets                  │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. REQUIREMENTS + DOMAIN DISCOVERY                               │
│ [A12] Human-in-the-loop interview  [A1] Prompt chaining          │
│ [D1] Event Storming → [D2] Bounded Contexts + Context Map       │
│ [D3] Ubiquitous Language glossary                                │
│ NFRs: SLOs, throughput, RPO/RTO, compliance, cost                │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. SPEC + ARCHITECTURE + CONTRACTS (Planner agent)               │
│ [A14] Spec-driven: spec.md, ADRs                                 │
│ Contracts first:                                                 │
│   openapi.yaml (commands/queries)  [P1] API-first                │
│   AsyncAPI + Avro/JSON Schema (Kafka topics) [E1] Schema Registry│
│   Mongo collection design (write model vs read models)           │
│   Saga definition (steps + compensations)                        │
│ [D2] Context Map: ACL / Open Host / Published Language           │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                ◇ GATE 1: human approves spec + ADRs? ◇ [A12]
                │ No → revise        │ Yes
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. TASK PLAN (tasks.md → GitHub Issues)                          │
│ [A4] Plan-and-Execute  [A2] Routing  [A3] Orchestrator-Workers   │
│ Order: contracts → platform baseline → domain → persistence →    │
│        messaging → resilience → security → observability → E2E   │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼ [A9] Parallelization
   ┌────────────┬────────────┼─────────────┬───────────────┐
   ▼            ▼            ▼             ▼               ▼
┌────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌─────────────┐
│5A Domain│ │5B CQRS + │ │5C Kafka  │ │5D Platform│ │5E Security +│
│ + App   │ │ Mongo    │ │ Saga     │ │ K8s/Istio │ │Observability│
└───┬────┘ └────┬─────┘ └────┬─────┘ └─────┬─────┘ └──────┬──────┘
    └───────────┴────────────┼─────────────┴──────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. VERIFY: unit, slice, Testcontainers, contract, saga, chaos    │
│ [A8] Evaluator-Optimizer  [E8] Consumer-driven contracts         │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
        ◇ All green? ◇ ─ No → route defect to owner agent → fix → re-verify
                             │ Yes
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. REVIEW                                                        │
│ [A7] Reviewer agent + Copilot code review + CodeQL               │
│ [A9] Parallel reviewers: security, architecture, performance     │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                ◇ GATE 2: human approves PR? ◇ [A12]
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 8. CI/CD: GitHub Actions                                         │
│ build → test → SAST/SCA → image → sign/SBOM → push to ACR        │
│ [P6] GitOps (Argo CD / Flux) deploy to AKS  [A15] git tag        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                ◇ GATE 3: prod approval? ◇ [A12]
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 9. PROGRESSIVE ROLLOUT                                           │
│ [P5] Canary via Istio VirtualService weights 5→50→100%           │
│ [R7] Auto rollback on SLO burn-rate alert                        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 10. FEEDBACK LOOP                                                │
│ [O6] Traces/metrics/incidents → backlog, evals, AGENTS.md        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
                            END
```

## 2) Flow 5A: Domain and application layer (DDD, hexagonal)

```
[Task: "Order bounded context"]
   ▼
[A10] retrieve golden template + similar aggregates
   ▼
[P2] Factory: scaffold Spring Boot service (Spring Initializr/archetype)
   Modules:  domain | application | adapters-in | adapters-out | bootstrap
   ▼
[A14] TDD: write FAILING tests first
   - domain unit tests (pure Java, no Spring)
   - ArchUnit tests enforcing layer rules [D8]
   ▼
Implement tactical DDD:
   [D4] Aggregate Root (Order) with invariants, small aggregates, ID references only
   [D5] Value Objects (Money, Address), Entities
   [D6] Domain Events (OrderPlaced, OrderCancelled)
   [D7] Domain Services, Specifications, Factories
   [D8] Hexagonal / Ports and Adapters (domain has no Spring/Mongo/Kafka imports)
   [D9] Application Services = use cases; Command and Query handlers
   [D10] Repository interface in domain, implementation in adapter
   [P3] Dependency Injection (constructor injection)
   [D11] DTO + MapStruct mappers, no entity leakage via REST
   [D12] Anti-Corruption Layer for external or legacy models
   ▼
[A6] run: mvn verify (compile, tests, Checkstyle/Spotless, ArchUnit, SonarQube)
   ▼
◇ Green? ◇ ─ No → [A5] ReAct: read error → fix → retry (limit N → [A12] escalate)
   │ Yes
   ▼
[A7] self-review diff vs spec → commit [A15] → PR
```

## 3) Flow 5B: CQRS with Mongo collections

```
        WRITE SIDE (Command)                         READ SIDE (Query)
┌─────────────────────────────┐              ┌──────────────────────────────┐
│ POST /orders                │              │ GET /orders/{id}, /search    │
│   ▼                         │              │   ▼                          │
│ CommandHandler              │              │ QueryHandler (no domain logic)│
│   ▼                         │              │   ▼                          │
│ Aggregate.apply(cmd)        │              │ Mongo: order_summary_view    │
│   ▼  [D13] Optimistic lock  │              │        order_by_customer_view│
│ Mongo: orders (write model) │              │ (denormalized, per-query     │
│  and/or event_store         │              │  shape, indexed for queries) │
│   +  outbox collection  ────┼─ same Mongo ─┤                              │
│       (single transaction)  │  transaction │ [D14] Projections built by   │
└──────────────┬──────────────┘              │  Kafka consumers (idempotent)│
               ▼                             └──────────────▲───────────────┘
   [E2] Outbox relay (poller / Debezium / change streams)    │
               ▼                                             │
          Kafka topic: orders.events.v1 ─────────────────────┘
```

Agent rules for this flow:

- **[D13]** Version field on aggregates for optimistic concurrency (`@Version`).
- **[E2]** The outbox write and the aggregate write go in **one Mongo multi-document transaction**, which requires a replica set (or Cosmos DB for Mongo with transaction support).
- **[D14]** Projections are **idempotent** (upsert keyed by `eventId` and `aggregateVersion`), tolerate out-of-order events, and can be **rebuilt by replay**.
- **[D15]** Eventual consistency is explicit in the API: return `202` or a version token, and let the UI poll or subscribe.
- Mongo schema work: collection per read use case, compound indexes that match query filters and sort, TTL on the outbox and idempotency collections, schema validation (`$jsonSchema`), and migration scripts (Mongock).
- **[D16]** Event Sourcing is **optional**: use it only if the spec demands an audit trail or temporal queries. Otherwise use state-based write models plus outbox events.

## 4) Flow 5C: Kafka, Saga, Outbox (event-driven)

```
[Task: "Place Order saga"]
   ▼
[E1] Define topics + schemas first (Avro/JSON Schema + Schema Registry, compatibility=BACKWARD)
   Topic naming: <context>.<type>.<name>.v1   keys = aggregateId (ordering per aggregate)
   ▼
[E3] Choose Saga style (ADR):
   ┌─────────────────────────────┬──────────────────────────────────┐
   │ Choreography                │ Orchestration                    │
   │ few steps, simple flows     │ many steps, complex compensations│
   │ services react to events    │ Order Saga Orchestrator, state   │
   │                             │ machine persisted in Mongo       │
   └─────────────────────────────┴──────────────────────────────────┘
   ▼
Saga steps with compensations [E4]:
   OrderCreated → ReserveInventory → ProcessPayment → ConfirmOrder
        ✗ PaymentFailed → ReleaseInventory → CancelOrder  (compensating transactions)
   ▼
Producer side:
   [E2] Transactional Outbox   [E5] Idempotent producer (acks=all, enable.idempotence)
   [E6] Event envelope: eventId, type, version, correlationId, causationId, traceparent
   ▼
Consumer side:
   [E7] Idempotent Consumer (processed_events collection / unique index)
   [E9] Retry topics (orders.retry.5s / 1m / 10m) + DLQ  (Spring Kafka @RetryableTopic)
   [E10] Poison-pill handling: ErrorHandlingDeserializer + DLT + alert
   [E11] Consumer group scaling, concurrency = partitions, manual ack after commit
   [E12] Saga timeout + Process Manager watchdog for stuck sagas
   ▼
Tests:
   Testcontainers Kafka + Mongo; duplicate-delivery, out-of-order, crash-mid-saga,
   compensation path, DLQ replay; Pact message contracts [E8]
   ▼
◇ Green? ◇ ─ No → [A5] fix loop → [A12] escalate if stuck
```

## 5) Flow: Resilience (Resilience4j and Istio layering)

```
Agent decides WHERE each resilience concern lives (ADR):

┌──────────────────────────┬───────────────────────────────────────────┐
│ Istio / Envoy (mesh)     │ Resilience4j (application)                │
├──────────────────────────┼───────────────────────────────────────────┤
│ mTLS, traffic shifting   │ Circuit Breaker  [R1]                     │
│ connection pool limits   │ Retry + exp. backoff + jitter [R2]        │
│ outlier detection        │ Time Limiter (timeout) [R3]               │
│ coarse timeouts          │ Bulkhead (semaphore/thread pool) [R4]     │
│ rate limiting (ingress)  │ Rate Limiter [R5]                         │
│                          │ Fallback / graceful degradation [R6]      │
└──────────────────────────┴───────────────────────────────────────────┘
```

Rules the agent follows:

- **Retry in only one layer per call path.** Mesh retries multiplied by app retries causes retry storms. Default: the app owns retries for business-aware calls, and the mesh owns connection-level protection.
- Retry **only idempotent operations**, or operations carrying an **idempotency key** [R8].
- Annotation order in Resilience4j: `Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( fn )))))`. Verify it with a test.
- Config goes in `application.yml`, never hard-coded:

```
resilience4j:
  circuitbreaker.instances.paymentClient:
    slidingWindowSize: 50
    failureRateThreshold: 50
    waitDurationInOpenState: 30s
    permittedNumberOfCallsInHalfOpenState: 5
  retry.instances.paymentClient:
    maxAttempts: 3
    waitDuration: 200ms
    enableExponentialBackoff: true
    exponentialBackoffMultiplier: 2
  timelimiter.instances.paymentClient.timeoutDuration: 2s
  bulkhead.instances.paymentClient.maxConcurrentCalls: 20
```

- Export Resilience4j metrics through Micrometer: breaker state, failure rate, retry count.
- Tests: WireMock or Toxiproxy fault injection to confirm the breaker opens, the fallback fires, and the breaker recovers via half-open.
- Chaos in staging: Istio fault injection (delay and abort), pod kill, and Kafka broker loss **[R7]**.

## 6) Flow 5D: Platform (containers, Kubernetes, Istio sidecar)

```
[Task: "Make it cloud-native"]
   ▼
[P2] Dockerfile: multi-stage, distroless/JRE-slim, non-root, layered jar,
     Buildpacks or Jib; JVM flags container-aware (-XX:MaxRAMPercentage)
   ▼
K8s manifests / Helm / Kustomize  [P4] 12-factor config:
   Deployment (rolling, resources requests/limits, topologySpreadConstraints)
   Probes: /actuator/health/liveness, /readiness, startup  [P7] Health Probe
   HPA (CPU + Kafka consumer lag via KEDA)  PodDisruptionBudget
   ConfigMap + Secret (CSI Secrets Store → Azure Key Vault) 
   ServiceAccount + Workload Identity (no static credentials)
   NetworkPolicy (default deny)
   ▼
Istio [P8] Sidecar Proxy pattern (Envoy injected, or ambient mode if adopted):
   Namespace label istio-injection=enabled
   PeerAuthentication: STRICT mTLS                [X3]
   RequestAuthentication + AuthorizationPolicy: validate Entra ID JWT, per-route rules [X2]
   VirtualService: canary weights, timeouts, header routing
   DestinationRule: connection pool, outlier detection, subsets
   Gateway: ingress TLS
   ServiceEntry: external dependencies (Mongo, Kafka if outside the mesh)
   ▼
Graceful shutdown: server.shutdown=graceful, preStop sleep, drain Kafka consumers
   ▼
Validation (sandbox/kind or ephemeral AKS namespace):
   kubeconform, kube-linter, Checkov/Trivy config scan, istioctl analyze,
   deploy preview → smoke tests
   ▼
◇ Pass? ◇ ─ No → [A5] fix → retry
```

Note that Kafka and Mongo traffic is TCP, so sidecar routing rules are limited there. Handle TLS and authentication at the driver level, and keep Kafka and Mongo outside strict L7 policies.

## 7) Flow 5E-1: Security (Entra ID)

```
[Task: "AuthN/AuthZ"]
   ▼
[X1] Spring Security OAuth2 Resource Server (JWT)
     issuer = https://login.microsoftonline.com/<tenant>/v2.0
     validate: signature (JWKS), iss, aud, exp, scp/roles
   ▼
[X2] Map Entra App Roles / scopes → Spring authorities
     Method security: @PreAuthorize("hasAuthority('Orders.Write')")
   ▼
[X3] Defense in depth: Istio RequestAuthentication + AuthorizationPolicy at the edge,
     Spring Security inside the service (never trust only one layer)
   ▼
[X4] Service-to-service: Entra Workload Identity (federated credentials) or
     client-credentials; propagate user context via token exchange (On-Behalf-Of) when needed
   ▼
[X5] Secrets: Azure Key Vault + CSI driver, none in git, none in env files
   ▼
[X6] Data protection: PII classification, field-level encryption (Mongo CSFLE) where required,
     audit events for sensitive actions, log redaction
   ▼
Tests: Spring Security test slices (valid, expired, wrong-audience, missing-role tokens),
       mock Entra via WireMock/Keycloak for local, Pact auth headers
   ▼
Scans: OWASP Dependency-Check/Trivy, CodeQL, secret scanning, SBOM
```

## 8) Flow 5E-2: Observability (Prometheus, Grafana, Azure Monitor)

```
[Task: "Telemetry and alerting"]
   ▼
Instrument [O1] Three pillars via Micrometer + OpenTelemetry:
   Metrics: spring-boot-starter-actuator + micrometer-registry-prometheus
            /actuator/prometheus  (JVM, HTTP, Kafka consumer lag, Mongo, Resilience4j)
   Traces : OTel (Micrometer Tracing bridge), W3C traceparent propagated through
            HTTP, Kafka headers, and Istio (Envoy spans)
   Logs   : structured JSON, with traceId/spanId/correlationId/aggregateId, no PII
   ▼
Business metrics [O2]: orders_placed_total, saga_completed/compensated_total,
                       saga_duration_seconds, outbox_lag, projection_lag
   ▼
Collection and storage:
   Prometheus scrapes pods (ServiceMonitor/PodMonitor) and Istio/Envoy metrics
   ── option A: Azure Monitor managed service for Prometheus + Azure Managed Grafana
   ── option B: self-hosted Prometheus + Grafana
   Azure Monitor / Application Insights: traces, logs, availability tests,
   Container Insights, Log Analytics (KQL) via the Azure Monitor OpenTelemetry distro or agent
   ▼
Dashboards as code [O3] (Grafana JSON/Terraform, version-controlled):
   RED metrics (Rate, Errors, Duration) per service
   JVM/GC/threads, Kafka lag, circuit breaker state, saga success vs compensation,
   Mongo latency, Istio golden signals, SLO burn rate
   ▼
Alerting [O4]:
   Grafana alert rules and/or Azure Monitor alerts, routed to Action Groups (Teams/PagerDuty/email)
   Symptom-based, SLO burn-rate alerts (fast and slow window); every alert links a runbook
   Examples: p99 latency, 5xx rate, breaker open > 5m, consumer lag growing,
             DLQ depth > 0, outbox lag, saga stuck, pod crashloop, Entra token failures
   ▼
[O5] Runbooks + health-check synthetic tests generated alongside the code
   ▼
Validation: test alerts fire in staging (inject faults) → confirm dashboard and runbook link
   ▼
◇ Pass? ◇ ─ No → fix
```

## 9) Flow 8: CI/CD with GitHub Copilot and the agents

```
GitHub Issue (from tasks.md, acceptance criteria included)
   ▼
Assign to Copilot coding agent  OR  your orchestrated agent  [A3]
   ▼ reads .github/copilot-instructions.md + AGENTS.md  [A11]
Branch → implement → tests → PR (draft)
   ▼
GitHub Actions:
   build + unit/slice tests → Testcontainers integration → Pact verify
   Spotless/Checkstyle + ArchUnit → SonarQube
   CodeQL + dependency review + secret scan + Trivy (image + IaC)
   build image (Jib/Buildpacks) → SBOM → sign (cosign) → push to ACR
   ▼
Copilot code review + reviewer agent [A7] + human CODEOWNERS review  [Gate 2]
   ▼
Merge → GitOps repo update (image tag) → Argo CD/Flux sync to AKS  [P6]
   ▼
Environment promotion: dev → test → staging (contract + E2E + chaos + load) → prod [Gate 3]
   ▼
Canary → SLO check → promote or auto-rollback
```

## 10) Verification loop (test pyramid for this stack)

```
Unit (domain, pure Java) → ArchUnit rules → Spring slice tests (@WebMvcTest, @DataMongoTest)
   ▼
Testcontainers: Mongo (replica set) + Kafka + Schema Registry
   ▼
Contract: OpenAPI conformance, Pact (HTTP + message), schema compatibility check
   ▼
Saga tests: happy path, each failure point, compensation, duplicate and out-of-order events
   ▼
Resilience tests: WireMock/Toxiproxy (timeouts, 5xx), breaker open/half-open/close
   ▼
Projection tests: replay from empty read DB → identical state
   ▼
Security tests: token matrix, authorization policies
   ▼
Performance: Gatling/k6 (throughput, p99, consumer lag under load)
   ▼
Chaos (staging): pod kill, broker outage, Mongo failover, Istio delay/abort faults
   ▼
◇ All pass? ◇ ─ No → [A8] route failure to owning agent → fix → re-run
```

## 11) Runtime view: Place Order saga with CQRS (orchestrated)

```
Client → Istio Gateway (TLS, JWT check) → Order Service (sidecar mTLS)
  1  Order Service: validate cmd, Aggregate creates Order(PENDING)
  2  Mongo TX: save orders + outbox(OrderCreated)
  3  Outbox relay → Kafka orders.events.v1
  4  Saga Orchestrator consumes OrderCreated → sends ReserveInventory cmd
  5  Inventory Service: reserve → outbox → InventoryReserved
  6  Orchestrator → ProcessPayment cmd → Payment Service (Resilience4j → PSP)
  7a Success: PaymentCompleted → Orchestrator → ConfirmOrder → Order(CONFIRMED)
  7b Failure: PaymentFailed → ReleaseInventory (compensation) → Order(CANCELLED)
  8  Projection consumers update order_summary_view in Mongo (idempotent)
  9  Client GET /orders/{id} → QueryHandler → read view (eventual consistency)
  *  Every hop: traceparent + correlationId; metrics to Prometheus; traces to Azure Monitor
```

## 12) Pattern catalog

### A) Agent patterns

| ID | Pattern | Use |
|---|---|---|
| A1 | Prompt chaining | Interview → domain model → spec → code |
| A2 | Routing | Send tasks to domain, messaging, platform, security or observability specialists |
| A3 | Orchestrator-Workers | One master agent delegates and merges |
| A4 | Plan-and-Execute | `tasks.md` before coding |
| A5 | ReAct | Read failure → fix → rerun |
| A6 | Tool use / MCP | Maven, git, kubectl, istioctl, Pact, Testcontainers, scanners |
| A7 | Reflection / reviewer agent | Independent review against the spec |
| A8 | Evaluator-Optimizer | Tests, chaos and SLO checks drive iteration |
| A9 | Parallelization and voting | Parallel tasks and multi-reviewer checks |
| A10 | RAG over the repo | Golden templates, existing aggregates, docs |
| A11 | Memory | `AGENTS.md`, copilot-instructions, compaction |
| A12 | Human-in-the-loop | Three gates plus escalation |
| A13 | Guardrails / sandbox | No prod access, scoped tokens, budgets |
| A14 | Spec-driven + TDD | Contracts and failing tests first |
| A15 | Checkpoint / rollback | Commit per step, tags |

### D) Domain and data patterns

| ID | Pattern | Use |
|---|---|---|
| D1–D3 | Event Storming, Bounded Context, Ubiquitous Language | Service boundaries |
| D4–D7 | Aggregate, Value Object, Domain Event, Domain Service | Tactical DDD |
| D8 | Hexagonal / Clean architecture | Domain isolated from frameworks |
| D9–D10 | Application service, Repository | Use cases and persistence ports |
| D11–D12 | DTO/Mapper, Anti-Corruption Layer | Boundary protection |
| D13 | Optimistic locking | Aggregate concurrency |
| D14 | CQRS projections | Mongo read models |
| D15 | Eventual consistency contract | API and UX behavior |
| D16 | Event Sourcing (optional) | Audit and temporal needs |

### E) Event-driven patterns

| ID | Pattern | Use |
|---|---|---|
| E1 | Schema Registry, versioned topics | Compatible evolution |
| E2 | Transactional Outbox | Atomic DB write and event publish |
| E3–E4 | Saga (choreography or orchestration) plus compensation | Distributed transactions |
| E5 | Idempotent producer | No duplicates on retry |
| E6 | Event envelope with correlation IDs | Traceability |
| E7 | Idempotent consumer | Safe redelivery |
| E9–E10 | Retry topics, DLQ, poison-pill handling | Failure isolation |
| E11 | Competing consumers, partition keys | Scale and ordering |
| E12 | Process Manager / saga timeout | Stuck-flow recovery |
| E8 | Consumer-driven contracts | Safe parallel development |

### R) Resilience patterns

| ID | Pattern | Use |
|---|---|---|
| R1 | Circuit Breaker | Stop calling a failing dependency |
| R2 | Retry with backoff and jitter | Transient faults |
| R3 | Timeout | Bound latency |
| R4 | Bulkhead | Isolate resource pools |
| R5 | Rate Limiter | Protect downstreams |
| R6 | Fallback / degradation | Graceful responses |
| R7 | Chaos testing, auto rollback | Prove and enforce resilience |
| R8 | Idempotency keys | Make retries safe |

### P) Platform patterns

| ID | Pattern | Use |
|---|---|---|
| P1 | API-first | OpenAPI before code |
| P2 | Factory / template scaffolding, containerization | Consistent services |
| P3 | Dependency Injection | Testability |
| P4 | 12-factor config | Externalized configuration |
| P5 | Canary / blue-green / feature flags | Safe rollout |
| P6 | GitOps | Declarative deploys |
| P7 | Health probe, graceful shutdown | Reliable orchestration |
| P8 | Sidecar proxy (Istio/Envoy) | mTLS, traffic policy, telemetry |

### O) Observability and X) Security patterns

| ID | Pattern | Use |
|---|---|---|
| O1 | Three pillars with trace correlation | Debuggability |
| O2 | Business metrics | Saga and outbox health |
| O3 | Dashboards as code | Reproducible Grafana |
| O4 | SLO burn-rate alerting | Actionable alerts |
| O5–O6 | Runbooks, feedback loop | Operations and learning |
| X1–X2 | JWT resource server, role mapping | Entra ID authN/authZ |
| X3 | Defense in depth (mesh plus app) | No single point of trust |
| X4–X6 | Workload identity, Key Vault, data protection | Secrets and PII |

## Selection rules for the agent

1. **Don't use every pattern everywhere.** CQRS, Saga and Event Sourcing are chosen **per bounded context**, and each choice gets an ADR. A simple CRUD context should stay simple.
2. **One layer owns retries** (app or mesh), never both on the same call path.
3. **Outbox is mandatory** whenever a DB write must be paired with an event. Never dual-write to Mongo and Kafka.
4. **Idempotency everywhere**: consumers, projections, saga steps and commands.
5. **Contracts first** (OpenAPI, AsyncAPI, schemas), so parallel agents don't diverge.
6. **Mongo transactions need a replica set.** If you use Cosmos DB for Mongo, check its transaction and change-stream support first.
7. **Agent safety**: Copilot or any agent only touches dev/sandbox. Production changes go through GitOps and human approval.
8. **Tests are the primary feedback signal** (A5 plus A8). Self-review alone is weaker.

If you share your Java and Spring Boot versions, whether you use Cosmos DB for Mongo or self-managed Mongo, Kafka flavor (Confluent, Event Hubs for Kafka, or Strimzi), and Istio mode (sidecar or ambient), I can produce the `AGENTS.md`, `.github/copilot-instructions.md`, a service template layout, and the ADR templates for this flow.
