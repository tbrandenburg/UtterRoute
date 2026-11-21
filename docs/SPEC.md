# UtterRoute v0.1 — Technical Specification

**Status:** Proposed, not implemented  
**Version:** 0.1-draft  
**Date:** 2026-11-21  
**Product scope:** [PRD.md](PRD.md) · [A2A-first ADR](ADR-001-A2A-FIRST.md)

## 1. Architecture and invariants

UtterRoute presents a **primary A2A HTTP+JSON interface** for receiving natural-language messages from other agents, then converts each accepted task into **at most one allowlisted GET operation** described by an OpenAPI 3.x document. It returns an A2A task with a plan/result artifact or refuses/requests clarification. The custom `POST /v1/execute` HTTP interface is deferred until v0.2 or later. v0.1 is not a general agent or workflow engine.

**Hard invariants:**

1. Outbound targets come only from operator-controlled connection configuration; never from the prompt, a model, or an OpenAPI server URL.
2. Only explicitly allowlisted GET operations may execute. No writes or implicit multistep calls.
3. Required arguments must be validated; missing/ambiguous values cause clarification, never guesses.
4. An A2A message with `utterroute.mode=plan` must make **zero upstream business API calls**. An accepted read-only execute task may make at most one allowlisted GET, including after clarification continuation.
5. The model is a selector, not an authority. Policy checks run regardless of model confidence.
6. Routing failures fail closed, with no automatic unsafe fallback.
7. An upstream HTTP response is not a guarantee of business correctness.

```text
A2A client
  |
  +--> GET /.well-known/agent-card.json
  +--> POST /message:send  (A2A HTTP+JSON)
  +--> GET /tasks/{id}
               |
               v
       A2A task manager + auth
               |
               v
OpenAPI operation registry + explicit GET allowlist
               |
               v
Candidate selector -> Jev router (choice + abstain)
               |
               v
Argument resolver -> Schema and policy validation
               |
               +--> plan: Task + plan artifact, zero business calls
               +--> execute: one fixed-origin GET -> Task + result artifact
```

## 2. Component contracts

- **A2A transport:** Serves the Agent Card, handles protocol-versioned `SendMessage` and `GetTask`, validates A2A messages and maps results to conformant Task/Artifact types. Does not authorize API operations by itself.
- **Task store:** Bounded local task retention (TTL), caller ownership, controlled input-required continuation and terminal-task handling.
- **OpenAPI importer:** Parses a local OpenAPI file at startup, resolves supported local references, builds operation and scalar-parameter schemas, rejects unsupported/invalid inputs.
- **Registry:** Immutable initialized collection of named connections, operation IDs, trusted origins, descriptions, method policy and authentication references.
- **Candidate selector:** Starts with eligible operations; can optionally shorten candidates deterministically for larger catalogs. It must preserve a no-match route.
- **Router adapter:** `route(instruction, candidates, context) -> RoutingDecision` where `RoutingDecision` contains only an allowed operation ID or abstain, a score (if available), engine/version and reason. Jev is the first adapter; don't assume generic JSON generation.
- **Argument resolver:** Parses explicit scalar path/query values and can use Jev for bounded enum selection. Tracks missing, conflicting or unsupported arguments.
- **Compiler:** Converts operation ID and parameters into a validated internal `CompiledPlan`; never takes method, origin, secrets or headers from router output.
- **Policy guard:** Enforces allowlist, configured origin, method, argument schema and execution mode.
- **Executor:** Issues at most one bounded GET per accepted A2A task using configured upstream credentials; no redirects or automatic retries in v0.1.
- **Telemetry:** Reports request IDs, stage durations, sanitized selected operations and outcomes.

## 3. Configuration and runtime

**Proposed implementation:** Python 3.11+, FastAPI, an HTTP client with strict timeouts, and Docker Compose. This is a recommendation, not existing code or a binding maintainer decision. Use a version-pinned A2A SDK/contract for HTTP+JSON message and task types where practical; the current codebase does not implement any A2A server yet.

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

## 5. Public A2A API (v0.1 proposed)

