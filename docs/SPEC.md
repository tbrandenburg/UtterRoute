# UtterRoute v0.1 — Technical Specification

**Status:** Proposed, not implemented  
**Version:** 0.1-draft  
**Date:** 2026-10-18  
**Product scope:** [PRD.md](PRD.md)

## 1. Architecture and invariants

UtterRoute converts **one natural-language instruction** into **at most one allowlisted GET operation** described by an OpenAPI 3.x document. It returns a validated plan, performs a read-only call, or abstains. v0.1 is not a general agent or workflow engine.

**Hard invariants:**

1. Outbound targets come only from operator-controlled connection configuration; never from the prompt, a model, or an OpenAPI server URL.
2. Only explicitly allowlisted GET operations may execute. No writes or implicit multistep calls.
3. Required arguments must be validated; missing/ambiguous values cause clarification, never guesses.
4. `plan` must make **zero upstream business API calls**; `execute` at most one.
5. The model is a selector, not an authority. Policy checks run regardless of model confidence.
6. Routing failures fail closed, with no automatic unsafe fallback.
7. An upstream HTTP response is not a guarantee of business correctness.

```text
POST /v1/execute
     |
     v
Request validation and registered connection
     |
     v
OpenAPI operation registry + explicit GET allowlist
     |
     v
Candidate selection -> Jev router (choice + abstain)
     |
     v
Argument resolution -> Schema and policy validation
     |
     +---- plan ----> Validated plan, no upstream call
     |
     +---- execute -> Fixed-origin GET -> Structured result
```

## 2. Component contracts

- **OpenAPI importer:** Parses a local OpenAPI file at startup, resolves supported local references, builds operation and scalar-parameter schemas, rejects unsupported/invalid inputs.
- **Registry:** Immutable initialized collection of named connections, operation IDs, trusted origins, descriptions, method policy and authentication references.
- **Candidate selector:** Starts with eligible operations; can optionally shorten candidates deterministically for larger catalogs. It must preserve a no-match route.
- **Router adapter:** `route(instruction, candidates, context) -> RoutingDecision` where `RoutingDecision` contains only an allowed operation ID or abstain, a score (if available), engine/version and reason. Jev is the first adapter; don't assume generic JSON generation.
- **Argument resolver:** Parses explicit scalar path/query values and can use Jev for bounded enum selection. Tracks missing, conflicting or unsupported arguments.
- **Compiler:** Converts operation ID and parameters into a validated internal `CompiledPlan`; never takes method, origin, secrets or headers from router output.
- **Policy guard:** Enforces allowlist, configured origin, method, argument schema and execution mode.
- **Executor:** Issues one bounded GET with configured upstream credentials; no redirects or automatic retry in v0.1.
- **Telemetry:** Reports request IDs, stage durations, sanitized selected operations and outcomes.

## 3. Configuration and runtime

**Proposed implementation:** Python 3.11+, FastAPI, an HTTP client with strict timeouts, and Docker Compose. This is a recommendation, not existing code or a binding maintainer decision.

Configuration sketch (illustrative, not executable today):

```yaml
server:
  bind: 127.0.0.1
  port: 8080
  max_request_chars: 2000

routing:
  provider: jev
  model: jev-version-pinned-by-operator
  timeout_ms: 2500

connections:
  - id: billing-demo
    openapi_file: ./examples/billing/openapi.yaml
    origin: http://billing-demo:9000
    allowed_operation_ids: [listInvoices, getInvoice]
    auth:
      type: bearer_env
      env: BILLING_DEMO_TOKEN

execution:
  allow_methods: [GET]
  timeout_ms: 5000
  max_response_bytes: 1048576
  follow_redirects: false

logging:
  include_request_text: false
  include_response_body: false
```

Provider API keys are supplied separately via environment/secret management, not in client requests. Default listener binds to localhost; public deployment without gateway-level authentication, TLS, hardened network egress and tenant policy is **not supported by the v0.1 demo**. Where necessary, operators provide those protections outside UtterRoute until later versions implement them.

## 4. OpenAPI ingestion contract

Input: one or more locally configured OpenAPI 3.0/3.1 YAML or JSON documents.

**v0.1 supported subset:**

- `GET` operations with stable `operationId` and optional summary/description.
- Path/query parameters with scalar types `string`, `integer`, `number`, `boolean`; enums and ISO-8601 date strings.
- Required/optional fields, simple literal defaults and local `#/components/...` references.

**Rejected/deferred:**

