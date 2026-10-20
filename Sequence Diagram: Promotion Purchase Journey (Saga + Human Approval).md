# Sequence Diagram: Promotion Purchase Journey (Saga + Human Approval)

**Scenario:** A customer asks the agent to buy a high-value promotional bundle (e.g., a premium plan with financing). The order is above the auto-approve threshold, so it needs **human approval**, and the purchase runs as a **Saga** with compensation.

## 1. Participants

```
U    = User (mobile/web)
GW   = Channel API Gateway (WAF, auth, rate limit)
RT   = Fast Path (guardrail, intent router, semantic cache)
SUP  = LOB Supervisor Agent (orchestrator)
SPA  = Specialist Agent (Offer/Purchase)
CRT  = Critic / Verifier Agent
MG   = Model Gateway (quota, routing, fallback)
LLM  = LLM tier (small / mid)
TG   = Tool / MCP Gateway (authz, idempotency, ACL)
RM   = CQRS Read Models (cache, replicas, vector DB)
HITL = Human Approval Service (approver console)
WF   = Saga / Workflow Engine
INV  = Inventory Service
PAY  = Payment Service
ORD  = Order Service
NTF  = Notification Service
K    = Kafka (event backbone + Outbox)
AUD  = Agent Decision Log (event log, audit)
```

## 2. Phase A: Intake, Triage, Understanding (hot path)

```
U        GW         RT          SUP         MG/LLM        TG        RM
│ "Buy   │          │           │            │            │         │
│ Promo  │          │           │            │            │         │
│ Bundle"│          │           │            │            │         │
│───────►│          │           │            │            │         │
│        │ authN(JWT), rate-limit, WAF       │            │         │
│        │─────────►│           │            │            │         │
│        │          │ input guardrail (PII, injection)    │         │
│        │          │ semantic cache lookup -> MISS       │         │
│        │          │ intent = PURCHASE (small model)     │         │
│        │          │──────────►│ handoff w/ user ctx     │         │
│        │          │           │ load session (Redis)    │         │
│        │          │           │ plan (Tier 1/2 model)   │         │
│        │          │           │───────────►│            │         │
│        │          │           │◄───────────│ intent+slots│        │
│        │          │           │ PARALLEL tool fan-out   │         │
│        │          │           │────────────────────────►│ get_offer│
│        │          │           │────────────────────────►│ get_eligibility
│        │          │           │────────────────────────►│ get_price│
│        │          │           │            │            │────────►│ read replica/cache
│        │          │           │◄────────────────────────│◄────────│
│◄ stream "Found offer X, ₹/$ price, eligible. Confirm?"   │         │
│ (token streaming, TTFT ~1.2s)                            │         │
```

## 3. Phase B: Confirmation and Risk Evaluation

```
U        SUP         CRT        MG/LLM       TG/Policy(OPA)     AUD
│ "Yes,  │           │           │               │               │
│ confirm"│          │           │               │               │
│───────►│           │           │               │               │
│        │ build purchase intent (task-level tool: place_promo_order)
│        │──────────►│ verify: price matches offer, eligibility valid,
│        │           │ amount < user limit, no policy breach
│        │           │──────────►│ (evaluator-optimizer pass)
│        │           │◄──────────│
│        │◄──────────│ verdict: OK, RISK = HIGH (amount > auto-approve limit)
│        │ policy check (delegated user identity, least privilege)
│        │──────────────────────────────────────►│ ALLOW w/ approval required
│        │ write decision record (prompt hash, tools, verdict) ───────►│
│◄ "This purchase needs a quick approval. Estimated: 2 min. I'll notify you."
```

## 4. Phase C: Human-in-the-Loop Approval (async, durable)

```
SUP        WF            HITL(Approver)      K            U
│          │                │                │            │
│ start saga instance (idempotency key = orderRequestId)  │
│─────────►│                │                │            │
│          │ state=PENDING_APPROVAL (durable, survives pod restarts)
│          │ create approval task (summary, amount, risk reason, agent rationale)
│          │───────────────►│                │            │
│          │                │ reviewer sees: offer, price, user, evidence
│          │                │ decision: APPROVE / REJECT / REQUEST-INFO
│          │◄───────────────│                │            │
│          │ log approver id, time, decision ─────────────► AUD
│          │ timeout rule: no response in SLA -> escalate / auto-cancel
│          │
│   (agent pods are stateless: nothing is held in memory while waiting)
```

