# UtterRoute — Product Requirements Document (PRD)

**Release:** v0.1 (proposed)  
**Status:** Draft for discussion; no implementation is shipped  
**Last updated:** 2026-11-21  
**Related:** [Technical specification](SPEC.md) · [Product scope ADR-003](ADR-003-CAPABILITY-RESOLVER.md) · [Project README](../README.md)

## 1. Summary

UtterRoute is a self-hostable, MIT-licensed **A2A-first capability resolver** for existing REST APIs **and independently running A2A agents**. It registers operator-approved capabilities, selects one that an authenticated caller is allowed to use, and delegates through a small protocol-specific adapter. It is **not** a general API gateway, reverse proxy, agent runtime or API-management product.

**Promise:** *Discover capabilities, resolve intent, dispatch safely.* A2A messages in; one approved REST operation or downstream A2A agent task out.

**Fast by default:** When an eligible capability is explicitly selected or determined by an unambiguous configured rule, UtterRoute dispatches **without inference**. For ambiguous intent, Jev is the first **optional** semantic selector; SystemOne may be compared later. Without a suitable configured router, unresolved ambiguity leads to clarification/abstention. A TypeSafe key is **not required** for deterministic usage.

**Infrastructure boundary:** Use established gateways/ingress for TLS, DDoS/WAF, generic HTTP rate limiting and identity infrastructure. UtterRoute retains caller-to-task and caller-to-capability authorization plus minimal fixed-egress and workload safeguards. See [ADR-003](ADR-003-CAPABILITY-RESOLVER.md).

## 2. Problem and customer

### Problem

An A2A client can invoke a known agent, and a conventional client can invoke a known REST endpoint. It is harder to **select** the right eligible capability among existing APIs and A2A agents without custom wiring, a slow inference step on every call or an unsafe black-box agent. API gateway products solve transport but do not eliminate this selection problem.

### Initial users

- **Primary:** Developers integrating existing REST capabilities and A2A agents into an A2A environment who need discoverable, authorized selection.
- **Secondary:** Agent/platform teams evaluating semantic selection against deterministic dispatch and direct invocation.
- **Not the v0.1 buyer:** High-throughput HTTP proxy users, API-management teams buying portals/traffic products, or end users seeking a general chatbot.

### Job to be done

> When my agent needs a capability available through an existing API or specialist A2A agent, I want to register it once, discover its eligible capabilities, and invoke the correct target through A2A without building a general API gateway or bespoke workflow orchestrator.

## 3. Product principles

1. **Capability resolution, not generic proxying:** Optimize discovery, selection and safe one-target dispatch; use existing gateways for infrastructure.
2. **Deterministic first:** Explicit authorized selection and unambiguous rules bypass Jev/SystemOne; ambiguous input abstains or invokes an **optional** configured semantic selector.
3. **One target per task:** A single permitted REST GET *or* one approved A2A delegation. No fan-out, write-capable REST operations or multi-agent orchestration.
4. **Authorization before selection:** Target metadata, caller hints and model outputs are untrusted; policy filters the candidate set and validates the decision.
5. **Honest downstream semantics:** REST is read-only in the first release. A2A agents may have side effects and persistent tasks; operator policy must explicitly approve their registered capabilities and classify delegation risks.
6. **Measurable benefit:** Compare against direct invocation and simple deterministic dispatch for overhead, accuracy, abstention and integration effort.
7. **Open, small and composable:** Go + A2A SDK, local configuration, replaceable router and thin target adapters; no proprietary infrastructure dependency for baseline functionality.

## 4. v0.1 scope

### In scope (P0)

| ID | Requirement | Acceptance criteria |
| --- | --- | --- |
| PRD-01 | Registered REST capabilities | Parse a local OpenAPI 3.x subset, allowlist GET operations and pin trusted configured origins |
| PRD-02 | Registered A2A capabilities | Register a **trusted existing remote A2A agent** and operator-approved local capability backed by a trusted agent URL; treat Agent Card declarations as discovery metadata, not enforceable per-capability permissions; deny unknown-side-effect targets by default |
| PRD-03 | Unified capability catalog | Stable capability IDs, target kinds (`rest_operation` / `a2a_agent`), descriptions and operator eligibility policy |
| PRD-04 | **A2A Agent Card (primary)** | Advertise truthful protocol-defined Agent Card `skills` entries and HTTP+JSON security/interface declarations |
| PRD-05 | **A2A `SendMessage` / `GetTask`** | An independent A2A client can send an instruction, retrieve a task and inspect its artifact |
| PRD-06 | Task ownership and continuation | Caller-scoped bounded/TTL Task storage; input-required clarification; no unauthorized or repeated downstream invocation |
| PRD-07 | Deterministic-first selection | Explicit capability hints and configured unambiguous rules execute **without Jev**; unknown/unapproved/ambiguous intents safely refuse |
| PRD-08 | Thin REST dispatch | Validate supported scalar path/query values; execute at most one authorized GET per task; bound response sizes/timeouts |
| PRD-09 | Thin A2A delegation | Send at most one authorized downstream `SendMessage` per task, correlate child Task/context, handle completed/input-required/working within bounded limits |
| PRD-10 | Plan-only mode | A2A `utterroute.mode=plan` returns the chosen capability and sanitized preview without invoking a REST or A2A target |
| PRD-11 | Trust and safety boundaries | Verify caller eligibility, fixed target origins, agent capability approval, credentials isolation, bounded work, task ownership and no dispatch on refusal |
| PRD-12 | Reproducible two-target spike | One sample REST billing operation **and** one independent mock A2A research agent behind UtterRoute; external A2A client invokes both |
| PRD-13 | Measurement and observability | Compare direct target vs deterministic resolution overhead; report p50/p95, refusal, correctness, task state, side-effect safety |
| PRD-14 | Explicit build-vs-integrate scope | No generic HTTP reverse proxy/WAF/API management; document supported deployment behind existing ingress and its shared responsibilities |

