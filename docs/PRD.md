# UtterRoute — Product Requirements Document (PRD)

**Release:** v0.1 (proposed)  
**Status:** Draft for discussion; no implementation is shipped  
**Last updated:** 2026-11-21  
**Related:** [Technical specification](SPEC.md) · [Project README](../README.md)

## 1. Summary

UtterRoute is a self-hostable, MIT-licensed **A2A-first natural-language gateway for existing REST APIs**. Developers register an OpenAPI-described service; other agents discover UtterRoute's business capabilities through an Agent Card and send natural-language instructions using A2A. UtterRoute returns retrievable task outcomes with validated read-only API results, plans, or clarification/refusal.

**Promise:** *Make REST APIs callable by agents.* A2A messages in; validated, traceable API results out. The custom `POST /v1/execute` HTTP interface is secondary, planned after v0.1.

For v0.1, use a **Jev-first, provider-pluggable** selection layer. Jev selects among permitted operations; deterministic code handles schema validation, authorization, argument checks, and HTTP execution. The router must abstain when it cannot establish a safe, unambiguous call. Jev is a hosted proprietary model; UtterRoute's own code remains MIT-licensed. A TypeSafe API key is required for the first Jev adapter, and third-party data handling must be documented.

## 2. Problem and customer

### Problem

An OpenAPI description documents the syntax of an API but does not let application builders invoke it reliably with ordinary language. LLM tool calling can work, yet often requires bespoke prompts, orchestration, credentials handling, validation, and failure management per API.

### Initial users

- **Primary:** Developers embedding natural-language read operations into a vertical SaaS product or internal tool.
- **Secondary:** API/platform engineers testing a natural-language interface to an existing REST API.
- **Not the v0.1 buyer:** Nontechnical end users looking for a general chatbot or autonomous agent platform.

### Job to be done

> When my team already operates a documented REST API, I want to expose a constrained, discoverable A2A capability that other agents can call, without building bespoke agent orchestration or surrendering control of API execution.

## 3. Product principles

1. **Safety over apparent cleverness:** Do not guess missing identifiers, invent values, or silently execute a different operation.
2. **Deterministic enforcement:** An AI selection never bypasses allowed operations, identity/policy, or schema validation.
3. **Explainability by default:** Return the operation, arguments, status, and a request ID. Keep sensitive detail out of logs.
4. **Open core in full:** Local/self-hosted mode, routing, validation, and tracing are open source; no artificial paywall on safety.
5. **Model independence:** Jev is the first adapter, not the external API or a permanent architectural requirement.
6. **Small and composable:** Build a gateway, not a full conversation agent, workflow builder, or tool marketplace.

## 4. v0.1 scope

### In scope (P0)

| ID | Requirement | Acceptance criteria |
| --- | --- | --- |
| PRD-01 | Import local OpenAPI 3.x | A local spec is ingested and validated at startup; malformed specs fail clearly |
| PRD-02 | Normalized operation registry | Stable operation IDs, descriptions, scalar path/query schemas and HTTP metadata |
| PRD-03 | Allowlisted read execution | Only explicitly enabled **GET** operations can execute; fixed configured origins |
| PRD-04 | **A2A Agent Card (primary)** | Serve valid A2A v1 Agent Card with truthful skill, supported `HTTP+JSON` interface and security declaration |
| PRD-05 | **A2A `SendMessage`** | `POST /message:send` accepts text instructions under authenticated local/controlled deployment; no custom wrapper required |
| PRD-06 | **A2A tasks and retrieval** | A2A-compatible `Task` for every accepted request; `GET /tasks/{id}` returns authorized task state and result; temporary bounded TTL store |
| PRD-07 | **Input-required continuation** | Missing/ambiguous parameters produce an input-required task; authorized follow-up `SendMessage` can resolve it without executing an extra operation |
| PRD-08 | Jev-first intent routing | Select an eligible operation or abstain; never produce arbitrary request URLs |
| PRD-09 | Scalar argument resolution and validation | Support constrained path/query scalar values; refuse ambiguous/unsupported required fields |
| PRD-10 | Read-only execution and result artifacts | At most one business GET per task; bounded upstream result and actual HTTP status appear in an A2A artifact |
| PRD-11 | Optional plan mode | A2A message metadata `utterroute.mode=plan` returns a plan artifact; no upstream business API call |
| PRD-12 | Safe task/error mapping | Denial, no-match, provider failures and upstream failures map to proper task states or protocol errors without guessing |
| PRD-13 | Security and task isolation | Authenticated caller-to-task ownership, no credential forwarding, fixed outbound origins, redirects off, size/timeout limits |
| PRD-14 | Local developer experience | One reproducible A2A client-to-mock-billing demo with documented setup and configuration |
| PRD-15 | Evaluation and observability | A2A wire conformance, router correctness, abstention, task state, sanitized request IDs/timings and no-side-effect tests |