## 5. Phase D: Saga Execution (core services on K8s)

```
WF          INV            PAY             ORD            K(Outbox)      NTF
│           │              │               │               │             │
│ STEP 1: reserve inventory (idempotent, key=reqId)                       │
│──────────►│              │               │               │             │
│◄──────────│ reserved (hold TTL 10 min)   │               │             │
│ STEP 2: authorize + capture payment      │               │             │
│─────────────────────────►│               │               │             │
│◄─────────────────────────│ payment OK (txnId)            │             │
│ STEP 3: create order                     │               │             │
│─────────────────────────────────────────►│               │             │
│◄─────────────────────────────────────────│ order CONFIRMED             │
│                                          │ write OrderCreated -> OUTBOX │
│                                          │──────────────►│ publish      │
│ STEP 4: notify                                           │─────────────►│ push/email
│ saga state = COMPLETED                                                  │
```

## 6. Phase E: Failure Path (Compensation)

```
WF          INV            PAY             ORD            SUP        U
│ (Step 2 payment FAILS or times out after retries+jitter,
│  circuit breaker OPEN on PAY dependency)
│           │              │               │               │         │
│ COMPENSATE in reverse order:
│ release inventory hold                    │               │         │
│──────────►│ released      │               │               │         │
│ void/refund payment (if partially authorized)             │         │
│─────────────────────────►│ voided        │               │         │
│ cancel order draft                        │               │         │
│─────────────────────────────────────────►│ cancelled      │         │
│ saga state = COMPENSATED ; emit SagaFailed event          │         │
│──────────────────────────────────────────────────────────►│ notify  │
│                                                           │────────►│ "Payment couldn't
│                                                                       complete, nothing
│                                                                       was charged. Retry?"
│ Fallback ladder: retry -> alternate payment route -> human support ticket
```

## 7. Phase F: Completion, Rejection, and Learning Loop

```
WF         SUP          U            K             AUD / Eval
│          │            │            │              │
│ COMPLETED event ─────►│            │              │
│          │◄───────────│ (event consumed by agent; session updated)
│          │ stream/push: "Order confirmed. Order #123, delivery details..."
│          │            │            │              │
│ If REJECTED by approver:                          │
│          │ saga cancelled, no side effects (nothing reserved yet)
│          │ polite explanation + alternatives (offer lower tier)
│          │            │            │              │
│          │ publish AgentTaskCompleted ───────────►│ full trace (decision log)
│          │            │            │              │ sampled for quality eval,
│          │            │            │              │ cost per conversation (FinOps)
```

## 8. Pattern Map for This Journey

| Step | Pattern |
|---|---|
| Edge auth, rate limit | Gateway, IAM, Rate Limiting |
| Cache miss, intent routing | Routing, Cache-Aside, Guardrail |
| Parallel offer/eligibility/price | Fan-out/Fan-in, CQRS read side, BFF task-level tools |
| Verifier check | Evaluator-Optimizer |
| Risk-based approval | HITL gate, Policy as Code (OPA), Risk tiering |
| Waiting for approver | Durable workflow, Stateless agents, Async request-reply |
| Reserve, pay, order | Saga (orchestration), Idempotency keys, DDD bounded contexts |
| Order event publish | Outbox, Event-driven (Kafka) |
| Payment failure | Circuit Breaker, Retry+Jitter, Compensation, Fallback ladder |
| Audit trail | Event Log, Agent decision log |
| Legacy payment system | Anti-Corruption Layer, Strangler facade |

## 9. Key Design Rules Illustrated

1. **Agent decides and requests; the Saga engine executes and guarantees.** The LLM never calls inventory, payment, and order directly in sequence.
2. **Approval is a workflow state, not an agent waiting.** Agents stay stateless, so 1M users/hour doesn't tie up pods.
3. **Every write carries an idempotency key**, so retries or duplicate agent calls cannot double-charge.
4. **Compensation is defined for each step** before go-live (release, void, cancel).
5. **User identity is delegated** end to end: the agent never has more rights than the user.
6. **Everything is logged** (decision, approval, saga steps) for audit and replay.
7. **Low-risk purchases skip Phase C**: below threshold, the verifier passes and the Saga starts immediately (keeps latency low at scale).

I can also produce this as Mermaid `sequenceDiagram` code for draw.io or documentation tools, or add a variant for a low-risk auto-approved purchase.