- External/remote `$ref` fetches, implicit discovery of API hosts, dynamic destinations.
- Non-GET operations, request bodies, form uploads, callback/webhook flows and model-created headers.
- Complex recursive/nested parameter schemas, unsupported `oneOf`/`anyOf` variants or serialization styles.
- Duplicate operation IDs in a connection, unresolved references and invalid path schemas.

Canonical internal IDs use `connection_id:operationId`. Imported summaries/descriptions assist routing but are untrusted text and may not change execution policy. The trusted connection configuration, not the OpenAPI `servers` field, controls outbound origins.

## 5. HTTP API (proposed)

### `GET /healthz`

Reports local process liveness; does not contact the Jev provider or upstream API.

### `POST /v1/execute`

Content type: `application/json`.

```json
{
  "connection_id": "billing-demo",
  "request": "List unpaid invoices",
  "mode": "plan"
}
```

| Field | Required | Behavior |
| --- | --- | --- |
| `connection_id` | Yes | Exact name of a preconfigured connection, not a URL |
| `request` | Yes | Nonempty natural-language instruction within configured size bounds |
| `mode` | Yes | `plan` or `execute`; absence must not default to execution |

#### Example: plan response (`200`)

```json
{
  "status": "planned",
  "request_id": "req_example1",
  "connection_id": "billing-demo",
  "operation": {
    "operation_id": "listInvoices",
    "method": "GET",
    "path_template": "/invoices"
  },
  "arguments": {
    "path": {},
    "query": { "status": "unpaid" }
  },
  "execution": { "eligible": true, "will_call_upstream": false },
  "routing": { "engine": "jev", "score": 0.92 }
}
```

The example assumes a registered API with `GET /invoices` and supported `status` enum. `score` is a model selection score, **not** an independently calibrated probability of request correctness. Preview values may be redacted if the deployment marks a parameter sensitive.

#### Example: execute response (`200`)

```json
{
  "status": "completed",
  "request_id": "req_example2",
  "connection_id": "billing-demo",
  "operation": {
    "operation_id": "listInvoices",
    "method": "GET"
  },
  "result": {
    "upstream_status": 200,
    "content_type": "application/json",
    "body": {
      "invoices": [{ "id": "inv_1", "status": "unpaid" }]
    }
  },
  "duration_ms": 231
}
```

A completed gateway call is not a claim that domain-level postconditions have been verified. Non-2xx upstream **business** statuses should be represented in `result.upstream_status` with a clear completion status; do not confuse them with routing failures. Upstream responses must be type/size bounded; binary payloads are out of scope.

#### Example: clarification (`422`)

```json
{
  "status": "needs_clarification",
  "request_id": "req_example3",
  "error": {
    "code": "MISSING_PARAMETER",
    "message": "A required invoice identifier could not be resolved.",
    "fields": ["invoice_id"]
  }
}
```

#### Error and status codes

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | `INVALID_REQUEST` | Invalid JSON, empty text or bad mode |
| 401 | `UNAUTHENTICATED` | Optional external/internal gateway auth denied |
| 403 | `OPERATION_NOT_ALLOWED` | Forbidden/non-GET/unallowlisted operation |
| 404 | `UNKNOWN_CONNECTION` | Unknown or invisible connection |
| 422 | `NO_MATCH` | No trustworthy operation |
| 422 | `MISSING_PARAMETER` | Unresolved required value |
| 422 | `AMBIGUOUS_ARGUMENT` | Conflicting plausible parameter values |
| 422 | `UNSUPPORTED_ARGUMENT` | Input schema/extraction beyond v0.1 |
| 429 | `RATE_LIMITED` | Quota or rate policy applied |
| 502 | `UPSTREAM_ERROR` | Upstream transport/protocol failure |
| 503 | `ROUTER_UNAVAILABLE` | Routing provider unavailable; nothing executed |
| 504 | `UPSTREAM_TIMEOUT` | Upstream timed out |
| 500 | `INTERNAL_ERROR` | Sanitized unexpected server error |

Non-success payloads use `status: rejected`, `needs_clarification`, or `failed`, plus `request_id` and structured `error`. Never include stack traces, credentials, raw model prompts or private request bodies in public errors.

## 6. Routing logic

1. Find the configured connection and filter operations by explicit allowlist and GET method.
2. Prepare a bounded set of operation labels/descriptions; include explicit `NONE_OF_THE_ABOVE` or equivalent abstention candidate.
3. Ask Jev to select an eligible operation using its bounded-choice mechanism. Send no upstream API secrets, headers or business response bodies.
4. Reject any operation ID not in the supplied set, even if the model produces a plausible endpoint.
5. Abstain when no match, contradictory intent, provider error, or low confidence under a **calibrated** policy.
6. Record model/adapter version, candidate-set identifier and sanitized timing metadata.

