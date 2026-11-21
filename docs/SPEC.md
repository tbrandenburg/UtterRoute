# UtterRoute v0.1 — Technical Specification

**Status:** Proposed, not implemented  
**Version:** 0.1-draft  
**Date:** 2026-11-21  
**Product scope:** [PRD.md](PRD.md) · [A2A-first ADR](ADR-001-A2A-FIRST.md) · [Capability resolver ADR-003](ADR-003-CAPABILITY-RESOLVER.md)

## 1. Architecture and invariants

UtterRoute is a **protocol-facing A2A capability resolver**, not a general HTTP API gateway. An inbound A2A request selects **one** registered, caller-authorized capability: either a permitted REST GET operation or an operator-approved existing A2A agent capability. The resolver dispatches through a thin target adapter, returns a task/artifact, or abstains/clarifies. A2A HTTP+JSON remains the first and only public business interface in v0.1; the proprietary `/v1/execute` design remains deferred.

**Hard invariants:**

1. **Capability eligibility first.** Operator configuration and authenticated caller policy decide which capabilities can be considered. Model decisions, Agent Cards, request hints and prompts never grant permission.
2. **One target per incoming task.** For a REST target: at most one allowlisted GET, validated origin and parameters. For an A2A target: at most one permitted outbound `SendMessage` initiating a child task, with bounded follow-up retrieval/clarification. Do not fan out or chain targets.
3. **No compulsory inference.** Explicit authorized IDs and unambiguous configured rules resolve deterministically with zero Jev/SystemOne calls. If the request remains ambiguous and no optional semantic selector is enabled, abstain/clarify.
4. **A2A agents are not inherently read-only.** Explicitly approve target agents and registered capabilities and their side-effect risk. Unknown-effect delegated skills are denied by default in the controlled v0.1 demo. The REST GET-only guarantee does not apply inside a remote agent.
5. **No untrusted destinations.** Only configured, trusted REST origins and registered A2A agent URLs. Model output, OpenAPI `servers`, Agent Card metadata and the user's instruction cannot introduce targets.
6. **Plan-only means no downstream target invocation:** `utterroute.mode=plan` may resolve a capability (including an optional inference call if configured) but must call **neither** the REST API **nor** the remote A2A agent.
7. **Task isolation and replay resistance:** Caller identity scopes Task retrieval/continuation and downstream task ID correlation. Duplicates do not trigger another dispatch; bounded storage, deadlines and result size are required.
8. **Do not replace existing gateways.** Ingress TLS/WAF/DDoS defenses and generic traffic management belong to existing infrastructure. UtterRoute retains minimal app-level authentication checks, capability authorization, fixed egress and workload limits.
9. Upstream REST status or a downstream agent's completed Task does not prove a business postcondition.

```text
A2A client -> existing edge / API gateway (optional)
       |
       v
A2A HTTP+JSON adapter, caller identity and owned tasks
       |
       v
Capability catalog: approved REST operations + A2A agents
       |
       v
Policy filter -> explicit/direct rule -> optional Jev choice -> abstain
       |
       v
Resolver + minimal dispatch (one target per task)
       |                                |
       +--> REST: validated GET         +--> A2A: SendMessage
            -> bounded result                -> correlated child Task
       |                                |
       +------------ A2A Task/artifact ------+
```

## 2. Component contracts

