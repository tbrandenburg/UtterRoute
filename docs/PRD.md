# UtterRoute — Product Requirements Document (PRD)

**Release:** v0.1 (proposed)  
**Status:** Draft for discussion; no implementation is shipped  
**Last updated:** 2026-11-21  
**Related:** [Architecture](ARCH.md) · [Product scope ADR-003](adr/ADR-003-CAPABILITY-RESOLVER.md) · [Project README](../README.md)

## 1. Summary

UtterRoute is a self-hostable, MIT-licensed **A2A-first capability resolver** for existing REST APIs **and independently running A2A agents**. It registers operator-approved capabilities, selects one that an authenticated caller is allowed to use, and delegates through a small protocol-specific adapter. It is **not** a general API gateway, reverse proxy, agent runtime or API-management product.

**Promise:** *Discover capabilities, resolve intent, dispatch safely.* A2A messages in; one approved REST operation or downstream A2A agent task out.

**Fast by default:** When an eligible capability is explicitly selected or determined by an unambiguous configured rule, UtterRoute dispatches **without inference**. For ambiguous intent, Jev is the first **optional** semantic selector; SystemOne may be compared later. Without a suitable configured router, unresolved ambiguity leads to clarification/abstention. A TypeSafe key is **not required** for deterministic usage.

**Infrastructure boundary:** Use established gateways/ingress for TLS, DDoS/WAF, generic HTTP rate limiting and identity infrastructure. UtterRoute retains caller-to-task and caller-to-capability authorization plus minimal fixed-egress and workload safeguards. See [ADR-003](adr/ADR-003-CAPABILITY-RESOLVER.md).

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

### Product experience at a glance

An operator registers one approved REST GET capability and one independent A2A agent capability. A caller discovers UtterRoute through A2A and requests one eligible capability. **Explicit or unambiguous deterministic routing makes zero inference calls.** A caller-owned Task returns a bounded result, clarification or refusal. The detailed journeys and negative-case acceptance criteria follow.

For implementation details—wire messages, child-task correlation, parameter validation and fixed egress—see [ARCH](ARCH.md). The current two-target executable validation sequence is in [PLAN](PLAN.md).

### User journeys and acceptance scenarios

The journeys below describe **UtterRoute-owned responsibilities** and the boundary with existing APIs, A2A agents and infrastructure. They use **Capability**, **Target** and **Resolver** as product terms; `AgentSkill` and Agent Card `skills` remain A2A protocol fields, **not** per-capability authorization controls. The first two invocation journeys and a bounded A2A delegation are the v0.1 validation baseline; semantic selection is a separately evaluated extension.

#### Journey 1 — Platform engineer registers existing capabilities (P0)

**As a** platform engineer, **I want** an existing REST API and an independently deployed A2A agent to appear as approved capabilities **so that** a calling agent can use them without changing either target.

1. Configure the local OpenAPI billing service with fixed origin and an allowlisted GET `billing.list_invoices`.
2. Configure a trusted research-agent target with a locally named `research.competitor` capability, approved caller policy and explicit delegation/side-effect risk.
3. Validate both target configurations at startup. The A2A Agent Card is **descriptive discovery metadata**, not proof of trust or an enforcement mechanism for individual `AgentSkill` entries.
4. Expose a truthful UtterRoute Agent Card without advertising private caller-specific capabilities publicly. Apply caller eligibility independently at invocation.

**Acceptance:** A configured capability resolves to its trusted target; malformed REST schemas, disallowed methods, untrusted agent destinations and unknown-effect delegations fail closed. Neither underlying service is redeployed or modified.

#### Journey 2 — Finance assistant uses a known REST capability (P0 fast path)

**As a** finance agent, **I want** to list unpaid invoices **so that** I can consume a safe REST result through A2A without paying inference latency for a known route.

1. The agent sends A2A `SendMessage` with an authorized explicit capability hint (`utterroute.capability_id=billing.list_invoices`) or an unambiguous configured deterministic match.
2. UtterRoute authenticates the caller, checks capability eligibility and required arguments, and chooses `billing.list_invoices` **without Jev/SystemOne**.
3. The REST target adapter executes **at most one validated GET** to the configured origin; UtterRoute returns a caller-owned Task with a bounded artifact and the actual upstream status.
4. The agent retrieves the Task with `GetTask` if needed. An input-required clarification can supply a missing argument without causing a second GET.