### Deferred / potential v0.2 features

- **Optional Jev semantic selection:** First supported inference adapter and held-out benchmark, added only if deterministic routing cannot resolve the request safely. Its use must be opt-in and cannot be required for direct selection.
- **Optional SystemOne HTTP adapter** if the Jev comparative evaluation justifies it; no bundled inference runtime.
- **Secondary native HTTP API:** `POST /v1/execute` using the same resolver, subject to demonstrated user demand.
- Multiple registries, scalable candidate preselection, richer argument extraction, and more registered A2A target configurations.

### Explicitly out of scope

- **Generic API gateway features:** Arbitrary HTTP proxying, transformations, traffic splitting, load balancing, centralized API analytics/developer portals, key issuance, metering/billing, generic TLS/WAF/DDoS termination or identity provider.
- **General agent orchestration:** Multiple downstream targets per request, fan-out, chained tool calls, long-running workflow engine, arbitrary delegated write approvals.
- **REST writes:** API `POST`/`PUT`/`PATCH`/`DELETE` execution, general JSON body synthesis, transaction rollback.
- Auto-discovery of arbitrary remote agents or caller-supplied URLs; blind trust in Agent Cards, model descriptions or downstream side-effect claims.
- Streaming, push/cancel, durable multiworker task queues, additional A2A protocol bindings, MCP/GraphQL ingestion or hosted multitenant control plane.
- Claims of guaranteed routing accuracy, zero gateway overhead, general agent correctness or safe autonomous behavior of a remote agent.

## 5. Core experience

### First run: resolve to a REST operation or an A2A agent

1. Operator configures two approved capabilities: an OpenAPI-described billing GET and a separately running A2A research agent with an explicitly approved capability.
2. UtterRoute exposes a bounded A2A Agent Card/HTTP+JSON interface; callers authenticate and receive an eligible capability set.
3. A client sends an A2A `SendMessage` naming an approved capability or matching a configured deterministic rule. **No model call is needed.**
4. UtterRoute dispatches to **one** target, returns a Task artifact and supports authorized `GetTask`.
5. For a REST operation, it validates arguments and sends at most one GET; for a remote A2A agent, it sends at most one downstream `SendMessage` and maps the child task/context as needed.

When no deterministic match exists, UtterRoute clarifies/refuses by default. A later **optional Jev adapter** may select among eligible candidates after comparative tests demonstrate value. No TypeSafe key is needed for the v0.1 baseline.

### Task and failure behavior

- “Cancel invoice 42” → no REST write. A similarly named remote agent capability is eligible **only** if explicitly approved by operator policy.
- Missing invoice identifier → input-required task, no downstream dispatch.
- An unapproved downstream capability, unknown ID, conflicting candidate match or unavailable inference provider → refusal/failure; never guess a target or silently switch providers.
- A remote agent returning a working or input-required child task is correlated to the caller-owned UtterRoute task; child status is retrieved or clarified under bounded limits and existing A2A semantics.
- A2A tasks may have remote side effects despite the REST GET-only rule. Completion does not prove remote business success.
- Replayed input and unauthorized task access must not initiate a second downstream call.

## 6. API-facing behavior

