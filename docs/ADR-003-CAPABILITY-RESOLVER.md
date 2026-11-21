# ADR-003: Capability resolver, not a general API gateway

**Status:** Proposed for acceptance  
**Date:** 2026-11-21  
**Scope:** Product differentiation, v0.1 target model, and build-vs-integrate boundaries  
**Related:** [PRD](PRD.md) · [SPEC](SPEC.md) · [ADR-001: A2A-first](ADR-001-A2A-FIRST.md) · [ADR-002: Go](ADR-002-GO-V0.1.md)

## Context

A generic HTTP/API gateway is a mature, crowded category. For known REST endpoints or known A2A destinations, putting Jev/SystemOne on every call adds latency, cost and failure modes without necessarily improving outcomes. Existing infrastructure already solves TLS termination, generic proxying, ingress rate limiting, API-key administration, analytics and traffic management.

UtterRoute also needs to support **existing A2A agents behind the service**, not only OpenAPI REST operations. Its distinctive, testable problem is choosing **which registered, eligible capability** can fulfill an agent's request and safely dispatching to it.

REST-to-A2A wrapping by itself is useful, but is not a sufficient unique selling point. Treat semantic selection as an **optional, measurable enhancement**, not a mandatory per-request toll.

## Decision

**UtterRoute is an A2A-first capability registry and intent-based resolver across existing REST APIs and A2A agents, with thin, policy-checked dispatch. It is not a general-purpose HTTP/API gateway.**

- **Primary interface:** A2A v1 HTTP+JSON; keep [ADR-001](ADR-001-A2A-FIRST.md)'s Agent Card / SendMessage / GetTask contract.
- **Implementation:** Go; keep [ADR-002](ADR-002-GO-V0.1.md)'s SDK/tooling decision.
- **Target types:** First-class `rest_operation` (allowlisted GET from operator-configured OpenAPI) and `a2a_agent` (operator-approved local capability backed by an existing agent; its Agent Card is discovery metadata).
- **Routing policy:** Authorize/filter eligible capabilities, then prefer **explicit, authorized selection** or provably deterministic rules. Use Jev (or optional future SystemOne) **only if configured and needed** for ambiguous natural-language selection. If ambiguity remains and no trusted semantic router is configured, abstain/clarify; do not guess.
- **Dispatch:** One selected capability per incoming task; REST uses one validated GET, A2A uses a standard outbound A2A message and task correlation. No fan-out, chaining, arbitrary HTTP proxying or general agent orchestration.
- **Infrastructure:** Reuse an existing ingress/API gateway, edge DDoS/WAF, authentication provider and upstream API management where available. UtterRoute remains vendor-neutral and works behind a plain reverse proxy for a local demo.
- **Safety:** External infrastructure never replaces per-caller capability authorization, task ownership, bounded work, destination checks, response caps or credential isolation inside UtterRoute.

This ADR **refines the product/target and model-routing scope** in earlier documents; it does not overturn A2A-first delivery or the Go selection. Historical Jev-first text means *Jev is the first optional semantic adapter*, **not** that every call must invoke Jev or that a TypeSafe key is required for deterministic operation.

## Build / integrate / exclude

| Area | Decision | Why |
| --- | --- | --- |
| Registered capability descriptors for REST operations and approved A2A agent capabilities | **Build** | This is the product's common selection vocabulary |
| Caller-aware eligibility, explicit selection, deterministic routing, safe abstention | **Build** | Differentiating decision logic |
| Optional Jev/SystemOne routing and evaluation | **Build thin adapters** | Useful when selection is genuinely ambiguous, never on the compulsory fast path |
| One-target REST GET / A2A SendMessage dispatch and task correlation | **Build minimally** | Needed to demonstrate an actual outcome |
| Task/caller security, fixed egress, bounded sizes/timeouts, duplicate-execution guards | **Build minimal safeguards** | Application-level trust boundaries cannot be delegated |
| TLS, ingress/WAF, volumetric DDoS defense, generic rate limiting, access tokens / identity issuance | **Integrate** | Existing ingress/API management products do this better |
| Upstream REST auth, lifecycle and quotas | **Use existing upstream infrastructure** | Do not replace the API's established security model |
| Reverse proxying arbitrary HTTP methods/paths, API transformations, traffic splitting/load balancing | **Do not build** | Commodity API-gateway territory |
| Developer portal, API product management, monetization, metering/billing UI | **Do not build** | Commodity API-management territory |
| Automatic multi-agent execution graphs, generic tool-chain planning, runtime hosting of agents | **Do not build** | A separate orchestration platform |
| Vendor-specific Kong/Envoy/Apigee plugins | **Defer** | Plain interoperable A2A/HTTP interfaces first |

## Minimal capability model

Each registered capability has a stable ID, target kind, operator-approved description, side-effect/risk classification and eligibility policy. Endpoints, credentials and trust rules remain internal configuration, **not** generated by models.

```text
Capability:
  id: stable registered ID
  kind: rest_operation | a2a_agent
  summary: short operator-reviewed description
  target_ref: trusted config reference
  allowed_callers: policy reference
  risk: read_only | delegated_effects_unknown | explicitly_approved
```