### Deferred / potential v0.2 features

- **Secondary custom HTTP adapter:** `POST /v1/execute` using the same core service; see [HTTP_API_FUTURE.md](HTTP_API_FUTURE.md).
- Deterministic shortlisting before Jev selection for larger catalogs.
- Optional CLI command to inspect imported operations.
- Additional routing engines via the same typed router interface.

### Explicitly out of scope

- Arbitrary POST/PUT/PATCH/DELETE calls, and write approvals or transaction rollback.
- Chaining multiple API operations, autonomous plans, stateful agents, or entity lookup across endpoints.
- GraphQL, MCP server ingestion, voice input, and browser/UI automation.
- A2A streaming, push notifications, cancellation, multi-instance durable task storage, and additional A2A protocol bindings (JSON-RPC/gRPC).
- The custom `POST /v1/execute` adapter in the initial release.
- Automatic OAuth connection setup, hosted multi-tenant control plane, visual UI, billing, or managed cloud.
- General arbitrary JSON-body synthesis, arbitrary string extraction, or guaranteed resolution of natural-language references.
- Claims of guaranteed correctness, end-to-end subsecond performance, or fully offline Jev inference.

## 5. Core experience

### First run: agent-to-API

1. Developer configures one OpenAPI-described read-only REST service, an explicit GET allowlist and a trusted origin.
2. UtterRoute exposes an A2A v1 Agent Card with a bounded read-API skill and an HTTP+JSON interface.
3. An A2A client sends `SendMessage` with text such as “List unpaid invoices.”
4. Jev selects the permitted operation and constrained arguments; code validates them and issues **at most one GET** using operator-configured credentials.
5. UtterRoute returns an A2A `Task` with a result artifact containing the operation, parameters, upstream HTTP status and bounded body. The client can retrieve the task via `GetTask`.

A2A requests default to **read-only execute** when eligible. For a no-upstream-call preview, the caller uses the optional, explicitly documented `message.metadata["utterroute.mode"] = "plan"`. No write operations are enabled.

**Important:** The example assumes the registered API exposes a `status=unpaid` parameter. No cross-API entity resolution is implied.

### Task and failure behavior

- “Cancel invoice 42” → task rejected for unsupported write; no business API call.
- “Fetch the invoice for Alex” without a unique identifier → input-required task asking for the identifier; no business API call until safely resolved.
- Unknown capability → task rejected or protocol error as appropriate; no guessed endpoint.
- Provider unavailable → task failed with sanitized diagnostic; no alternative provider selected silently.
- Follow-up `SendMessage` with matching `taskId` and `contextId` can continue an input-required task, but cannot restart terminal tasks or bypass caller authorization.

## 6. API-facing behavior