**Acceptance:** Explicit/deterministic dispatch causes **zero inference calls** and no unauthorized URL, method or credential forwarding. “Cancel invoice” must not be reinterpreted as a harmless read just because the text contains “invoice”. Plan mode and invalid/ambiguous arguments trigger **zero target calls**. Benchmark incremental p50/p95 against direct REST invocation.

#### Journey 3 — Support agent submits ambiguous intent (baseline abstention; optional semantic extension)

**As a** support agent, **I want** to ask “Why hasn't this customer paid?” **so that** UtterRoute can choose the right approved capability *only when there is enough evidence*.

1. UtterRoute filters the catalog by the caller's policy **before** attempting selection.
2. If no unique deterministic match exists, v0.1 asks for clarification or refuses; it **does not** call Jev merely because the request arrived.
3. In a later opt-in semantic experiment, Jev/SystemOne can select **one** candidate from the eligible catalog or abstain. The resolver validates the returned capability ID and policy before dispatch.
4. Missing customer identifiers, conflicting matches, model outages and low-confidence/invalid selections lead to clarification/refusal, not guessed execution.

**Acceptance:** With semantic routing disabled, the ambiguous request causes **zero model calls and zero downstream calls**. If enabled later, only eligible candidates reach the model, a model-produced URL is never trusted, and correctness/abstention/latency/cost are measured against the deterministic baseline.

#### Journey 4 — Strategy agent delegates to a trusted A2A agent (P0 bounded delegation)

**As a** strategy agent, **I want** to request competitor research **so that** an existing specialist agent performs its own work while UtterRoute provides A2A interoperability and caller-scoped task visibility.

1. The caller explicitly selects the authorized local `research.competitor` capability; UtterRoute resolves the approved **agent target**, not an individually enforceable remote `AgentSkill`.
2. UtterRoute sends **one initial outbound A2A `SendMessage`** to the configured research agent. The remote agent decides its own internal steps; UtterRoute does not pick downstream sequences.
3. The remote agent may return a direct `Message`, a completed Task, or a bounded `WORKING`/`INPUT_REQUIRED` Task. UtterRoute maps that result to its own caller-owned Task and correlates child task/context identifiers where present.
4. If clarification is required, an authenticated caller can continue the **same** child task within supported limits. Remote task retrieval/polling is bounded and **not** triggered for every caller `GetTask` poll.
5. A timeout after submission has **uncertain delivery**: do not blindly retry the initial delegation or claim guaranteed exactly-once execution.

**Acceptance:** One accepted parent Task initiates **no more than one new remote delegation** (continuation of an existing child is distinct). Unauthorized callers cannot read or resume either task; an unapproved/unknown-risk agent target gets **zero calls**. Support a direct Message and completed Task in the first demo, plus a controlled input-required continuation. Full durable asynchronous processing, arbitrary remote authorization and restart recovery remain out of scope.

#### Journey 5 — Caller requests a multi-target workflow (P0 refusal boundary)

**As a** calling agent, **I want** to “Research Acme, then update the CRM” **so that** I understand whether UtterRoute can fulfill my request or whether I need an orchestrator.

1. The resolver checks whether a **single approved capability** can truthfully handle the whole intent.
2. If none can, UtterRoute declines to execute the composite request or asks the caller to narrow it.
3. The **calling orchestrator**, not UtterRoute or Jev, decides whether and when to send separate requests for research and CRM changes.

**Acceptance:** UtterRoute generates **no executable multistep plan**, no second capability selection based on intermediate results, no cross-target data passing and no unapproved REST writes. A downstream A2A agent may have its own internal workflow, but those actions remain outside UtterRoute's orchestration and authorization guarantees.

#### Cross-journey release guardrails