- **REST:** Parsed from a local OpenAPI spec, normalized and restricted to allowlisted GET operations with schema-checked scalar arguments.
- **Remote A2A:** Registered as a locally named, operator-approved capability backed by a trusted A2A agent URL. The protocol-defined Agent Card `skills` entries are **untrusted descriptive metadata**, not individually invocable or enforceable permissions; `SendMessage` addresses the remote agent. The controlled v0.1 demo registers one approved local capability per remote agent target and never claims skill-level confinement. Remote agents may perform **writes or long-running side effects**; the GET-only REST guarantee does *not* extend to them. Unknown-effect delegation is denied by default until explicitly approved.
- **Explicit selection:** A caller may supply a documented capability hint, but it is untrusted input; the resolver verifies ownership/authorization and required parameters before dispatch.
- **Deterministic selection:** Only use unambiguous, configured rules. If several capabilities match, abstain or use configured semantic inference; never quietly choose the first.
- **Semantic inference:** Model input contains only approved eligible capability descriptions. Selection is checked against the same eligible set. Any provider failure yields abstention/error, never hidden backend fallback.

The `Resolver` knows *what* can be called. A small `Dispatcher` knows *how* to invoke the chosen target. Neither becomes a general traffic proxy.

### Remote task handling, deliberately bounded

The incoming task remains UtterRoute-owned. On outbound A2A calls, store a mapping to the permitted downstream task/context IDs, scoped to the authenticated caller. Return a completed result when the child completes, an input-required message when it needs clarification, or a working task if the child remains active and the selected A2A binding supports retrieval. Bound polling, concurrent child tasks and retention; do not blindly mirror an untrusted downstream task ID to a different caller. No v0.1 streaming, push, cancellation or multi-agent fan-out.

## Thin Go shape

```text
cmd/utterroute/
internal/
  a2a/          # inbound Agent Card/SendMessage/GetTask and task mapping
  resolver/     # policy-filtered capability selection; deterministic first
  catalog/      # REST operations + registered A2A agent capabilities
  targets/      # rest.go and a2a.go thin outbound clients
  config/       # approved targets, policy and secret references
examples/
  billing-api/
  research-agent/
```

Keep argument checks and execution authorization in the resolver/target path. Do not introduce interfaces, packages or persistence layers until the first vertical slice needs them. A separate native `/v1/execute` route remains deferred and must not define the core architecture.

## Product-validation spike and stop criteria

Demonstrate **one registered REST GET capability and one independently running A2A agent** behind UtterRoute, both exposed to a separate A2A client. Start with a fake/deterministic selector; validate semantic routing in a subsequent experiment rather than making the hosted model key mandatory.

Three comparisons use the same payloads/capability set:

1. **Direct invocation** of each registered target (baseline).
2. **UtterRoute direct/deterministic** dispatch with caller authorization (measure p50/p95 incremental overhead).
3. **UtterRoute optional semantic** selection (measure additional p50/p95 and per-request provider cost, plus selection correctness and abstentions).

Provisional **hypotheses, not benchmarked release claims**:
- The deterministic path adds little processing delay; target **under 10 ms additional p95 in-process processing**, measured separately from network overhead.
- Semantic selection must improve held-out ambiguity-resolution quality enough to justify latency and cost; measure rather than assume an acceptable universal threshold.
- A third-party A2A client can discover and use both kinds of capability without learning a proprietary invocation protocol.
- Invalid, denied and ambiguous requests dispatch to **zero** targets. Duplicate/replayed requests do not cause additional downstream execution.
- If registration is no simpler than bespoke A2A wrappers and selection quality fails to improve over a simple baseline, **stop or narrow to a reusable library**, rather than expand into API-management features.

## Contribution scope filter

A proposed feature belongs in UtterRoute only if it directly improves **discovery, selection quality, correct abstention, safe one-target dispatch, or A2A interoperability**, and cannot be delivered as effectively by configuration of existing infrastructure. Require a measurable acceptance test and a compelling reason to put any expensive step on the hot path.

For v0.1 explicitly **do not build**: generic reverse proxy, WAF, TLS gateway, API catalog portal, OAuth server, rate-limit product, distributed queue, streaming gateway, multi-agent orchestrator, automatic write-operation generator or vendor gateway plugins.

## Decision effects

- Update PRD and SPEC from REST-only gateway to an **A2A-first capability resolver**.
- Make Jev the first **optional** semantic router, not a runtime prerequisite for deterministic or explicit calls.
- Include an approved A2A target path in the initial interoperability spike, with strict limits.
- Scope the one-GET rule only to REST targets; for A2A targets use one-approved-delegation semantics and honest side-effect declarations.
- Integrate existing ingress/API-management infrastructure, without assuming it can enforce UtterRoute's application-level authorization.
- Product remains proposed; no implementation, adoption, latency or interoperability results are claimed.

## References

- [A2A v1 specification](https://a2a-protocol.org/v1.0.0/specification/)
- [ADR-001: A2A-first](ADR-001-A2A-FIRST.md)
- [ADR-002: Go](ADR-002-GO-V0.1.md)
- [SystemOne patterns](SYSTEMONE_PATTERNS.md)