**v0.1 public contract:** [A2A v1](https://a2a-protocol.org/v1.0.0/specification/) over `HTTP+JSON`:

- `GET /.well-known/agent-card.json` — discover declared skills/interface.
- `POST /message:send` — A2A `SendMessage`, returns a `Task` for an accepted request.
- `GET /tasks/{id}` — authorized `GetTask`, with bounded local retention.
- Task results live in **artifacts**, clarification in an input-required status message. A completed task indicates gateway completion, not domain/business postcondition success.

**Secondary custom HTTP adapter:** `POST /v1/execute` is **not part of v0.1**. See [future HTTP design](HTTP_API_FUTURE.md) and [A2A-first ADR](ADR-001-A2A-FIRST.md).

A2A's authorization and task security are separate from upstream API credentials. No A2A client message can provide trusted target URL, HTTP method, upstream credentials or an unrestricted operation ID. This remains a local/controlled v0.1 product, not a production-ready public agent service.

## 7. Success metrics and validation

The primary metric is **correct, safe end-to-end execution**, not the router's top-1 classification rate.

| Area | Release evaluation target (initial, to validate) |
| --- | --- |
| A2A conformance | A pinned v1-compatible A2A client discovers the Agent Card, invokes SendMessage, retrieves Task and continues input-required Task |
| Task safety | Caller cannot retrieve another caller's task; expired tasks disappear; terminal tasks reject replay; no unauthorized business call |
| Operation routing | ≥95% on a held-out set of supported, unambiguous single-step cases |
| Full request success | ≥90% on the same supported cases, including parameter correctness and API result |
| Unsafe side effects | Zero non-GET requests; zero unauthorized destinations in integration/security tests |
| Ambiguous requests | Abstain or request clarification rather than execute the wrong operation |
| Latency | Report p50/p95 independently for routing, parameter resolution and end-to-end gateway overhead; compare against a small-LLM baseline |
| Developer setup | A fresh developer can use a documented A2A client to discover, query, and retrieve a mock-billing task within 10 minutes once dependencies/keys are available |

Targets are hypotheses, not measurements. Benchmark samples must include paraphrases, out-of-domain requests, missing parameters, swapped operations, provider failures, adversarial instructions, and identical inputs with reordered routing candidates. Evaluate on fixtures with known ground truth and preserve regression data.

## 8. Milestones (A2A first)

1. **A2A protocol skeleton (Go):** Pin Go toolchain and official `a2a-go/v2` release, implement Agent Card, `SendMessage`, `GetTask`, and conformance fixtures.
2. **Task lifecycle and isolation:** Bounded single-instance task store, TTL, caller authorization, input-required continuation and terminal-state handling.
3. **API gateway core:** OpenAPI importer, allowlist, mock billing API, safe network boundaries, deterministic fake router.
4. **Jev and execution:** Jev adapter, scalar resolution, safe plan/read-only GET execution, artifact/task mapping and traces.
5. **Release gate:** Interoperability demo using an external A2A client, integration/security tests and end-to-end evaluation.

The native `POST /v1/execute` HTTP adapter comes **after** the first A2A-first release; it is not a dependency of the A2A core.

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
| GTM becomes too broad | Begin with one use case and a small set of business APIs; validate with embedding developers |

## 10. Decisions still open

**Implementation language decision:** [ADR-002](ADR-002-GO-V0.1.md) proposes **Go** with the official `a2a-go/v2` SDK, standard `net/http`, and `kin-openapi`; the A2A-first product contract is unchanged.

- Exact pinned A2A Go SDK release and required Go 1.26+ toolchain patch for the first implementation PR.
- Whether the A2A plan-only artifact should expose raw argument values or a redacted preview by default.
- Which pinned A2A SDK version and data storage implementation to choose for the first local-only task store.
- Which OpenAPI schema features and remote references are supported in the first importer.
- Default routing-confidence policy; must be calibrated from fixtures rather than treated as a probability of business success.
- Which authenticated local development policy and token mechanism to use by default for A2A task isolation.

**Definition of done:** A new contributor can discover UtterRoute via an A2A Agent Card, use an independent A2A client to send a read-only request and retrieve a Task with structured API result, continue an input-required Task, and demonstrate that plan-only/ambiguous/denied requests make **zero** business calls; the custom native HTTP interface is not required.