| Failure case | Expected outcome |
| --- | --- |
| Caller supplies an unauthorized capability ID or tries to read another caller's task | Deny without exposing private catalog/task data; **zero downstream calls** |
| Duplicate `SendMessage` or concurrent continuation | No second REST execution or second initial A2A delegation; safeguard replay/races |
| Agent Card changes its advertised metadata or redirects to another origin | Do not expand permissions or change trusted destinations automatically |
| Model returns an unknown capability or several deterministic matches remain | Abstain/clarify rather than choosing the first |
| REST response or A2A artifact is oversized | Enforce bounded body/artifact storage; fail with sanitized result |
| Remote A2A request times out after possible acceptance | Record an uncertain outcome; do not automatically replay remote work |
| Remote agent requests out-of-band authorization | Report unsupported/authorization-required handling; no credential-broker workflow in v0.1 |
| Downstream calls UtterRoute again, or client polls excessively | Enforce configured delegation/loop limits, task TTL and bounded polling; no recursive unbounded work |

**Scope rule:** One **selected capability** per incoming task: either one allowed REST GET or one initial approved A2A delegation. Multiple protocol messages for *the same* child task do **not** make UtterRoute a workflow orchestrator. Jev's only permitted decision, if enabled, is **one eligible capability ID or abstention**, never a sequence of calls.



## 6. Public interface and trust boundary

The **v0.1 public interface is A2A v1 over HTTP+JSON**, for discovery, `SendMessage`, and `GetTask`. Only approved capabilities are eligible for dispatch. An incoming A2A message cannot dictate arbitrary target URLs, HTTP methods or credentials. A REST target remains read-only; approved remote agents can have their own side effects.

The complete protocol/task/error contract—including plan mode, caller ownership, child task correlation and supported states—belongs in [ARCH](ARCH.md). The proprietary `POST /v1/execute` interface is explicitly deferred; the [archived sketch](archive/HTTP_API_FUTURE.md) is not an active requirement.

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

## 8. Current validation milestone

The authoritative **S0–S4 execution checklist, scope guardrails, fixtures, benchmarks and stop/go decision** are in [PLAN.md](PLAN.md). PRD tracks the *outcome*, not a second implementation checklist.

For v0.1, validate a Go A2A service with one registered billing REST GET capability and one independently running research A2A agent. Both must be invocable through a third-party A2A client, with caller isolation, no unintended target calls, safe refusal and **zero inference dependency** for explicit/deterministic selection.

**Decision gate:** Compare direct REST/A2A target invocation to the UtterRoute path for correctness, safety, p50/p95 and developer setup effort. Continue the standalone service only if it adds measurable value; otherwise consider a small reusable resolver library or stop. An optional Jev experiment can follow only after this baseline.

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
| Scope drifts into generic API management | Enforce [ADR-003](adr/ADR-003-CAPABILITY-RESOLVER.md)'s build/integrate/exclude filter; use existing ingress/API gateways |
| Remote A2A agents have side effects | Explicitly register and authorize registered A2A capabilities; unknown-effect delegation denied by default; one target per task |
| Mandatory semantic routing increases latency/cost | Keep explicit/deterministic resolution as zero-inference default; evaluate Jev separately |
| GTM becomes too broad | Validate two target types and measured capability-selection value before additional connectors |

## 10. Product questions to validate

- Is one registered REST capability plus one remote A2A agent useful enough to justify a standalone resolver?
- Does registration reduce integration work versus directly invoking known targets or writing a small A2A wrapper?
- Are safe abstention and policy-filtered discovery valuable enough to justify the extra hop?
- Should optional semantic selection be adopted only after a held-out benchmark proves improved outcomes?

The SDK/toolchain pin, schema subset, plan redaction, local identity policy and task-store implementation are **technical choices**, owned by [ARCH](ARCH.md) and the active [PLAN](PLAN.md), not repeated here.

**Definition of done:** An independent A2A client discovers and invokes **one registered REST GET** and **one approved existing A2A agent**, receives caller-owned Tasks/artifacts, observes an input-required continuation and correct abstention; **explicit routing makes zero model calls**, disallowed requests make zero downstream calls, and no custom HTTP/API-management system is required. Benchmarks compare direct vs resolved dispatch.