**v0.1 public contract:** [A2A v1](https://a2a-protocol.org/v1.0.0/specification/) over `HTTP+JSON`:

- `GET /.well-known/agent-card.json` — discover the protocol-defined Agent Card `skills`/interface.
- `POST /message:send` — A2A `SendMessage`, returns a `Task` for an accepted request.
- `GET /tasks/{id}` — authorized `GetTask`, with bounded local retention.
- Task results live in **artifacts**, clarification in an input-required status message. A completed task indicates gateway completion, not domain/business postcondition success.

**Secondary custom HTTP adapter:** `POST /v1/execute` is **not part of v0.1**. See [future HTTP design](HTTP_API_FUTURE.md) and [A2A-first ADR](ADR-001-A2A-FIRST.md).

A2A's authorization and task security are separate from upstream API credentials. No A2A client message can provide trusted target URL, HTTP method, upstream credentials or an unrestricted operation ID. This remains a local/controlled v0.1 product, not a production-ready public agent service.

## 7. Success metrics and validation

The validation goal is **incremental selection value with low fast-path overhead**, not throughput or API gateway market share.

| Area | Release evaluation target (initial, to validate) |
| --- | --- |
| A2A conformance | An independent client discovers UtterRoute, sends messages to both target kinds, retrieves tasks and handles input-required continuation |
| Registration | Operator can configure one REST operation and one registered A2A capability without changing either target implementation |
| Task safety | Caller ownership, TTL, replay guards and bounded child-task polling; zero unauthorized downstream invocations in tests |
| Explicit/deterministic path | **Zero inference calls**; compare p50/p95 against direct invocation, with an initial goal of <10 ms additional in-process p95 overhead (network separately measured) |
| Semantic selection (optional later) | Compare held-out accuracy, abstention, cost and p50/p95 against deterministic baseline; never force inference for explicit selection |
| REST security | Zero non-GET requests and zero unauthorized origins in test fixtures |
| Agent authorization | Zero calls to unapproved remote A2A capabilities, and deny unknown side-effect risks by default |
| Failure behavior | Ambiguous or unavailable targets abstain; provider outages do not trigger hidden cloud fallback |
| Adoption hypothesis | Developer can discover and invoke the two registered capability types without writing a bespoke A2A bridge |
| Stop condition | If neither setup nor selection improves meaningfully over direct A2A or existing adapters, narrow to a reusable resolver library rather than expand into gateway features |

These are **hypotheses, not measured results**. The deterministic p95 goal is an in-process budget, not an end-to-end latency guarantee. Test direct target call, deterministic UtterRoute dispatch, and (later) semantic dispatch with the same fixtures.

## 8. Milestones (A2A first, resolver scope)

1. **A2A shell:** Pin Go + A2A SDK; Agent Card, SendMessage, GetTask, auth-aware bounded tasks, independent-client smoke test.
2. **Capability catalog:** Register one trusted REST GET and one approved remote A2A capability, with stable IDs, policy and deterministic selection.
3. **Two thin target adapters:** REST GET and outbound A2A SendMessage; child-task mapping, deadlines, refusal and replay tests.
4. **Measure the wedge:** Compare direct invocations vs deterministic path latency, correctness, setup friction and A2A interoperability; document whether a standalone service is justified.
5. **Optional semantic experiment:** Add a Jev adapter only after the baseline works; measure held-out improvement before promoting it to a required release feature.

Do **not** implement a generic HTTP gateway or proprietary `/v1/execute` API as a release prerequisite.

## 9. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| A2A protocol version drift or partial implementation | Pin an official specification/SDK, validate with an independent A2A client, advertise only supported binding/features |
| A2A task ownership, replay, and storage leaks | Authenticate callers, isolate task retrieval, cap retention, reject terminal-task continuation and duplicate execution |
| Jev may choose a plausible but wrong operation | Restrict choices to eligible catalog; explicit abstain; benchmark and calibrate thresholds |
| Jev is not an arbitrary structured-output generator | Keep parameter parsing/validation in code; abstain rather than fabricate unsupported data |
| API docs are incomplete or misleading | Allow a curated operation allowlist and human-edited descriptions; fail closed |
| SSRF and credential leakage from imported specifications | Pin outbound origins, block redirects to untrusted hosts, prevent arbitrary URLs/headers, and redact secrets |
| Imported spec or upstream response includes prompt-injection text | Treat descriptions/responses as untrusted data; never grant authority over policies |
| Provider cost, latency, outage or licensing changes | Provider interface, bounded timeouts, version pinning, no silent risky fallback |
| Scope drifts into generic API management | Enforce [ADR-003](ADR-003-CAPABILITY-RESOLVER.md)'s build/integrate/exclude filter; use existing ingress/API gateways |
| Remote A2A agents have side effects | Explicitly register and authorize registered A2A capabilities; unknown-effect delegation denied by default; one target per task |
| Mandatory semantic routing increases latency/cost | Keep explicit/deterministic resolution as zero-inference default; evaluate Jev separately |
| GTM becomes too broad | Validate two target types and measured capability-selection value before additional connectors |

## 10. Decisions still open

**Implementation language decision:** [ADR-002](ADR-002-GO-V0.1.md) proposes **Go** with the official `a2a-go/v2` SDK, standard `net/http`, and `kin-openapi`; the A2A-first product contract is unchanged.

- Exact pinned A2A Go SDK release and required Go 1.26+ toolchain patch for the first implementation PR.
- Whether the A2A plan-only artifact should expose raw argument values or a redacted preview by default.
- Which pinned A2A SDK version and data storage implementation to choose for the first local-only task store.
- Which OpenAPI schema features and remote references are supported in the first importer.
- Default routing-confidence policy; must be calibrated from fixtures rather than treated as a probability of business success.
- Which authenticated local development policy and token mechanism to use by default for A2A task isolation.

**Definition of done:** An independent A2A client discovers and invokes **one registered REST GET** and **one approved existing A2A agent**, receives caller-owned Tasks/artifacts, observes an input-required continuation and correct abstention; **explicit routing makes zero model calls**, disallowed requests make zero downstream calls, and no custom HTTP/API-management system is required. Benchmarks compare direct vs resolved dispatch.
