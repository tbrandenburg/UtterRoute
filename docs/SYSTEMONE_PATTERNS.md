# Reusable patterns from SystemOne

**Status:** Architecture reference, not an implemented integration  
**Upstream:** [codesoda/systemone](https://github.com/codesoda/systemone) (independent project, MIT license)  
**Applies to:** [UtterRoute v0.1 technical specification](SPEC.md)  
**Reviewed:** 2026-10-26

## Why this reference exists

[SystemOne](https://github.com/codesoda/systemone) provides a Jev-compatible HTTP server and interchangeable inference backends for typed decisions (Choice, Noul, Score). It does **not** import OpenAPI catalogs, resolve API business entities, check API permissions, or execute arbitrary REST operations. Those remain UtterRoute's responsibilities.

We can **reuse design patterns** from SystemOne without copying implementation code, embedding Rust crates, or making it a dependency. The first UtterRoute release remains a Jev-first, single-operation, allowlisted-GET gateway as documented in the [PRD](PRD.md).

## Patterns worth adopting

| Pattern | Verified upstream reference | UtterRoute application | Priority |
| --- | --- | --- | --- |
| **1. Provider-neutral decision interface** | [`DecisionHost` and `CallContext`](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/host.rs) | Define `Router.route(instruction, candidates, context)` returning a selected registered operation ID **or abstention**. Keep provider credentials out of call context. | **v0.1: Adopt the idea** |
| **2. Explicit backend capabilities and limits** | [`Capabilities` and validation](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/capabilities.rs) | Define per-provider capability metadata (Choice support, maximum choices, accepted state formats, model identity, confidence semantics) and reject unsupported jobs before inference. | **v0.1: Minimal; expand later** |
| **3. Typed HTTP contract and strict parsing** | [Jev-compatible JSON schema](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) and [wire parsing](https://github.com/codesoda/systemone/blob/main/crates/systemone-http/src/wire.rs) | Keep incoming request/response types stable. Validate all model response fields, chosen IDs and numeric ranges; never repair unexpected selections into apparently valid API calls. | **v0.1: Adopt** |
| **4. Explicit backends, pinning, no silent fallback** | [SystemOne sample configuration](https://github.com/codesoda/systemone/blob/main/examples/systemone.config.toml) and [backend contract](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/host.rs) | Put router provider/model/version, connection secrets and permitted inference destination in operator configuration. Never switch to hosted inference automatically when local inference fails. | **v0.1: Adopt** |
| **5. Bounded queues and deadlines** | [Resident worker registry](https://github.com/codesoda/systemone/blob/main/crates/systemone-http/src/registry.rs) and [server](https://github.com/codesoda/systemone/blob/main/crates/systemone-http/src/server.rs) | Enforce concurrency limits and deadlines for gateway requests and provider calls; distinguish overload, timeout and provider failure. Do not reproduce a complex resident model worker unless we actually host models. | **v0.1: Simple limits; advanced later** |
| **6. Safe hosted transport** | [Remote transport](https://github.com/codesoda/systemone/blob/main/crates/systemone-remote/src/transport.rs) | Fixed destinations, no redirects, bounded response bodies, per-call secrets, timeouts, no transparent retries, and no blindly forwarded client headers. Apply separately to model calls and business GET calls. | **v0.1: Adopt** |
| **7. Diagnostic metadata outside the model result** | [HTTP server headers](https://github.com/codesoda/systemone/blob/main/crates/systemone-http/src/server.rs) | Keep internal model version, provider, duration and routing evidence as sanitized metadata; preserve UtterRoute's documented plan/result/error schema. Do not leak prompts, credentials or upstream body. | **v0.1: Adopt** |
| **8. Compatibility tests distinct from quality evaluations** | [TypeSafe SDK compatibility smoke](https://github.com/codesoda/systemone/blob/main/compat/sdk-js/smoke.mjs) and [GLiNER2 evaluation](https://github.com/codesoda/systemone/blob/main/docs/gliner2-evaluation.md) | Test wire interoperability separately from operation accuracy, abstention quality, argument correctness and end-to-end API-call safety across backends. | **v0.1: Adopt evaluation philosophy** |
| **9. Accurate third-party attribution** | [LICENSE](https://github.com/codesoda/systemone/blob/main/LICENSE) and [third-party notices](https://github.com/codesoda/systemone/blob/main/THIRD_PARTY.md) | Keep UtterRoute MIT; attribute referenced projects. Verify any downloaded model weights separately; MIT project code does **not** license model artifacts or hosted API use. | **v0.1: Adopt** |

**Important:** These are architecture inspirations, not declarations that UtterRoute currently implements the behaviors. Source files and compatibility may change upstream; pin and test a chosen release before building against it.

## Concrete mapping to UtterRoute

### Pattern A — Routing is a typed decision, not a generated HTTP call

SystemOne's [neutral `DecisionHost`](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/host.rs) separates typed inference from transport and model ownership. UtterRoute should use a smaller analogous interface:

```python
# Interface sketch only; not implemented.
def route(instruction, allowed_candidates, request_context) -> RoutingDecision:
    # operation_id: registered candidate ID or None
    # score: provider score, not verified correctness
    # reason: selected / no_match / unavailable / unsupported
    ...
```

Only the **allowlisted** OpenAPI operations are ever passed in as candidates. A decision is an untrusted *proposal*; the gateway rechecks the operation method, parameters and configured destination.

### Pattern B — Respect Jev-compatible Choice limits

The [published wire subset](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) caps a single Choice question at **16 criteria**. Reserve `none_of_the_above` for abstention, leaving a maximum of **15** real operation candidates when using that subset.

Illustrative request to a separately configured SystemOne server:

```http
POST /v1/systemone
Content-Type: application/json
```

```json
{
  "state": "List unpaid invoices",
  "questions": {
    "operation": {
      "type": "choice",
      "instructions": "Select the best permitted API operation, or none if no operation matches.",
      "criteria": {
        "billing-demo:listInvoices": "List invoices, optionally filtered by status",
        "billing-demo:getInvoice": "Retrieve a single invoice by known invoice ID",
        "none_of_the_above": "No eligible operation safely matches the request"
      }
    }
  }
}
```

This is a **decision request only**. A valid answer still requires independent parameter extraction, schema checks, origin restrictions, and the configured execution policy. For more than 15 eligible operations, **do not silently truncate the catalog**. First build and evaluate a candidate-retrieval stage, or return a controlled failure until it exists. The subset may differ from individual hosted model limits; each adapter must declare and test its own constraints.

### Pattern C — Capabilities and confidence are not interchangeable

SystemOne's [capability metadata](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/capabilities.rs) describes supported primitives, option caps and how confidence values should be interpreted. Its [GLiNER2 evaluation](https://github.com/codesoda/systemone/blob/main/docs/gliner2-evaluation.md) shows that apparently compatible backends can perform very differently depending on the task.

For UtterRoute:

- Reject unsupported request shapes rather than letting inference return a plausible but wrong answer.
- Log configured backend/model version and capability profile for evaluations.
- Treat provider scores as **model-specific ranking signals**, not calibrated probabilities that the chosen business API call will succeed.
- Do not reuse one confidence threshold across all models unless validated per backend.

### Pattern D — Fail-closed transport, separately for inference and business APIs

SystemOne's [hosted transport](https://github.com/codesoda/systemone/blob/main/crates/systemone-remote/src/transport.rs) uses one send, no transparent fallback, no redirects, bounded response reads and per-request authentication rather than forwarding client credentials.

For UtterRoute use two separate trusted destination sets:

1. **Inference:** Only the configured TypeSafe or future SystemOne inference endpoint may receive decision requests, subject to provider-specific data disclosure.
2. **Business API:** Only origins and GET operations explicitly configured and allowlisted by the developer.

The inference result must never become a URL, HTTP method or credential source. Preserve the stricter [UtterRoute SSRF and DNS-rebinding requirements](SPEC.md#8-security-and-network-execution); SystemOne's own transport policy is not a replacement for them.

### Pattern E — Separate wire conformance from task correctness

Follow two independent testing layers:

| Layer | Tests |
| --- | --- |
| **Adapter contract** | Serialize Choice correctly; validate response shape and IDs; honor candidate cap; handle 429/timeout/malformed JSON; never leak secrets or silently switch provider. |
| **Business execution** | Correct operation and scalar arguments; abstain on ambiguity; no writes; `plan` sends zero upstream business calls; `execute` sends at most one approved GET; compare results on held-out cases. |

A successful HTTP request or passing SDK smoke test proves **wire compatibility**, not equivalent answer quality. Quality needs a labeled UtterRoute dataset and end-to-end tests.

## Where *not* to reuse SystemOne

- **No embedded Rust inference runtime in the initial Python service.** If desired later, call SystemOne through its documented HTTP interface as an independently deployed optional service.
- **No assumption that local backends are TypeSafe Jev weights.** They are distinct models with separate capabilities, performance and weight licenses.
- **No local-to-cloud fallback by surprise.** Explicit provider selection and data-egress disclosures remain mandatory.
- **No dependency on SystemOne for OpenAPI ingestion, parameter resolution, policy enforcement or HTTP business execution.** It is the decision layer, not the API gateway.
- **No premature feature expansion** to writes, MCP, multi-agent planning or persistent inference workers just because SystemOne supports multiple backends.

## Recommended adoption sequence

1. **Now (foundation):** Define neutral `Router` types, configuration/secret boundaries, response validation, error mapping and evaluation fixtures. Keep a deterministic fake router for integration tests.
2. **v0.1 baseline:** Implement a direct hosted Jev adapter and the read-only single-operation `plan`/`execute` path. Verify refusal and security invariants.
3. **Optional follow-up:** Add an explicit `systemone-http` adapter as a separately configured endpoint. Reuse the Jev-compatible schema, and contract-test it against a pinned SystemOne release.
4. **Comparative evaluation:** Run the same held-out UtterRoute requests through direct Jev and selected SystemOne backends. Measure end-to-end correctness, abstention rate, request latency, token/usage costs and unintended business calls; decide whether offering this optional backend is worthwhile.

## Attribution and scope

SystemOne is independently developed by codesoda and [MIT-licensed](https://github.com/codesoda/systemone/blob/main/LICENSE). This document links to its patterns but does not copy source code. UtterRoute is not affiliated with or endorsed by SystemOne, TypeSafe AI or the other backend providers. Downloaded model artifacts and hosted services retain their own terms.

**Decision:** Reference reusable patterns now; make SystemOne interoperability an optional, separately validated future implementation—not a mandatory v0.1 dependency.