- **A2A ingress:** Uses the official A2A SDK for Agent Card, `SendMessage`, `GetTask`, Task and Artifact types. Never acts as a general reverse proxy.
- **Task store:** Authenticated caller identity, TTL/size bounds, input-required continuation, terminal-task replay protection and mappings to approved downstream A2A task/context IDs.
- **Capability catalog:** A normalized list of stable IDs, `rest_operation` or `a2a_agent` kind, descriptions, trusted target references, caller eligibility and side-effect risk. No public arbitrary destination registration.
- **REST importer:** Validates configured local OpenAPI and builds allowlisted GET operations with scalar parameter schemas.
- **A2A agent registration:** Loads an **operator-approved** agent URL and local capability ID; may read its Agent Card to validate declared protocol compatibility, but treats it as untrusted descriptive data. Do not recursively discover agents.
- **Resolver:** Filters capabilities by caller policy; chooses explicit/authoritatively configured deterministic match or abstains. A semantic `Router` adapter is an **optional extension** that may select only from that filtered set.
- **REST target:** Validates scalar arguments, origin/method policy, assembles a bounded GET with configured credentials and returns the actual HTTP status/body metadata.
- **A2A target:** Sends one A2A `SendMessage` to an approved agent, correlates child Task state/context, bounds retrieval/polling and maps result/clarification. It does not govern the remote agent's internal actions.
- **Safety:** Local caller/task authorization, fixed egress, timeouts, bounded pending work, redaction and no duplicate dispatch remain inside UtterRoute even behind existing ingress.
- **Observability:** Request and Task correlation IDs, direct/semantic route type, selected capability, timings, redacted outcomes; never model secrets or agent credentials.

Prefer a small Go service with plain interfaces and constructors, not a generic plugin or proxy architecture.

## 3. Configuration and runtime

