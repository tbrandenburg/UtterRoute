# UtterRoute — Two-target validation spike

**Status:** Proposed, documentation only; no Go implementation has shipped.  
**Steering docs:** [PRD](PRD.md) (product scope and journeys) · [ARCH](ARCH.md) (technical contract) · [AGENTS](../AGENTS.md) (implementation instructions). This PLAN is the **one active S0–S4 execution checklist**; update it as slices are implemented, not the PRD.

**Decision basis:** [PRD](PRD.md) · [ADR-001: A2A-first](adr/ADR-001-A2A-FIRST.md) · [ADR-002: Go](adr/ADR-002-GO-V0.1.md) · [ADR-003: Capability resolver](adr/ADR-003-CAPABILITY-RESOLVER.md)

## Question to answer

**Does UtterRoute provide enough interoperability and safe, low-overhead capability selection to justify a standalone service rather than direct client calls, bespoke adapters, or a small resolver library?**

This spike is a **product and architecture experiment**, not an attempt to ship every v0.1 requirement. The only supported incoming business interface is A2A v1 HTTP+JSON. The deterministic path must run with **no Jev/SystemOne configuration or provider key**.

## Demonstration: two real targets, one resolver

```text
Independent A2A client
        |
        v
Existing ingress / loopback for spike
        |
        v
UtterRoute A2A adapter: identity + caller-owned tasks
        |
        v
Catalog + caller eligibility -> deterministic Resolver
        |
        +--> billing.list_invoices -> REST adapter -> configured billing GET
        |
        +--> research.competitor  -> A2A adapter  -> existing research agent

Returns a caller-owned Task + bounded artifact, or safe clarification/refusal.
```

Use **two independently running processes** as sample targets:

- **Billing REST API:** one OpenAPI-described, allowlisted `GET /invoices?status=unpaid`; fixed origin, no redirects; controlled scalar parameter `status`.
- **Research A2A agent:** one trusted remote A2A target with local UtterRoute capability ID `research.competitor`; operator-approved caller policy and delegation risk. The remote agent is independently deployable and has its own Agent Card. The A2A Agent Card `skills` entries are descriptive protocol fields, **not** granular invocable permissions.

**Selection:** For the experiment, a caller may supply the documented untrusted `message.metadata["utterroute.capability_id"]` hint. The resolver validates eligibility and dispatches only after policy checks. Optionally add **one clearly unambiguous configured rule** for the phrase “List unpaid invoices.” Do **not** implement general keyword matching that could misroute “cancel invoices.” If the capability is not uniquely determined, abstain/clarify.

## Implementation slices and explicit exit criteria

### S0 — Protocol and fixtures

**Deliver:** Initialize Go module and pin the tested Go toolchain and official `a2a-go/v2` release. Create a small A2A service with correct v1 HTTP+JSON Agent Card, `SendMessage`, `GetTask`, sanitized error behavior, and caller-identity-aware bounded Task store. Add the standalone billing fixture, standalone research-agent fixture and an independent A2A client test.

**Prove:** A separate client discovers the Agent Card, can create/retrieve its own Task and cannot read another caller's Task. Unknown/expired tasks fail safely. Local/controlled loopback only until auth and egress gates pass. No model provider integration.

### S1 — One deterministic REST capability

**Deliver:** Operator-configured catalog with stable `billing.list_invoices`, local OpenAPI subset and GET allowlist, policy-filtered explicit selection, schema validation and one bounded REST adapter. Return an A2A Task with a typed result artifact and real upstream HTTP status. Plan mode must do **zero downstream calls**.

**Prove:** Authorized request makes **exactly one** GET and **zero** inference calls; invalid arguments, denied capability hints, write intent, missing required values and ambiguous requests cause zero business calls. No client- or model-provided origin/method/credentials. No follow redirects; bound timeout and body size.

### S2 — One bounded A2A delegation

**Deliver:** Register `research.competitor` to a fixed trusted remote A2A agent target. Dispatch one initial A2A `SendMessage`; map direct `Message` and completed `Task` results into a caller-owned UtterRoute Task/artifact. Keep caller identity and credentials separate; correlate remote task/context only when returned.

**Prove:** Authorized selection makes **one initial downstream delegation** and zero inference calls. Denied/unapproved target causes zero messages. Handle a bounded `INPUT_REQUIRED` continuation of the **same** remote child task, if supported by the fixture; test direct Message, completed Task and controlled failure/timeout. `WORKING` may be represented only with explicitly bounded status retrieval; do not add durable background processing in this spike. A timeout with uncertain remote acceptance must **not** cause an automatic replay. No guarantee of exactly-once remote execution.

### S3 — Cross-cutting safety and refusal

**Deliver:** Minimal application-level admission/quotas and active-work bounds, task TTL/capacity, authenticated caller-to-capability and caller-to-task policy, fixed upstream origins, redirect/DNS/SSRF safeguards, bounded A2A child polling, no credential forwarding, and race-safe replay/continuation protection.

