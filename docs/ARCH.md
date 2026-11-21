# UtterRoute — Architecture and Technical Contract

**Status:** Proposed architecture; the repository remains documentation-only until a code PR lands.  
**Authority:** Technical contract and invariants live here. Product scope and user journeys live in [PRD](PRD.md); active implementation slices and exit gates live in [PLAN](PLAN.md). Decision rationale is retained in [ADRs](adr/ADR-003-CAPABILITY-RESOLVER.md); reference designs are in [archive](archive/SYSTEMONE_PATTERNS.md).

**Working vocabulary:** *Capability* = registered authorized unit of work; *Target* = approved REST API or remote A2A agent; *Resolver* = eligibility-filtered selector. The A2A wire protocol's `AgentSkill`/`skills` fields are descriptive metadata, not selectable, enforceable UtterRoute permissions.



**Status:** Proposed, not implemented  
**Version:** 0.1-draft  
**Date:** 2026-11-21  
**Product scope:** [PRD.md](PRD.md) · [A2A-first ADR](adr/ADR-001-A2A-FIRST.md) · [Capability resolver ADR-003](adr/ADR-003-CAPABILITY-RESOLVER.md)

## 1. Architecture and invariants

UtterRoute is a **protocol-facing A2A capability resolver**, not a general HTTP API gateway. An inbound A2A request selects **one** registered, caller-authorized capability: a permitted REST GET or an approved A2A agent in the baseline. A later **embedded Go SDK** may register a third `local_handler` target for a Go function running **in the same process**. The resolver dispatches through a thin target adapter (or a later in-process handler adapter), returns a task/artifact, or abstains/clarifies. A2A HTTP+JSON remains the first and only public business interface in v0.1; the proprietary `/v1/execute` design remains deferred.

**Hard invariants:**

1. **Capability eligibility first.** Operator configuration and authenticated caller policy decide which capabilities can be considered. Model decisions, Agent Cards, request hints and prompts never grant permission.
2. **One target per incoming task.** For a REST target: at most one allowlisted GET, validated origin and parameters. For an A2A target: at most one permitted outbound `SendMessage` initiating a child task, with bounded follow-up retrieval/clarification. Do not fan out or chain targets.
3. **No compulsory inference.** Explicit authorized IDs and unambiguous configured rules resolve deterministically with zero Jev/SystemOne calls. If the request remains ambiguous and no optional semantic selector is enabled, abstain/clarify.
4. **A2A agents are not inherently read-only.** Explicitly approve target agents and registered capabilities and their side-effect risk. Unknown-effect remote-agent delegations are denied by default in the controlled v0.1 demo. The REST GET-only guarantee does not apply inside a remote agent.
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
- **Capability catalog:** A normalized list of stable IDs, target kinds (`rest_operation`, `a2a_agent`, or proposed embedded-only `local_handler`), descriptions, trusted target references/registrations, caller eligibility and risk classification. No public arbitrary destination registration. S0–S4 supports only REST and remote A2A.
- **REST importer:** Validates configured local OpenAPI and builds allowlisted GET operations with scalar parameter schemas.
- **A2A agent registration:** Loads an **operator-approved** agent URL and local capability ID; may read its Agent Card to validate declared protocol compatibility, but treats it as untrusted descriptive data. Do not recursively discover agents.
- **Resolver:** Filters by caller policy and selects an explicit/authoritatively configured deterministic match or abstains. Optional semantic `Router` may select only from the filtered set. One selection means one local-handler invocation **or** one remote dispatch, never a sequence.
- **REST target:** Validates scalar arguments, origin/method policy, assembles a bounded GET with configured credentials and returns the actual HTTP status/body metadata.
- **A2A target:** Sends one A2A `SendMessage` to an approved agent, correlates child Task state/context, bounds retrieval/polling and maps result/clarification. It does not govern the remote agent's internal actions.
- **Safety:** Local caller/task authorization, fixed egress, timeouts, bounded pending work, redaction and no duplicate dispatch remain inside UtterRoute even behind existing ingress.
- **Observability:** Request and Task correlation IDs, direct/semantic route type, selected capability, timings, redacted outcomes; never model secrets or agent credentials.

Prefer a small Go service with plain interfaces and constructors, not a generic plugin or proxy architecture.

## 3. Configuration and runtime

