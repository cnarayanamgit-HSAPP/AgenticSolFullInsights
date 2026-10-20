Coding Agent workflow on React MFE
Two kinds of patterns appear in the diagram, tagged inline:

- **[A#] Agent patterns**: how the coding agent itself works.
- **[S#] Software patterns**: what the agent puts into the generated app.

## 1) Master workflow with pattern tags

```
┌─────────────────────────────────────────────────────────────┐
│ START: Human goal                                           │
│ "React Orders app, embed in ASP.NET/JSP/PHP/jQuery/AngJS"   │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. CONTEXT LOAD                                             │
│ [A10] RAG / repo map retrieval   [A11] Memory (AGENTS.md)   │
│ [A13] Sandbox + least-privilege permissions                 │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. REQUIREMENTS INTERVIEW                                   │
│ [A12] Human-in-the-loop   [A1] Prompt chaining (Q&A → spec) │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SPEC + ARCHITECTURE  (Planner agent)                     │
│ [A14] Spec-driven development                               │
│ [S1] DDD bounded contexts      [S2] Strangler Fig strategy  │
│ [S6] Shell + Remote MFE        [S3] Web Component adapter   │
│ [S7] API Gateway + BFF         [S16] Anti-corruption layer  │
│ Outputs: spec.md, ADRs, openapi.yaml, DB schema, mfe-contract│
└───────────────────────────┬─────────────────────────────────┘
                            ▼
                ◇ GATE 1: human approves? ◇   [A12]
                │ No → revise        │ Yes
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. TASK PLAN (tasks.md)                                     │
│ [A4] Plan-and-Execute   [A2] Routing (task → specialist)    │
│ [A3] Orchestrator-Workers                                   │
└───────────────────────────┬─────────────────────────────────┘
                            ▼  [A9] Parallelization (independent tasks)
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
┌───────────────┐  ┌─────────────────┐  ┌──────────────────┐
│ 5A BACKEND+DB │  │ 5B REACT MFE    │  │ 5C HOST EMBED    │
│ (flow 2)      │  │ (flow 3)        │  │ (flow 4)         │
└───────┬───────┘  └────────┬────────┘  └────────┬─────────┘
        └───────────────────┼────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. INTEGRATION + E2E VERIFY                                 │
│ [A8] Evaluator-Optimizer (tests/scans as evaluator)         │
│ [S14] Consumer-driven contract tests                        │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
              ◇ All green? ◇ ── No → route defect to owner → fix → re-verify [A2][A5]
                            │ Yes
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. INDEPENDENT REVIEW                                       │
│ [A7] Reflection via separate Reviewer agent                 │
│ [A9] Voting (security + quality reviewers in parallel)      │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
                ◇ GATE 2: human approves PR? ◇   [A12]
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. BUILD, SIGN, PUBLISH                                     │
│ versioned bundle + manifest.json   [A15] Checkpoint (git tag)│
└───────────────────────────┬─────────────────────────────────┘
                            ▼
                ◇ GATE 3: prod approval? ◇   [A12]
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. PROGRESSIVE ROLLOUT                                      │
│ [S15] Feature flag + canary / blue-green                    │
│ [S11] Auto rollback on SLO breach                           │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 10. FEEDBACK LOOP                                           │
│ [A16] Observability + eval set   [A11] update AGENTS.md     │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
                           END
```

## 2) Flow 5A: Backend and database

```
[Task] → [A10] retrieve similar service code + golden template
   ▼
[S17] Factory: scaffold service from template
   ▼
[A14] TDD: write FAILING tests (unit + contract from OpenAPI)
   ▼
Implement, layered / Clean Architecture [S8]:
   Controller → Service (domain) → Repository → DB
   [S8] Repository + Unit of Work   [S9] Dependency Injection
   [S19] DTO/Mapper   [S10] CQRS (only if read/write models diverge)
   [S11] Retry + Circuit Breaker + Timeout + Idempotency keys
   [S19] Saga / Outbox (only if multiple services or async events)
   ▼
[A6] Tool use: build, test, lint, SAST
   ▼
◇ Green? ◇ ── No → [A5] ReAct: read error → reason → fix → retry
   │                (retry limit exceeded → [A12] escalate to human)
   │ Yes
   ▼
Pact provider verification [S14] → commit [A15] → PR → CI → publish API + TS types
```

## 3) Flow 5B: React MFE

```
[Task] (needs openapi.yaml, mfe-contract.md, mock server)
   ▼
[S17] Factory: scaffold React + TypeScript from template
   ▼
[S18] Generated API client from OpenAPI (Proxy to the API Gateway)
   ▼
[A14] Write FAILING component tests (React Testing Library)
   ▼
Implement React patterns:
   [S13] Container / Presentational components
   [S13] Custom hooks (data fetching, auth)
   [S13] Context Provider (theme, auth, i18n)
   [S13] Flux/Redux-style or React Query state (server vs client state split)
   [S12] Error Boundary + Suspense fallback
   [S13] Lazy loading / code splitting
   [S4]  Facade: one small public API surface to the host
   ▼
Implement MFE contract (legacy compatibility):
   [S3]  Adapter: mount(el, props) / unmount(el), custom element <orders-mfe>
   [S5]  Observer / Pub-Sub: CustomEvents OUT, listeners IN
   [S17] Singleton: React/ReactDOM shared or bundled once
   [S3]  Shadow DOM or CSS prefix (style isolation)
   [S16] Anti-corruption layer: map legacy props/events to clean internal models
   ▼
typecheck + lint + unit tests + bundle-size budget
   ▼
◇ Green? ◇ ── No → [A5] ReAct fix loop → [A7] self-reflect on diff
   │ Yes
   ▼
Smoke test vs mock server → commit [A15] → PR → CI builds versioned bundle
```

## 4) Flow 5C: Host embedding

```
[S4] Facade: loader.js (read manifest → load bundle → mount → fallback)
   ▼
[S2] Strangler Fig: legacy section replaced behind the same URL, with legacy fallback kept
   ▼
┌──────────────┬───────────┬────────────┬─────────────┬────────────────────┐
│ ASP.NET      │ JSP       │ PHP        │ jQuery      │ AngularJS 1.x      │
│ div + data-* │ include + │ template + │ $(document) │ Directive wraps    │
│ attributes   │ escaped   │ json_encode│ .on(event)  │ mount; $applyAsync │
│              │ JSON      │ + CSP nonce│ [S5]        │ [S3] Adapter       │
│ [S3] Adapter │ [S3]      │ [S3]       │             │ unmount on $destroy│
└──────┬───────┴─────┬─────┴──────┬─────┴──────┬──────┴──────────┬─────────┘
       └─────────────┴────────────┴────────────┴─────────────────┘
                             ▼
   [S18] Token relay: short-lived token passed via props (never scraped from cookies)
   [S7]  All calls through API Gateway / BFF
                             ▼
   Host harness pages + docker-compose (hosts + API + DB + mock IdP)
```

## 5) Flow 6: Verification loop

```
docker-compose up (ephemeral env)   [A13] sandboxed
   ▼
Per host: load → mount → API call → render → click → host receives event
   ▼
[S14] Pact consumer/provider   a11y (axe)   visual regression
   ▼
CSS-bleed check   duplicate-React check   unmount cleanup check
   ▼
Security: dependency scan, DAST, CSP, XSS on injected props   [A9] parallel runs
   ▼
Negative tests: API 5xx, bundle 404 (fallback), expired token   [S11]
   ▼
◇ All pass? ◇ ── No → [A8] Evaluator-Optimizer: feed failures back to owner agent
   │ Yes
   ▼
Report to Orchestrator
```

## 6) Agent inner loop (applies to every task)

```
[A5] ReAct:  THINK → ACT (tool call) → OBSERVE → repeat
[A4] Plan:   write short plan before editing
[A7] Reflect: critique own diff against spec before commit
[A15] Checkpoint: commit per step, so rollback is cheap
[A11] Memory: scratchpad + compaction when context grows
[A12] Escalate: stop and ask when ambiguous, risky, or retry/budget limit hit
```

## 7) Pattern catalog

### A) Agent design patterns

| ID | Pattern | Where used |
|---|---|---|
| A1 | Prompt chaining | Interview → spec → plan → code |
| A2 | Routing | Orchestrator sends tasks to backend/React/integration/QA agents |
| A3 | Orchestrator-Workers | Master agent delegates and merges results |
| A4 | Plan-and-Execute | `tasks.md` before any coding |
| A5 | ReAct (reason-act-observe) | Fix loops after failed tests or builds |
| A6 | Tool use / MCP | Shell, git, OpenAPI lint, Pact, Playwright, scanners |
| A7 | Reflection / self-critique | Reviewer agent and pre-commit self-review |
| A8 | Evaluator-Optimizer | Tests, scans and evals drive iteration |
| A9 | Parallelization | Independent tasks, parallel scans, multi-reviewer voting |
| A10 | RAG / repo retrieval | Repo map, golden examples, docs lookup |
| A11 | Memory | `AGENTS.md`, scratchpad, context compaction |
| A12 | Human-in-the-loop | Gates at spec, PR, production, plus escalation |
| A13 | Guardrails / sandbox | No prod secrets, scoped permissions, budget limits |
| A14 | Spec-driven + TDD | Contracts and failing tests first |
| A15 | Checkpoint and rollback | Commit per step, git tags, rollback |
| A16 | Observability and eval loop | Traces, cost, pass rate feed the backlog |

### S) Software design patterns in the generated app

| ID | Pattern | Purpose |
|---|---|---|
| S1 | DDD bounded context | Split domains into services/MFEs |
| S2 | Strangler Fig | Replace legacy screens incrementally |
| S3 | Adapter / Web Component | `mount/unmount` bridging React and legacy hosts |
| S4 | Facade | `loader.js` and a minimal MFE public API |
| S5 | Observer / Pub-Sub | CustomEvents between React and jQuery/AngularJS |
| S6 | Shell + Remote (MFE composition) | Module Federation or single-spa when modern hosts exist |
| S7 | API Gateway / BFF | Single entry, auth, rate limit, UI-shaped APIs |
| S8 | Layered/Clean architecture, Repository, Unit of Work | Backend structure |
| S9 | Dependency Injection | Testable services |
| S10 | CQRS (optional) | Separate read/write models when needed |
| S11 | Resilience: retry, circuit breaker, timeout, bulkhead, error boundary | Fault tolerance |
| S12 | Error Boundary | Contain React render failures |
| S13 | React patterns: container/presentational, hooks, context provider, lazy loading, server-state cache | UI structure |
| S14 | Consumer-driven contract testing | Safe parallel work across agents |
| S15 | Feature flags, canary, blue-green | Safe rollout |
| S16 | Anti-corruption layer | Isolate legacy data and prop shapes |
| S17 | Factory, Singleton | Template scaffolding; shared React instance |
| S18 | Proxy / token relay | Typed API client, safe auth handoff |
| S19 | Saga, Outbox, DTO/Mapper | Multi-service consistency and data mapping |

## Selection rules for the agent

- Use **Web Component + Shadow DOM (S3)** by default for legacy hosts, and **Module Federation (S6)** only for modern hosts.
- Add **CQRS and Saga** only when the spec shows multiple services or divergent read/write needs. Avoid pattern overuse.
- Every pattern choice goes into an **ADR**, so humans can review it at Gate 1.
- Prefer **A5 + A8** (tests as the feedback signal) over **A7** alone, since self-critique without executable checks is weaker.

If you share your host versions and cloud (for example ASP.NET MVC 5, AngularJS 1.8, Tomcat/JSP, PHP 8, Azure or AWS), I can produce the `AGENTS.md` with these patterns encoded as rules, plus the ADR templates.