**Prove these cases:** (1) concurrent duplicate SendMessage produces no second REST execution or initial A2A delegation; (2) unauthorized caller cannot inspect or continue a Task; (3) unknown/ambiguous capability, invalid params and plan mode produce zero target calls; (4) oversized body/response, slow target, redirect and destination escape fail closed; (5) downstream `AUTH_REQUIRED` or recursive delegation is controlled/unsupported, not an invitation to build OAuth brokerage or a workflow engine; (6) overload rejects before model or target work begins.

**Boundary:** Reuse an external API gateway/reverse proxy for TLS, volumetric DDoS defense, WAF and generic ingress traffic control; UtterRoute must still enforce its own task and target authorization. The demo is **not** production-ready public ingress.

### S4 — Measure, compare and decide

Run all three execution paths on the **same configured targets**:

| Case | Baseline and measurements |
| --- | --- |
| Billing REST | Direct GET vs A2A-to-UtterRoute-to-REST; distinguish A2A adaptation/network overhead from resolver's in-process time |
| Research A2A | Direct independent A2A client-to-agent vs client-to-UtterRoute-to-agent; measure additional hop and task mapping cost |
| Resolution safety | Known explicit IDs, an unambiguous deterministic rule, conflicting/unknown/unauthorized IDs and composite requests; count correct dispatch, abstain, false dispatch and unwanted target calls |
| Developer setup | Document steps/config needed to register two targets, invoke via a third-party client and diagnose an error; compare against direct calls or a trivial bespoke wrapper |

**Publish actual data:** repeated trials with fixed environment, request counts, p50/p95 end-to-end and in-process resolver overhead where measurable, allocations/peak task count, authorization/refusal correctness, zero-inference verification, and documentation of limitations. Compare the direct A2A target to the delegated A2A path fairly; don't compare an HTTP GET against A2A semantics without labeling the protocol overhead.

**Provisional hypothesis:** under **10 ms incremental p95 in-process** deterministic resolver processing, *excluding* ingress, network and downstream execution. It is **not** a guaranteed benchmark or a blanket end-to-end latency target.

## Follow-up experiment: embedded Go SDK (`local_handler`; after S4)

**Not part of S0–S4 and not a prerequisite for the two-target decision gate.** Evaluate this separately only after the shared resolver, A2A server boundary and basic safety gates exist. If the S4 decision is to narrow/stop, reconsider whether an SDK-only library is the better product instead of automatically growing the standalone service.

**Question:** Does in-process typed function registration make an existing Go application meaningfully easier to expose to A2A callers than writing a small A2A wrapper or exposing functions via MCP?

### SDK-0 — Reuse the existing A2A core in-process

**Deliver:** A small importable Go package allowing a host application to mount UtterRoute's A2A handler and register one local `orders.lookup` capability. Keep the same catalog, caller policy, resolver and Task/Artifact lifecycle as the standalone service. Do **not** create a third network protocol.

**Prove:** An independent A2A client can discover and invoke the embedded Go service. A standalone UtterRoute may view it as a normal remote `a2a_agent` target, never as a cross-process `local_handler`. No new hosted model requirement.

### SDK-1 — One typed, read-only function and safe arguments

**Deliver:** A minimal registration API for a function conceptually shaped `func(context.Context, Input) (Output, error)`; stable capability ID, reviewed description, caller eligibility and effect policy. Support **structured caller-supplied input** with typed/schema validation; a developer-defined deterministic parser is optional after that baseline.

**Prove:** Explicit/deterministic routing selects the handler with **zero inference calls**. Missing/ambiguous/invalid data produces clarification/refusal with **zero handler calls**. No arbitrary method reflection, model-invented struct values, remote code registration or generated function-call sequences. Jev remains optional and can select one ID/abstain only.

**MCP Go SDK lesson to test:** Typed registration should infer and check **both** input and output JSON Schemas **before serving**, with clear errors for unsupported `any`/ambiguous types, duplicate capability IDs, invalid handler signatures and schema-generation failures. Reject unknown input properties under a strict default, while keeping A2A envelope/metadata outside the handler argument schema. Exercise required-vs-optional, absent-vs-null-vs-zero, enum/range refinements, and custom JSON marshaling. Do **not** copy MCP transport or its tool result envelope.

### SDK-2 — Security and local execution limits

**Deliver:** Caller authorization, input/output bounds, cooperative context deadline, concurrency cap, duplicate-task prevention and safe A2A artifact/error mapping.

**Prove:** Plan-only and unauthorized requests execute **zero** handlers; replay cannot execute the same admitted task twice. Slow and oversized handlers are bounded at the SDK boundary, with the explicit limitation that cancellation cannot forcibly kill a running Go function. Initial SDK supports **reviewed read-only synchronous functions only**; developers remain responsible for side effects, credentials and tenant data isolation in the host process.