**Proposed reference implementation:** **Go 1.26+** with official [`a2a-go/v2`](https://github.com/a2aproject/a2a-go) for A2A v1 HTTP+JSON (`a2asrv`/REST handler), standard `net/http` for serving and outbound clients, and [`kin-openapi`](https://github.com/getkin/kin-openapi) for a constrained OpenAPI 3.0/3.1 importer. See [ADR-002](adr/ADR-002-GO-V0.1.md) for the language decision, Rust trade-offs, task-store caveats and phased PRs. Pin a tested SDK version and Go 1.26+ toolchain in the **first implementation PR**; these documents are not runnable code and no SDK is installed yet.

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

Config is a **design sketch, not an executable schema**. Remote A2A agents need explicit trusted hosts and capability approvals; a discovered Agent Card alone does not grant authorization. The local example's HTTP loopback is not a production pattern. Optional model keys and upstream credentials come from the environment/secret management, never incoming messages. Default to loopback; reuse existing ingress/API gateways for public TLS, WAF/DDoS and general rate limits. Public deployment is **not supported by the v0.1 demo** without independent application policy review and appropriate external controls.

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

A downstream `a2a_agent` capability is **not** an OpenAPI operation or a remotely enforceable AgentSkill. It maps one local capability ID to an operator-approved agent origin, caller eligibility, credential reference and delegation risk policy. The A2A Agent Card's protocol-defined `skills` entries are **descriptive discovery metadata**; A2A `SendMessage` is addressed to an agent, not an individually enforceable AgentSkill. For v0.1 register one approved capability per remote agent target rather than promising per-`AgentSkill` confinement. Validate the Agent Card and supported binding against **local configuration**; it must not override allowlists, credentials or permitted origins. Send at most one new A2A request per incoming task, and track any returned child Task/context IDs for bounded retrieval or authorized input-required continuation. Remote agents may have side effects; unknown-effect delegation is **not automatically safe**.

## 4a. Proposed embedded Go SDK (`local_handler`; post-S4 experiment)

**Purpose:** Let an existing Go application expose reviewed typed functions as caller-authorized A2A capabilities **without writing its own A2A server**. The SDK is a third **target kind/deployment mode**, **not** a third network protocol or a baseline S0–S4 requirement.

### Deployment boundary

```text
Standalone: A2A ingress -> Catalog -> Resolver -> REST GET | remote A2A
Embedded:   A2A ingress -> Catalog -> Resolver -> local_handler -> Go function
                                                          (same process)
Standalone UtterRoute can call an embedded Go application's A2A endpoint
as a normal approved remote a2a_agent target; it cannot invoke arbitrary
functions in a different process.
```

Reuse the catalog, policy, resolver, task ownership/retention, A2A status/artifact mapping and observability. Add only a typed local-handler adapter plus mounting of the existing A2A interface. Avoid a second resolver or private RPC system.

### Illustrative Go registration API (not yet implemented)

```go
type LookupInput struct { OrderID string `json:"order_id"` }
type LookupOutput struct { Status string `json:"status"` }

func lookup(ctx context.Context, in LookupInput) (LookupOutput, error) {
    return orders.Lookup(ctx, in.OrderID)
}

router := utterroute.New(/* auth, policies, limits */)
utterroute.Register(router, utterroute.Capability{
    ID: "orders.lookup",
    Description: "Look up an order's status",
    Effect: utterroute.ReadOnly,
}, lookup)
http.Handle("/a2a/", router.Handler())
```

The exact generic registration signature, caller-policy API, A2A handler mounting and result mapping must be verified during the SDK experiment against the pinned Go A2A SDK. Do not present this sketch as compiling code.

### Local function invocation contract

1. **Trusted registration:** Only application startup code registers stable capability IDs, reviewed descriptions, typed inputs/outputs, risk classification, authorization policy and handlers. Client/model/Agent Card text cannot register functions.
2. **Selection vs argument binding:** The resolver chooses one eligible capability ID (explicit, deterministic or **optional** semantic selection) or abstains. Jev/SystemOne does **not** synthesize Go struct arguments or execution plans. Initial SDK bindings use **caller-provided structured arguments**; developer-provided deterministic parsing may follow. Missing/ambiguous values produce clarification/refusal.
3. **Pre-dispatch checks:** Recheck caller permissions; validate required fields, types, sizes and inputs against a bounded schema. Do not reflect over arbitrary exported Go methods, generate code or trust caller-supplied function names.
4. **Read-only synchronous first:** A registered handler has the conceptual shape `func(context.Context, Input) (Output, error)`. The initial experiment accepts only explicitly reviewed read-only functions. An effect annotation is not a sandbox: a handler could still perform writes. `context.Context` cancellation is **cooperative**; a timeout cannot forcibly stop arbitrary Go code.
5. **One local invocation per admitted Task:** Protect against concurrent replay and duplicate execution. `plan`, denied or ambiguous requests must execute **zero** handlers. Enforce deadlines, admission/concurrency caps, bounded input/output/artifacts and caller-scoped Task retrieval. No model or hosted-provider key needed for explicit resolution.
6. **Embedding trust:** Handlers run with the host process's credentials, memory and privileges. The embedding application must provide suitable identity mapping, caller/tenant isolation and domain authorization; UtterRoute does not manufacture a security sandbox.
7. **No remote Go RPC:** A Go app embedding UtterRoute publishes an A2A endpoint. Standalone UtterRoute may invoke it through existing `a2a_agent` dispatch. No dynamic plugin loading, cross-process Go callbacks or generic method reflection.

### SDK acceptance and value test

A third-party A2A v1 client discovers and invokes a registered read-only Go handler; malformed/denied/plan/ambiguous inputs invoke **zero** handlers. Test schema validation, Task/artifact mapping, duplicate requests, timeouts, oversized results and caller identity isolation. Compare **direct Go invocation** against the embedded A2A path for p50/p95, allocations and integration effort. Compare adoption friction with a small hand-written A2A wrapper or MCP exposure, not just a REST proxy. Run **after S0–S4**; see [PLAN](PLAN.md).

## 5. Public A2A API (v0.1 proposed)

The **only business-facing v0.1 protocol is A2A v1 HTTP+JSON**, consistent with the [A2A specification](https://a2a-protocol.org/v1.0.0/specification/) and [ADR-001](adr/ADR-001-A2A-FIRST.md). This binding uses normal HTTP methods and A2A JSON bodies; it is not the JSON-RPC binding.

### `GET /.well-known/agent-card.json`

Return a valid A2A Agent Card declaring:
- `name`, `description`, `version`, supported input/output modes, and truthful, bounded protocol-defined `skills` entries describing the resolver;
- `supportedInterfaces` with `protocolBinding: "HTTP+JSON"`, the declared A2A protocol version and an externally configured absolute HTTPS URL in deployed environments;
- an accurate authentication/security scheme; no secret connection names or upstream credentials;
- optional capabilities such as `streaming`, `pushNotifications`, and `extendedAgentCard` set false/omitted until actually supported.

Agent Card discovery does not grant access to tasks or upstream APIs. The protocol-defined Agent Card `skills` entries are descriptions, not authority to select arbitrary REST operations or constrain remote-agent behavior. For the v0.1 demo, one configured REST operation and one configured remote A2A agent target are enough.

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

`POST /v1/execute` is **not part of v0.1**. The old request/response draft is retained for potential **v0.2+** work at [HTTP_API_FUTURE.md](archive/HTTP_API_FUTURE.md); it must call the same core service and must not bypass A2A-era policy/validation. No extra proprietary HTTP endpoint is needed to pass v0.1 acceptance.


## 6. Capability selection logic (deterministic first)

1. Normalize the configured catalog into stable REST operation and registered A2A agent capability IDs.
2. **Filter by caller eligibility and operator policy** before considering selection. Remote Agent Cards and incoming hints are untrusted.
3. If the caller provides an explicit capability ID hint, validate it against the filtered catalog; if valid and arguments/side-effect policy pass, directly dispatch with **zero inference calls**.
4. Otherwise apply only unambiguous, operator-configured deterministic rules. If no safe unique match exists, request clarification/refuse by default.
5. **Optional later:** An explicitly configured Jev (or alternative) router receives only the eligible descriptions and an abstain candidate; it may choose **only** one supplied ID. Reject any hallucinated destination/unknown operation or low-confidence decision. Never silently switch local to hosted inference.
6. Validate REST arguments and allowed GET method or registered A2A capability/delegation-risk approval again at dispatch time. Record a sanitized routing-path label: `explicit`, `deterministic`, `semantic` or `abstain`.

The fast path is the product baseline. Jev is the **first planned optional semantic adapter**, not a required `SendMessage` dependency. If no model is configured, direct and deterministic resolution still operate. Semantic thresholds require labeled evaluations, not universal confidence assumptions.

### Optional semantic provider contract

Inference is **not** in the required execution path. A future Jev or SystemOne adapter may choose **one eligible capability ID or abstention**; it cannot generate URLs, methods, credentials or a sequence of downstream calls. Revalidate the chosen ID against the caller-filtered catalog and fail closed on provider errors, invalid output and uncertain results.

**Candidate limits:** The published SystemOne/Jev-compatible Choice subset permits at most 16 criteria per question; reserve one for `none_of_the_above` (at most 15 real candidates). Do not silently truncate a larger catalog. Stop/clarify or add evaluated deterministic preselection in a future implementation. Model score semantics, backends and licenses vary; never imply SystemOne's local weights are hosted Jev or that SDK compatibility proves decision accuracy.

Detailed provider examples, transport limits and comparative evaluation notes are **reference-only**, archived at [SystemOne patterns](archive/SYSTEMONE_PATTERNS.md). They do not expand the v0.1 spike or make a model key mandatory.

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

**Plan semantics:** A2A requests marked `utterroute.mode=plan` may run deterministic selection or optional inference. They make **zero** outbound REST business calls, **zero** outbound A2A target calls and (in embedded SDK mode) **zero local handler invocations**. Document data-processing implications for deployers.

**Failure semantics:** If a GET was sent but transport failed before a response, report uncertainty; never claim the backend was not called.

## 9. Observability

Emit structured request ID, timestamp, connection ID, selected operation ID (when any), route engine/version, phase durations, outcome, and sanitized error code. Suppress raw prompts, request/response bodies and credentials by default. Correlation headers to upstream services may be forwarded only under explicit static policy.

For evaluation, create a labeled fixture set with expected operation, expected arguments, and expected abstentions. Publish route accuracy, end-to-end correctness, false-execution/abstention rates, p50/p95 latencies and cost. Compare against small-LLM function calling. Performance figures remain targets until benchmarked.

## 10. Verification and release gates

**The active S0–S4 implementation/test plan is [PLAN.md](PLAN.md).** This section is the persistent technical verification contract, not a second execution checklist.

- **Protocol:** Independently pinned A2A v1 HTTP+JSON client verifies discovery, `SendMessage`, `GetTask`, Task/Artifact and `INPUT_REQUIRED` continuation. Only advertise implemented capabilities and bindings.
- **Resolution:** Authenticated caller eligibility precedes selection. Explicit/deterministic resolution calls **zero** inference providers. Unknown, ambiguous, unauthorized or model-invalid matches cause **zero** target invocations.
- **REST:** Imported spec subset and scalar argument validation, GET allowlists, trusted origin, URL/path encoding, bounded response/timeout and no redirects. `utterroute.mode=plan` executes **zero** upstream calls.
- **Remote A2A:** One initial authorized `SendMessage`, direct Message/completed Task mapping, bounded child Task retrieval and continuation, caller correlation and no blind retry on uncertain delivery. Treat delegated side effects as unknown until explicitly approved.
- **Isolation and resource limits:** Task TTL/capacity and caller ownership, replay/concurrent continuation, no credential forwarding, sanitized logs, bounded queues/body sizes, DNS/SSRF safeguards and controlled rejection under overload.
- **Negative cases:** Hostile OpenAPI and Agent Cards, prompt injection, forbidden methods/destinations, oversized artifacts, missing IDs, recursive agent hops, upstream failures and private task disclosure.
- **Measurements:** Compare direct REST and direct A2A invocation to deterministic UtterRoute invocation, recording end-to-end and in-process p50/p95, correctness, abstention and setup effort. Semantic evaluation is a **separate optional** experiment after the baseline.

**Public deployment is not in the spike's supported scope.** Network DDoS/WAF/TLS and generic API management belong to existing infrastructure. Production exposure additionally requires demonstrated application-level authorization, egress isolation, overload behavior and protocol/security tests.

## 11. Suggested repository structure (future, not present)

```text
README.md
AGENTS.md
LICENSE
docs/
  PRD.md        # product contract and user journeys
  ARCH.md       # this technical contract
  PLAN.md       # active spike work and stop/go evidence
  adr/          # historical architectural decisions
  archive/      # deferred HTTP and SystemOne reference notes
go.mod
cmd/utterroute/main.go
internal/
  a2a/          # inbound protocol and owned Task correlation
  resolver/     # caller eligibility + deterministic-first resolution
  catalog/      # REST operations and registered A2A capabilities
  targets/      # thin rest.go and a2a.go clients; optional later local.go adapter
  config/       # fixed destinations, limits, secret references
examples/{billing-api,research-agent}/
tests/{integration,evaluation}/
Dockerfile
```

The first Go vertical slice may use fewer packages. Do not implement a generic proxy, multi-agent orchestrator or backend plugin framework merely to satisfy a hypothetical future architecture. Embedded SDK support is a **post-S4 experiment**, not part of the two-target baseline.

Later releases may consider a secondary custom HTTP interface, streaming/cancel/push, durable tasks and approved writes only after explicit product decisions in the [PRD](PRD.md) and, when warranted, new ADRs. See the [archived HTTP proposal](archive/HTTP_API_FUTURE.md) for historical context.