Thresholds should be selected from evaluation data, not hardcoded as a universal guarantee. Evaluate sensitivity to candidate ordering, phrasing, and schema-description changes. Model scoring does not replace checking the final operation and arguments.

### Optional SystemOne interoperability (reference)

[SystemOne](https://github.com/codesoda/systemone) provides an independent, MIT-licensed Jev-compatible decision server across multiple local/hosted backends. For a **future optional adapter**, UtterRoute could call its `POST /v1/systemone` over HTTP using the [published compatible wire schema](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json). SystemOne is **not** an API/tool executor; OpenAPI operation filtering, parameter validation, destination policy and upstream execution remain inside UtterRoute.

### Reusable upstream patterns for implementation

SystemOne's design provides several patterns we can adapt **without adding SystemOne as a dependency**:

| Pattern to reuse | How it maps to UtterRoute | Upstream source |
| --- | --- | --- |
| Typed provider-neutral decision contract | Keep `Router` adapters replaceable; return only eligible operation IDs or abstain | [DecisionHost](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/host.rs) |
| Capability-declared inference limits | Check primitives, candidate count, model identity and score semantics before routing | [Capabilities](https://github.com/codesoda/systemone/blob/main/crates/systemone-core/src/capabilities.rs) |
| Strict Jev-compatible wire contracts | Validate selection responses and reject unknown operation IDs | [Wire schema](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) |
| Bounded, fail-closed service and transport | Use explicit timeouts/admission, fixed origins, no redirects, no silent retries | [Registry](https://github.com/codesoda/systemone/blob/main/crates/systemone-http/src/registry.rs) · [Remote transport](https://github.com/codesoda/systemone/blob/main/crates/systemone-remote/src/transport.rs) |
| Compatibility tests **versus** decision quality | Test model API conformance separately from operation/argument correctness and abstention | [SDK smoke](https://github.com/codesoda/systemone/blob/main/compat/sdk-js/smoke.mjs) · [Evaluation](https://github.com/codesoda/systemone/blob/main/docs/gliner2-evaluation.md) |

See [Reusable SystemOne patterns](SYSTEMONE_PATTERNS.md) for verified source links, an illustrative Choice request, implementation priorities, security boundaries, and acceptance tests. This is a **design reference**, not an assertion that UtterRoute or its Jev adapter currently supports SystemOne.

Design considerations if that adapter is implemented:

- Keep a single internal `Router` interface. Map allowed operation IDs and human-readable descriptions to a `choice` question, with an explicit no-match candidate. Parse the selected ID only; never accept generated target URLs or authorization data.
- The documented SystemOne Choice subset accepts **at most 16 options per question**. Preserve one for `none_of_the_above`, leaving at most 15 operation candidates. If a catalog is larger, add evaluated deterministic preselection or refuse routing; **do not silently drop eligible operations**. Candidate narrowing remains outside the v0.1 P0 scope.
- Send state as compact plain text where possible; its local OpenJev backend rejects floating-point JSON state, while other backends may accept it. Do not assume the wire subset matches every TypeSafe input.
- Run SystemOne as a separately configured process/service (for example port 8090 to avoid UtterRoute's proposed 8080); require an explicit trusted base URL and enforce network policy. Default loopback is suitable for development, not public unauthenticated deployment.
- Model/backends have different prediction quality, semantics, accuracy and calibration. A Jev-compatible response is **not** evidence of identical decisions. Configure the runtime/backend explicitly, pin tested versions, log sanitized backend identity, and **never** silently fall back from local inference to a hosted provider.
- Its local backends use separate model weights with independent license terms. SystemOne's MIT license covers its project code, not downloaded weights or hosted inference services.
- Add contract tests against [the published HTTP schema](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json), plus UtterRoute end-to-end tests for abstention and no-upstream-call behavior. Benchmark candidate counts, accuracy, refusal rate, latency and cost against direct hosted Jev before selecting a default.

**v0.1 decision:** Reference and evaluate SystemOne, but do not add it as a required dependency, claim support for it, or expand the current Jev-first single-operation GET release scope. Revisit it after the baseline direct Jev adapter and evaluation harness work.

## 7. Argument extraction and validation

The minimal supported resolution strategy is deliberately conservative:

- Unambiguous literal path IDs (e.g., “invoice `inv_123`”), without hallucinated resource IDs.
- Schema enum values via exact matching or bounded Jev selection.
- Deterministically parsed booleans, numeric values and explicit ISO dates with bounds checks.
- Clearly labeled scalar query strings; arbitrary free-text generation not guaranteed.
- Unknown optional parameters are omitted; missing required fields cause clarification.
- Multiple possible values cause `AMBIGUOUS_ARGUMENT`, not silent selection.

The compiler produces an internal plan:

```text
CompiledPlan:
  request_id
  connection_id
  operation_id
  method = GET
  trusted_origin_ref
  validated_path_parameters
  validated_query_parameters
  credential_ref
  route_metadata
```

The URL assembler safely escapes path segments and query strings using a robust HTTP client, rejecting path traversal and injection. It cannot accept absolute URLs, changes of scheme/host, or unregistered headers from model content. Revalidate all constraints **again** immediately before execution. The returned plan is not an executable approval token.

## 8. Security and network execution

**MVP security requirements:**

- Resolve destination from operator configuration, never model text or OpenAPI `servers`.
- Apply HTTP method/operation allowlist and prevent prompt-driven escalation.
- Protect against SSRF and DNS rebinding using explicit permitted targets, address checks and network policy. Private IP access must be deliberately enabled for local demo services.
- Disable redirects for upstream requests; bound connect/read timeouts, response size, overall concurrency and per-client rate.
- Configure credentials outside user prompts; never log tokens, Authorization headers or secret query values.
- Keep request text and upstream body logging off by default; treat OpenAPI docs and HTTP results as untrusted data.
- Do not automatically retry business API calls; GET endpoints are expected to be safe, but upstreams may violate HTTP semantics.
- Do not imply tenant-scoped identity/RBAC or OAuth account linking: not implemented in v0.1.

**Plan semantics:** `plan` may call the external Jev inference provider, which can receive the instruction and minimal operation descriptions. It must not make an upstream business API call. Document data-processing implications for deployers.

**Failure semantics:** If a GET was sent but transport failed before a response, report uncertainty; never claim the backend was not called.

## 9. Observability

Emit structured request ID, timestamp, connection ID, selected operation ID (when any), route engine/version, phase durations, outcome, and sanitized error code. Suppress raw prompts, request/response bodies and credentials by default. Correlation headers to upstream services may be forwarded only under explicit static policy.

For evaluation, create a labeled fixture set with expected operation, expected arguments, and expected abstentions. Publish route accuracy, end-to-end correctness, false-execution/abstention rates, p50/p95 latencies and cost. Compare against small-LLM function calling. Performance figures remain targets until benchmarked.

## 10. Test and release plan

**Unit tests:** OpenAPI import and rejection, operation ID stability, candidate validation, provider outages, enum/number/date parsing, conflicting values, URL encoding/traversal, status mapping and redaction.

**Integration tests:** A deterministic mock `billing-demo` exposing `GET /invoices` and `GET /invoices/{invoice_id}`:

- `plan` sends zero calls to the mock upstream.
- `execute` sends exactly one valid GET.
- Write intents, unknown operations, ambiguous IDs and unsupported schemas send zero calls.
- Provider timeout, upstream 4xx/5xx, oversized responses and redirects have explicit test coverage.
- Secrets do not appear in responses/logs.

**Security tests:** SSRF cases, hostile OpenAPI descriptions, prompt injection, malicious path values, credentials in logs, DNS/redirect manipulation, and method-bypass attempts.

**Release checklist:**

- [ ] Docker-based sample functions from a fresh clone with documented dependencies
- [ ] Local OpenAPI 3.x importer and GET operation allowlist
- [ ] Pinned Jev integration and explicit routing error behavior
- [ ] Working `plan` / `execute` and documented JSON/error contracts
- [ ] Tests prove no upstream call on plan/refused input
- [ ] Network/security hardening and diagnostic redaction tested
- [ ] Honest routing/latency benchmark versus baseline
- [ ] Security limits and third-party data egress documented

## 11. Suggested repository structure (future, not present)

```text
README.md
LICENSE
docs/{PRD.md,SPEC.md}
src/utterroute/
  api/
  config/
  openapi/
  registry/
  routing/{base.py,jev.py}
  arguments/
  policy/
  execution/
  telemetry/
examples/billing/
tests/{unit,integration,evaluation}/
Dockerfile
compose.yaml
pyproject.toml
```

Later releases may add approved writes, verification receipts, multiple dependent operations, entity lookup across APIs, MCP/GraphQL, delegated tenant access and a managed control plane. These require separate product decisions and threat modeling.