**MCP-derived failure-path tests:** Validate and bound **serialized output** before emitting an A2A artifact (including `nil`, empty results, custom marshalers and oversized payloads); distinguish malformed A2A requests from handler validation/business errors; sanitize handler errors and recover panics per invocation. Freeze local registrations after startup; reject duplicate IDs and runtime replacement. Verify a `ReadOnly` annotation alone cannot bypass actual caller/domain authorization. No handler executes on invalid schema/input, denied task, plan-only request or unknown capability.

### SDK-3 — Compare and decide

**Measure:** Direct Go function invocation vs embedded A2A invocation (p50/p95, allocations, overhead); integration steps to expose one function through the SDK vs a hand-written A2A wrapper and a comparable MCP tool; correctness/abstention and no-unwanted-execution rates.

**Exit gate:** Keep `local_handler` only if one registered function can be exposed via an independent A2A client with less integration complexity and acceptable measurable overhead. Include a table of registration/schema failures, wrong-caller/unknown-field refusals, output-validation failures and panic recovery in the published test evidence. If the SDK merely wraps a function with no selection or interoperability benefit, narrow or drop it. Do not bundle semantic extraction or multi-agent orchestration merely to make the SDK seem useful.

**Reference basis:** [Official MCP Go SDK typed `AddTool` documentation](https://github.com/modelcontextprotocol/go-sdk/blob/main/docs/server.md), [registration/schema issue #731](https://github.com/modelcontextprotocol/go-sdk/issues/731) and [strict-property issue #892](https://github.com/modelcontextprotocol/go-sdk/issues/892). These motivate the **UtterRoute SDK's tests**; MCP is not a new transport/dependency.

**Shared invariant:** A single Task invokes **one authorized capability**—one REST GET, one initial remote A2A delegation, **or (embedded mode) one local Go handler**. No fan-out, chaining or executable model plans.

## Definition of done for the spike

- [ ] Pinned, independently verifiable A2A v1 HTTP+JSON server and client behavior (`Agent Card`, `SendMessage`, `GetTask`)
- [ ] Two separately running targets invoked using two configured local capability IDs
- [ ] One authorized REST GET or one initial approved A2A delegation **per incoming task**, not a chain
- [ ] Deterministic selection uses **zero Jev/SystemOne calls**, and needs no model key
- [ ] Caller ownership, target approval, argument validation, task/response bounds, no unsafe retries and failure handling tested
- [ ] Denied/ambiguous/plan-only requests cause **zero downstream target calls**
- [ ] Direct vs UtterRoute latency, correctness, refusal and setup-effort measurements published as reproducible fixtures
- [ ] Independent reviewer can run the demo from a fresh clone with documented commands/dependencies
- [ ] Written **continue / narrow to library / stop** decision with evidence, not assumed adoption

## Scope guardrails: avoid scope creep

**Included in the spike:** a minimal Go A2A adapter; one operator-configured OpenAPI GET; one trusted A2A agent; basic catalog/eligibility; explicit/deterministic resolver; tiny REST/A2A target clients; bounded in-memory Task store; required safety/benchmark tests.

**Deferred beyond the S0–S4 spike:** embedded Go SDK / `local_handler` (separate SDK-0–SDK-3 evaluation above); hosted Jev or SystemOne adapter; open-ended natural-language routing; broad OpenAPI features; generic proxying, WAF/TLS/API-management tooling, multi-agent plans, REST writes, agent OAuth brokerage, persistent/distributed task queue, arbitrary long-running recovery, A2A streaming/push/cancellation and custom `/v1/execute`.

**Optional Jev experiment only after S4:** On a labeled set of genuinely ambiguous requests, compare safe abstention/direct rules vs optional semantic resolution for correct selection, unsafe dispatch rate, p50/p95 and provider cost. Jev may choose **one eligible capability ID or abstain**; it may never choose a downstream call sequence. Implement only if the measured additional value is persuasive.

## Suggested small Go layout

```text
cmd/utterroute/main.go
internal/
  a2a/          # protocol boundary and caller-owned tasks
  resolver/     # caller policy + deterministic selection
  catalog/      # approved REST and A2A capabilities
  targets/      # rest.go and a2a.go
  config/       # fixed destinations, limits and secrets references
examples/
  billing-api/
  research-agent/
tests/integration/
docs/PLAN.md
```

Avoid a generic plugin system, separate worker service or new package for every abstraction. One narrow interface at the resolver boundary is enough for a later optional semantic adapter.

## Spike review questions

1. Is using existing APIs/A2A agents through one A2A surface easier than bespoke client-specific integration?
2. Does deterministic capability resolution add little overhead relative to the target itself?
3. Can we prove that unapproved, ambiguous and duplicate requests never cause accidental extra work?
4. Does the two-target path add value over simply having the caller invoke the known target directly?

**Stop or narrow** if the answers are weak. Do not broaden into generic API gateway or workflow orchestration features to justify the service.
