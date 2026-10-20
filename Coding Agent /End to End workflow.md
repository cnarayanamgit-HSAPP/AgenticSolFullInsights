
# End-to-end workflow

**Phase 0: Setup (one-time, human-owned)**
- Create the repo(s) with `AGENTS.md`/`CLAUDE.md`: stack, commands, conventions, do/don't rules.
- Provide golden templates (service skeleton, MFE remote skeleton, Terraform module, agent skeleton).
- Set up a sandbox, CI pipeline, secrets isolation, and MCP tools (git, cloud CLI, issue tracker, docs).

**Phase 1: Requirements intake**
1. The agent interviews the human: users, scope, NFRs, compliance, cloud target, and autonomy limits for any AI features.
2. It produces `spec.md` with user stories, Given/When/Then acceptance criteria, and non-goals.
3. **Gate: human approves the spec.**

**Phase 2: Architecture and contracts**
1. Decompose into bounded contexts (DDD), then map them to MFEs, microservices, and agents.
2. Write ADRs: sync vs. async, federation approach, data ownership, auth, deployment model.
3. Generate machine-readable contracts first: OpenAPI/GraphQL, AsyncAPI event schemas, DB schemas, agent tool schemas.
4. **Gate: human approves the architecture and contracts.**

**Phase 3: Plan and task breakdown**
- Convert the spec into small, independently testable tasks with dependencies (`plan.md`/`tasks.md`).
- Order them: contracts, then infra baseline, then backend, then MFEs, then agentic layer, then integration.

**Phase 4: Build, in dependency order, each task in a TDD loop**

The loop for every task is: write failing tests, implement, run lint/type check/tests, self-correct, commit on a branch.

| Layer | Agent actions | Verification |
|---|---|---|
| **Cloud-native foundation** | IaC (Terraform/Bicep/CDK), container images, Kubernetes/serverless manifests, config and secrets, observability setup | `terraform validate/plan`, policy checks, image scan |
| **Backend and microservices** | Scaffold from template, implement domain logic, persistence, events, resilience (retries, circuit breakers, idempotency) | Unit and integration tests, **contract tests** (Pact), API conformance to OpenAPI |
| **MFE and full stack UI** | Build the shell and remotes, shared design system, routing, cross-MFE events, API clients generated from contracts | Component tests, E2E (Playwright), accessibility checks, federation build test |
| **Agentic AI layer** | Implement agent loop, tools, memory/RAG, guardrails, prompt versioning, tracing | Eval dataset (pass rate, hallucination, tool accuracy), red-team tests, cost/latency budgets |

**Phase 5: Integration and system verification**
- Spin up the full stack in an ephemeral environment (docker-compose or a preview namespace).
- Run E2E user journeys against the acceptance criteria, plus load, security (SAST/DAST/dependency scan), and chaos or failure-mode tests.

**Phase 6: Review**
- A separate reviewer agent checks diffs against the spec, ADRs, and standards. Fixes go back to the implementer.
- **Gate: human PR review.**

**Phase 7: Deploy and operate**
- CI/CD builds, signs, and deploys with progressive rollout (canary/blue-green) and automatic rollback on SLO breach.
- **Gate: human approval for production.**
- Dashboards, alerts, and runbooks are generated as part of delivery.

**Phase 8: Feedback loop**
- Feed production traces, incidents, and defects back into the backlog, the eval set, and `AGENTS.md` rules.

# Multi-agent roles (optional but effective)

An **orchestrator** routes work to:
- **Planner/architect** (spec, ADRs, contracts)
- **Infra agent** (IaC, pipelines)
- **Backend agent** (services, events)
- **Frontend agent** (shell, MFEs)
- **AI agent** (agent logic, evals)
- **Tester agent** (contract, E2E, security)
- **Reviewer agent** (independent check)

Each has scoped tools and permissions, so the frontend agent can't touch production infra, for example.

# Key principles

- **Contracts before code**, so parallel agents stay consistent.
- **Verification at every step**, since tests and scans are the agent's feedback signal.
- **Small tasks, small diffs**, to keep context focused and reviews easy.
- **Human gates** at spec, architecture, PR, and production.
- **Everything logged** (prompts, tool calls, diffs, cost) for debugging and evaluation.

If you share your concrete stack (for example React with Module Federation, Spring Boot or Node, Kubernetes on AWS or Azure, and an agent framework), I can turn this into a ready-to-use `AGENTS.md`, a `spec.md` template, and a task-breakdown prompt.