**Proposed reference implementation:** **Go 1.26+** with official [`a2a-go/v2`](https://github.com/a2aproject/a2a-go) for A2A v1 HTTP+JSON (`a2asrv`/REST handler), standard `net/http` for serving and outbound clients, and [`kin-openapi`](https://github.com/getkin/kin-openapi) for a constrained OpenAPI 3.0/3.1 importer. See [ADR-002](ADR-002-GO-V0.1.md) for the language decision, Rust trade-offs, task-store caveats and phased PRs. Pin a tested SDK version and Go 1.26+ toolchain in the **first implementation PR**; these documents are not runnable code and no SDK is installed yet.

Configuration sketch (illustrative, not executable today):

```yaml
server:
  bind: 127.0.0.1
  port: 8080
  max_request_chars: 2000

routing:
  mode: deterministic   # no inference credentials needed
  semantic_provider: disabled  # optional future: jev

capabilities:
  rest:
    - id: billing-demo
      openapi_file: ./examples/billing/openapi.yaml
      origin: http://billing-demo:9000
      allowed_operation_ids: [listInvoices]
      auth:
        type: bearer_env
        env: BILLING_DEMO_TOKEN
  a2a:
    - id: research.competitor  # UtterRoute capability ID
      kind: a2a_agent
      target: research-agent  # operator-controlled downstream agent
      description: Research a competitor
      agent_url: http://127.0.0.1:9001  # trusted local demo only
      side_effect_policy: explicitly_approved
      auth:
        type: bearer_env
        env: RESEARCH_AGENT_TOKEN

limits:
  rest_allow_methods: [GET]
  max_target_calls_per_task: 1
  max_response_bytes: 1048576
  rest_timeout_ms: 5000
  a2a_timeout_ms: 5000
  max_child_task_polls: 4

logging:
  include_request_text: false
  include_response_body: false
```

Config is a **design sketch, not an executable schema**. Remote A2A agents need explicit trusted hosts and skill approvals; a discovered Agent Card alone does not grant authorization. The local example's HTTP loopback is not a production pattern. Optional model keys and upstream credentials come from the environment/secret management, never incoming messages. Default to loopback; reuse existing ingress/API gateways for public TLS, WAF/DDoS and general rate limits. Public deployment is **not supported by the v0.1 demo** without independent application policy review and appropriate external controls.

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

### Registered downstream A2A agents (v0.1 subset)

A downstream `a2a_agent` capability is **not** an OpenAPI operation or a remotely enforceable AgentSkill. It maps one local capability ID to an operator-approved agent origin, caller eligibility, credential reference and delegation risk policy. The A2A Agent Card's protocol-defined `skills` entries are **descriptive discovery metadata**; A2A `SendMessage` is addressed to an agent, not an individually enforceable AgentSkill. For v0.1 register one approved capability per remote agent target rather than promising fine-grained skill confinement. Validate the Agent Card and supported binding against **local configuration**; it must not override allowlists, credentials or permitted origins. Send at most one new A2A request per incoming task, and track any returned child Task/context IDs for bounded retrieval or authorized input-required continuation. Remote agents may have side effects; unknown-effect delegation is **not automatically safe**.

## 5. Public A2A API (v0.1 proposed)

The **only business-facing v0.1 protocol is A2A v1 HTTP+JSON**, consistent with the [A2A specification](https://a2a-protocol.org/v1.0.0/specification/) and [ADR-001](ADR-001-A2A-FIRST.md). This binding uses normal HTTP methods and A2A JSON bodies; it is not the JSON-RPC binding.

### `GET /.well-known/agent-card.json`

Return a valid A2A Agent Card declaring:
- `name`, `description`, `version`, supported input/output modes, and a truthful, bounded A2A Agent Card `skills` entry describing the resolver;
- `supportedInterfaces` with `protocolBinding: "HTTP+JSON"`, the declared A2A protocol version and an externally configured absolute HTTPS URL in deployed environments;
- an accurate authentication/security scheme; no secret connection names or upstream credentials;
- optional capabilities such as `streaming`, `pushNotifications`, and `extendedAgentCard` set false/omitted until actually supported.

Agent Card discovery does not grant access to tasks or upstream APIs. The protocol-defined Agent Card `skills` entries are descriptions, not authority to select arbitrary REST operations or constrain remote-agent behavior. For v0.1, one operator-configured connection per deployed agent is enough.

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

An ordinary text request may dispatch **one registered, authorized capability** when selection is deterministic; the REST path remains GET-only and an A2A agent capability can have separately approved side-effect risk. A documented optional `message.metadata["utterroute.capability_id"]` is an **untrusted selection hint**, not an A2A standard field and never an authorization grant. The optional UtterRoute-specific `message.metadata["utterroute.mode"] = "plan"` returns a preview with **zero downstream REST or A2A calls**. Model inference is not required unless separately configured for ambiguous requests.

### Task lifecycle and continuation

| Internal result | A2A state | Required behavior |
| --- | --- | --- |
| Successful preview | `TASK_STATE_COMPLETED` | Plan artifact; no upstream call |
| Valid REST GET with upstream response | `TASK_STATE_COMPLETED` | Artifact includes selected REST capability, sanitized typed args, actual upstream status and bounded body |
| Approved A2A delegation | Mirrors mapped child state (`WORKING`, `INPUT_REQUIRED`, `COMPLETED` or appropriate failure) | Correlate caller-owned parent Task to approved child task/context; bound polling and metadata disclosure |
| Missing/ambiguous required argument | `TASK_STATE_INPUT_REQUIRED` | Clarifying status message; no upstream business call |
| Denied/non-GET/no matching operation | `TASK_STATE_REJECTED` | No business call; sanitized explanation |
| Routing/transport failure | `TASK_STATE_FAILED` | Sanitized error; do not invent a fallback call |

A task may reach `TASK_STATE_WORKING` during execution, but terminal tasks cannot be resumed. For input-required tasks, a follow-up with matching `taskId` and `contextId` resumes the **same** caller-owned task after authorization. The runtime rejects context/task mismatches, unauthorized reads, duplicate execution and expired tasks. **At most one REST GET, or one remote A2A SendMessage initiating a delegation, per task** even across retries/continuation; permitted follow-up to an existing child task is not a second delegation. Poll only within configured bounded capacity/deadlines.

A `TASK_STATE_COMPLETED` status means the resolver has finished its task. A REST API can still return a non-2xx response; a remote A2A Task may report failure/input-required or contain application-level failure. Keep downstream status and risk explicit in the artifact.

### `GET /tasks/{id}` — A2A `GetTask`

Retrieve a scoped task using the v1 A2A Task schema. Tasks are stored in a **bounded, temporary in-memory single-instance store** with documented TTL/size limits. Unknown, expired or unauthorized tasks must not disclose details; production-grade persistence, resumable streaming and multiworker synchronization are beyond v0.1.

### Protocol errors and operational failures

Malformed A2A messages, unsupported protocol versions, invalid task IDs, unsupported capabilities, and authentication failures use the protocol's error shape and HTTP status defined by the pinned A2A binding. **Do not reuse** the deferred proprietary HTTP error envelope for A2A.

For an accepted task, represent expected application outcomes by Task state and/or artifact; in particular, distinguish refused operations, requests for clarification, inference outages and upstream failures. A2A authentication must be separate from credentials for either target type (REST or remote A2A agent).

### Deferred custom native HTTP adapter

`POST /v1/execute` is **not part of v0.1**. The old request/response draft is retained for potential **v0.2+** work at [HTTP_API_FUTURE.md](HTTP_API_FUTURE.md); it must call the same core service and must not bypass A2A-era policy/validation. No extra proprietary HTTP endpoint is needed to pass v0.1 acceptance.


## 6. Capability selection logic (deterministic first)

1. Normalize the configured catalog into stable REST operation and registered A2A agent capability IDs.
2. **Filter by caller eligibility and operator policy** before considering selection. Remote Agent Cards and incoming hints are untrusted.
3. If the caller provides an explicit capability ID hint, validate it against the filtered catalog; if valid and arguments/side-effect policy pass, directly dispatch with **zero inference calls**.
4. Otherwise apply only unambiguous, operator-configured deterministic rules. If no safe unique match exists, request clarification/refuse by default.
5. **Optional later:** An explicitly configured Jev (or alternative) router receives only the eligible descriptions and an abstain candidate; it may choose **only** one supplied ID. Reject any hallucinated destination/unknown operation or low-confidence decision. Never silently switch local to hosted inference.
6. Validate REST arguments and allowed GET method or registered A2A capability/delegation-risk approval again at dispatch time. Record a sanitized routing-path label: `explicit`, `deterministic`, `semantic` or `abstain`.

The fast path is the product baseline. Jev is the **first planned optional semantic adapter**, not a required `SendMessage` dependency. If no model is configured, direct and deterministic resolution still operate. Semantic thresholds require labeled evaluations, not universal confidence assumptions.

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

**Current decision:** SystemOne remains a reference or future optional router. Neither SystemOne nor hosted Jev is required for the deterministic v0.1 baseline. If semantic selection is added, evaluate direct Jev before treating SystemOne as an optional runtime. Never embed a required inference server in the fast path.

## 7. Argument extraction and validation

The minimal REST argument-resolution strategy is deliberately conservative; it applies only when a `rest_operation` capability has been selected, regardless of ingress protocol:

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
- Disable redirects for REST and remote A2A requests; bound connect/read timeouts, response size, overall concurrency and per-client rate. Implement bounded child-task polling and prevent recursive agent calls.
- Configure credentials outside user prompts; never log tokens, Authorization headers or secret query values.
- Keep request text and upstream body logging off by default; treat OpenAPI docs and HTTP results as untrusted data.
- Do not automatically retry business API calls; GET endpoints are expected to be safe, but upstreams may violate HTTP semantics.
- v0.1 requires explicit local caller-to-task and caller-to-capability authorization; do not imply production-ready tenant-level RBAC, OAuth account linking, or delegated identity.

**Plan semantics:** A2A requests marked `utterroute.mode=plan` may run deterministic selection or, if explicitly configured, optional semantic inference. They make **zero** outbound REST business calls and **zero** outbound A2A target calls. Document data-processing implications for deployers.

**Failure semantics:** If a GET was sent but transport failed before a response, report uncertainty; never claim the backend was not called.

## 9. Observability

Emit structured request ID, timestamp, connection ID, selected operation ID (when any), route engine/version, phase durations, outcome, and sanitized error code. Suppress raw prompts, request/response bodies and credentials by default. Correlation headers to upstream services may be forwarded only under explicit static policy.

For evaluation, create a labeled fixture set with expected operation, expected arguments, and expected abstentions. Publish route accuracy, end-to-end correctness, false-execution/abstention rates, p50/p95 latencies and cost. Compare against small-LLM function calling. Performance figures remain targets until benchmarked.

## 10. Test and release plan

**Unit tests (Go `testing` and `httptest`, with `-race` in CI):** A2A request/Task/Agent Card schemas, task-state mapping and ownership/expiry; OpenAPI import and rejection, operation ID stability, candidate validation, provider outages, enum/number/date parsing, conflicting values, URL encoding/traversal, status mapping and redaction.

**Integration tests:** A pinned, independent A2A client interacting with a deterministic mock billing REST API **and** an independent mock A2A research agent:

- A2A Agent Card can be discovered and parsed; `SendMessage` returns a Task; `GetTask` retrieves only authorized tasks.
- A2A plan-only metadata produces a completed plan artifact and zero business calls.
- An explicit, authorized REST capability request executes at most one valid GET per Task and returns an artifact, with **zero** inference calls.
- An explicit, approved A2A research capability causes one outbound `SendMessage`, correlates child task/context and returns a mapped artifact or bounded working/input-required task.
- Unauthorized/unapproved remote A2A capabilities and unknown-effect risks cause **zero** outbound messages; duplicate/replayed tasks cause no additional delegation.
- Ambiguous requests without a configured semantic router abstain rather than forcing Jev.
- Input-required tasks can be continued; terminal tasks cannot be resumed or re-executed.
- Unauthorized task retrieval and missing/expired task references reveal no confidential data.
- Write intents, unknown operations, ambiguous IDs and unsupported schemas send zero calls.
- Provider timeout, upstream 4xx/5xx, oversized responses and redirects have explicit test coverage.
- Secrets do not appear in responses/logs.

**Security tests:** SSRF cases, hostile OpenAPI descriptions, prompt injection, malicious path values, credentials in logs, DNS/redirect manipulation, and method-bypass attempts.

**Release checklist:**

- [ ] Docker-based sample functions from a fresh clone with documented dependencies
- [ ] Local OpenAPI 3.x importer and GET allowlist **plus** one trusted remote A2A agent capability registration
- [ ] Explicit/deterministic resolution works with no Jev/SystemOne key; optional inference remains a separately evaluated extension
- [ ] A2A HTTP+JSON Agent Card, `SendMessage`, `GetTask`, task state and artifact contracts pass independent client tests
- [ ] Plan-only task and read-only execution through the same internal service; custom HTTP adapter deferred
- [ ] Tests prove no target invocation on plan/refused/ambiguous requests; per task at most one REST GET **or** one initial outbound A2A delegation
- [ ] Network/security hardening and diagnostic redaction tested
- [ ] Compare direct target invocation vs deterministic UtterRoute path p50/p95, including zero-inference verification; optional semantic comparison when configured
- [ ] Security limits and third-party data egress documented

## 11. Suggested repository structure (future, not present)

```text
README.md
LICENSE
docs/{PRD.md,SPEC.md,ADR-001-A2A-FIRST.md,ADR-002-GO-V0.1.md,ADR-003-CAPABILITY-RESOLVER.md}
go.mod
cmd/utterroute/main.go
internal/
  a2a/           # inbound protocol and owned Task correlation
  resolver/      # caller-eligible, deterministic-first selection
  catalog/       # REST operations + approved remote A2A capabilities
  targets/       # rest.go and a2a.go outbound adapters
  config/        # trusted target/credential/policy references
examples/{billing-api,research-agent}/
tests/{integration,evaluation}/
Dockerfile
compose.yaml
```

Keep the first implementation small. REST argument resolution and minimal dispatch policy can live in `resolver/` and `targets/` until package size merits extraction. Avoid a generic proxy/plugin framework.



Later releases may add the secondary custom `/v1/execute` interface, streaming/cancel/push, durable tasks, approved writes, verification receipts, multiple dependent operations, entity lookup across APIs, MCP/GraphQL, delegated tenant access and a managed control plane. These require separate product decisions and threat modeling.