The **only business-facing v0.1 protocol is A2A v1 HTTP+JSON**, consistent with the [A2A specification](https://a2a-protocol.org/v1.0.0/specification/) and [ADR-001](ADR-001-A2A-FIRST.md). This binding uses normal HTTP methods and A2A JSON bodies; it is not the JSON-RPC binding.

### `GET /.well-known/agent-card.json`

Return a valid A2A Agent Card declaring:
- `name`, `description`, `version`, supported input/output modes, and a constrained read-API skill;
- `supportedInterfaces` with `protocolBinding: "HTTP+JSON"`, the declared A2A protocol version and an externally configured absolute HTTPS URL in deployed environments;
- an accurate authentication/security scheme; no secret connection names or upstream credentials;
- optional capabilities such as `streaming`, `pushNotifications`, and `extendedAgentCard` set false/omitted until actually supported.

Agent Card discovery does not grant access to tasks or upstream APIs. Skill names are stable business-level descriptions, not unlimited permission to choose arbitrary OpenAPI operations. For v0.1, one operator-configured connection per deployed agent is enough.

### `POST /message:send` — A2A `SendMessage`

Receive an A2A v1 `SendMessageRequest` with a `ROLE_USER` message and at least one text part. The authenticated caller may submit a new request or continue an existing `TASK_STATE_INPUT_REQUIRED` task with matching `taskId` and `contextId`. Valid requests always produce a server-assigned, retained A2A Task; do not return a custom UtterRoute JSON envelope.

Illustrative HTTP+JSON request (transport headers, including content type and A2A version negotiation, must follow the pinned A2A spec):

```http
POST /message:send
Content-Type: application/a2a+json
A2A-Version: 1.0
Authorization: Bearer <local-development-token>
```

```json
{
  "message": {
    "messageId": "msg-001",
    "role": "ROLE_USER",
    "parts": [
      {"text": "List unpaid invoices"}
    ]
  }
}
```

An ordinary text request defaults to **read-only execution**. The optional UtterRoute-specific `message.metadata["utterroute.mode"] = "plan"` requests a preview and **zero upstream business API calls**. This metadata field is a documented application convention, not a core A2A field. No write operation is enabled by omitting it.

### Task lifecycle and continuation

| Internal result | A2A state | Required behavior |
| --- | --- | --- |
| Successful preview | `TASK_STATE_COMPLETED` | Plan artifact; no upstream call |
| Valid GET with upstream response | `TASK_STATE_COMPLETED` | Result artifact includes selected operation, sanitized typed args, actual upstream status and bounded body |
| Missing/ambiguous required argument | `TASK_STATE_INPUT_REQUIRED` | Clarifying status message; no upstream business call |
| Denied/non-GET/no matching operation | `TASK_STATE_REJECTED` | No business call; sanitized explanation |
| Routing/transport failure | `TASK_STATE_FAILED` | Sanitized error; do not invent a fallback call |

A task may reach `TASK_STATE_WORKING` during execution, but terminal tasks cannot be resumed. For input-required tasks, a follow-up with matching `taskId` and `contextId` resumes the **same** task after caller authorization. The runtime rejects context/task mismatches, unauthorized reads, duplicate execution and expired tasks. **At most one upstream business GET per task** even across retries/continuation.

A `TASK_STATE_COMPLETED` status means the gateway finished its task. A business API can still return a non-2xx status, which must be included in the result artifact rather than disguised as a successful business outcome.

### `GET /tasks/{id}` — A2A `GetTask`

Retrieve a scoped task using the v1 A2A Task schema. Tasks are stored in a **bounded, temporary in-memory single-instance store** with documented TTL/size limits. Unknown, expired or unauthorized tasks must not disclose details; production-grade persistence, resumable streaming and multiworker synchronization are beyond v0.1.

### Protocol errors and operational failures

Malformed A2A messages, unsupported protocol versions, invalid task IDs, unsupported capabilities, and authentication failures use the protocol's error shape and HTTP status defined by the pinned A2A binding. **Do not reuse** the deferred proprietary HTTP error envelope for A2A.

For an accepted task, represent expected application outcomes by Task state and/or artifact; in particular, distinguish refused operations, requests for clarification, inference outages and upstream failures. A2A authentication must be separate from service credentials for the target REST API.

### Deferred custom native HTTP adapter

`POST /v1/execute` is **not part of v0.1**. The old request/response draft is retained for potential **v0.2+** work at [HTTP_API_FUTURE.md](HTTP_API_FUTURE.md); it must call the same core service and must not bypass A2A-era policy/validation. No extra proprietary HTTP endpoint is needed to pass v0.1 acceptance.


## 6. Routing logic

1. Find the configured connection and filter operations by explicit allowlist and GET method.
2. Prepare a bounded set of operation labels/descriptions; include explicit `NONE_OF_THE_ABOVE` or equivalent abstention candidate.
3. Ask Jev to select an eligible operation using its bounded-choice mechanism. Send no upstream API secrets, headers or business response bodies.
4. Reject any operation ID not in the supplied set, even if the model produces a plausible endpoint.
5. Abstain when no match, contradictory intent, provider error, or low confidence under a **calibrated** policy.
6. Record model/adapter version, candidate-set identifier and sanitized timing metadata.

Thresholds should be selected from evaluation data, not hardcoded as a universal guarantee. Evaluate sensitivity to candidate ordering, phrasing, and schema-description changes. Model scoring does not replace checking the final operation and arguments.

### Optional SystemOne interoperability (reference; transport independent of A2A)

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

The minimal supported resolution strategy is deliberately conservative; it applies to the core service regardless of outer A2A or future native HTTP transport:

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

The URL assembler safely escapes path segments and query strings using a robust HTTP client, rejecting path traversal and injection. It cannot accept absolute URLs, changes of scheme/host, or unregistered headers from model content. Revalidate all constraints **again** immediately before execution. An A2A plan artifact is informational, not an executable approval token.

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

**Plan semantics:** A2A requests explicitly marked `utterroute.mode=plan` may call the external Jev inference provider, which can receive the instruction and minimal operation descriptions. They must not make an upstream business API call. Document data-processing implications for deployers.

**Failure semantics:** If a GET was sent but transport failed before a response, report uncertainty; never claim the backend was not called.

## 9. Observability

Emit structured request ID, timestamp, connection ID, selected operation ID (when any), route engine/version, phase durations, outcome, and sanitized error code. Suppress raw prompts, request/response bodies and credentials by default. Correlation headers to upstream services may be forwarded only under explicit static policy.

For evaluation, create a labeled fixture set with expected operation, expected arguments, and expected abstentions. Publish route accuracy, end-to-end correctness, false-execution/abstention rates, p50/p95 latencies and cost. Compare against small-LLM function calling. Performance figures remain targets until benchmarked.

## 10. Test and release plan

**Unit tests:** A2A request/Task/Agent Card schemas, task-state mapping and ownership/expiry; OpenAPI import and rejection, operation ID stability, candidate validation, provider outages, enum/number/date parsing, conflicting values, URL encoding/traversal, status mapping and redaction.

**Integration tests:** A pinned, independent A2A client interacting with a deterministic mock `billing-demo` exposing `GET /invoices` and `GET /invoices/{invoice_id}`:

- A2A Agent Card can be discovered and parsed; `SendMessage` returns a Task; `GetTask` retrieves only authorized tasks.
- A2A plan-only metadata produces a completed plan artifact and zero business calls.
- An ordinary read-only A2A request executes at most one valid GET per Task, returning a structured result artifact.
- Input-required tasks can be continued; terminal tasks cannot be resumed or re-executed.
- Unauthorized task retrieval and missing/expired task references reveal no confidential data.
- Write intents, unknown operations, ambiguous IDs and unsupported schemas send zero calls.
- Provider timeout, upstream 4xx/5xx, oversized responses and redirects have explicit test coverage.
- Secrets do not appear in responses/logs.

**Security tests:** SSRF cases, hostile OpenAPI descriptions, prompt injection, malicious path values, credentials in logs, DNS/redirect manipulation, and method-bypass attempts.

**Release checklist:**

- [ ] Docker-based sample functions from a fresh clone with documented dependencies
- [ ] Local OpenAPI 3.x importer and GET operation allowlist
- [ ] Pinned Jev integration and explicit routing error behavior
- [ ] A2A HTTP+JSON Agent Card, `SendMessage`, `GetTask`, task state and artifact contracts pass independent client tests
- [ ] Plan-only task and read-only execution through the same internal service; custom HTTP adapter deferred
- [ ] Tests prove no upstream call on plan/refused/input-required requests, and at most one GET per task
- [ ] Network/security hardening and diagnostic redaction tested
- [ ] Honest routing/latency benchmark versus baseline
- [ ] Security limits and third-party data egress documented

## 11. Suggested repository structure (future, not present)

```text
README.md
LICENSE
docs/{PRD.md,SPEC.md,ADR-001-A2A-FIRST.md,HTTP_API_FUTURE.md,SYSTEMONE_PATTERNS.md}
src/utterroute/
  a2a/
  tasks/
  api/  # internal service; public native HTTP adapter deferred
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

Later releases may add the secondary custom `/v1/execute` interface, streaming/cancel/push, durable tasks, approved writes, verification receipts, multiple dependent operations, entity lookup across APIs, MCP/GraphQL, delegated tenant access and a managed control plane. These require separate product decisions and threat modeling.
