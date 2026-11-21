# ADR-001: A2A-first external interface for UtterRoute

**Status:** Proposed for acceptance in v0.1 (documentation decision; implementation not yet shipped)  
**Date:** 2026-11-21  
**Related:** [PRD](PRD.md) · [Technical specification](SPEC.md) · [Future native HTTP API](HTTP_API_FUTURE.md)

> **Scope update (ADR-003):** [Capability resolver, not API gateway](ADR-003-CAPABILITY-RESOLVER.md) refines the target model to include **both REST GET operations and existing A2A agent capabilities**, with deterministic-first resolution and optional inference. This ADR continues to define the primary **inbound A2A protocol** and task lifecycle. Historical references here to one REST connection/GET reflect the original narrower proposal; they are **superseded on target scope and mandatory inference** by ADR-003.

## Context

UtterRoute converts a natural-language instruction into a safe API operation over an existing OpenAPI-described REST service. Our original v0.1 design started from the custom `POST /v1/execute` endpoint and deferred agent interoperability. The intended early adopters, however, are agents and frameworks that need to *discover* and *invoke* business capabilities without learning an ad hoc protocol.

The [A2A protocol v1.0](https://a2a-protocol.org/v1.0.0/specification/) already defines discovery (Agent Card), messages, tasks, artifacts, task retrieval, protocol bindings, and explicit states for additional input. It is a better initial public contract than a proprietary text-to-HTTP envelope.

## Decision

**Make A2A the primary and first implemented public integration surface.** UtterRoute v0.1 advertises an A2A Agent Card and accepts A2A `SendMessage` requests, delivering structured `Task` results for validated read-only calls. The existing native `POST /v1/execute` contract moves to a **future secondary interface (target v0.2)**; see [HTTP_API_FUTURE.md](HTTP_API_FUTURE.md).

A2A is a protocol layer over HTTP, not an instruction to remove the HTTP server. Our first wire binding is **A2A HTTP+JSON** (not JSON-RPC): `POST /message:send`, `GET /tasks/{id}`, and an Agent Card at `/.well-known/agent-card.json`. Use a standards-conformant, pinned A2A SDK if feasible; verify behavior against the pinned A2A release and an independent client rather than imitating only example payloads.

All interfaces call the same internal service, which owns OpenAPI discovery, Jev/SystemOne-neutral operation selection, typed argument validation, security policy, upstream GET execution, and tracing. **A2A task state is never the source of API authorization.**

### v0.1 external contract

| Surface | Priority | Behavior |
| --- | --- | --- |
| Agent Card | P0 | Advertise a truthful, bounded protocol-defined Agent Card `skills` entry and supported HTTP+JSON binding; no secret endpoints or operations |
| A2A `SendMessage` (`POST /message:send`) | P0 | Accept a text instruction; default to a read-only execute operation within a configured connection |
| A2A `GetTask` (`GET /tasks/{id}`) | P0 | Retrieve scoped task state/results from a bounded, temporary store |
| A2A continuation via `SendMessage` | P0 | Continue an `input-required` task using its `taskId` and matching `contextId` |
| Structured result artifact | P0 | Return the selected operation, sanitized arguments, upstream HTTP status, bounded result, and correlation ID |
| Plan-only variant | P0 | An explicit `message.metadata["utterroute.mode"] = "plan"` avoids upstream business calls |
| Custom `POST /v1/execute` | v0.2 | Secondary convenience API over the **same** execution service; not in v0.1 |
| Streaming, task cancellation, push notifications, persistent/multiworker task queues | Later | Do not advertise support until implemented and tested |

**v0.1 connection scope:** One trusted, operator-configured OpenAPI connection per deployed agent instance is sufficient. Incoming natural language cannot register new origins, select arbitrary destinations, alter credentials, or bypass the GET allowlist. Multiple connections with per-caller selection belong to later work.

### Task lifecycle

An accepted A2A message creates a server-owned task ID and context ID (or uses a permitted existing context). For v0.1 we deliberately return tasks rather than direct messages, even for calls that finish synchronously, so all outputs are retrievable artifacts.

| Core outcome | A2A task state | Side effects |
| --- | --- | --- |
| Valid plan-only request | `TASK_STATE_COMPLETED` with plan artifact | **Zero** business API calls |
| Valid execute request and upstream response | `TASK_STATE_COMPLETED` with result artifact | At most one allowlisted GET |
| Missing/ambiguous required parameters | `TASK_STATE_INPUT_REQUIRED` with clarifying message | Zero business API calls before clarification |
| Unsupported/denied operation | `TASK_STATE_REJECTED` | Zero business API calls |
| Inference/transport failure | `TASK_STATE_FAILED` and sanitized diagnostic | No invented operation; transport failures may be uncertain after send |

A task can move from input-required to working to completed after an authorized follow-up; terminal tasks must not be restarted. `contextId` groups interactions, but it is **not** a credential and does not automatically authorize access to a task. Task access must be checked against the authenticated caller.

The A2A task's completion state means **gateway processing finished**, not that a domain-specific business postcondition was verified. Report upstream status separately in the artifact.

### Plan and execute

Ordinary A2A text requests default to **executing only allowlisted GETs**. Callers that want preview-only behavior set the documented UtterRoute metadata field:

```json
{
  "message": {
    "messageId": "message-001",
    "role": "ROLE_USER",
    "parts": [{"text": "List unpaid invoices"}],
    "metadata": {"utterroute.mode": "plan"}
  }
}
```

This is an UtterRoute-specific optional metadata convention (not an A2A standard property). Its absence does not permit writes. For later mutation support, introduce **explicit approvals** and a separate security design before changing defaults. Plan-only output never serves as an authorization token.

### Runtime and retention

The first release can use a **bounded in-memory task store** with a documented TTL and single-instance deployment for local development and controlled testing. It must implement authorization-aware task retrieval, enforce size limits, avoid storing plaintext credentials, and reject expired/unknown tasks safely. Crash recovery, durable task persistence, cross-replica coordination and resumable asynchronous work are out of scope for v0.1.

The `SendMessage` handler may process a GET synchronously, but must still create and retain the Task before returning. During clarification continuation, ensure one task can execute at most one upstream GET; duplicate/replayed messages must not issue another business call. `messageId` is not inherently an authorization token or a sufficient deduplication mechanism.

### Discovery and security

- Publish a valid A2A v1 Agent Card at the well-known path, declaring `supportedInterfaces` with `protocolBinding: "HTTP+JSON"`, an accurately implemented protocol version, limited `skills`, `defaultInputModes`, `defaultOutputModes`, and **false/absent** unsupported optional capabilities.
- Only advertise publicly discoverable capabilities. Restrict private business operation names and tenant-specific Agent Card `skills` entries to suitably authenticated discovery mechanisms when supported.
- Local development should default to loopback. Public deployment requires HTTPS, authentication, rate limiting, allowed origins, caller-to-task authorization, and egress safeguards. An Agent Card is **not** evidence of caller permissions.
- Never accept a caller-provided URL, HTTP method, credential, or backend operation name as authority. Limit upstream business execution to explicitly configured and allowed GET operations.
- Do not forward raw A2A auth headers to TypeSafe/SystemOne or to upstream APIs. Model providers receive only the minimal necessary instruction and candidate descriptions under documented data-handling terms.
- Do not invent A2A protocol fields; use a vetted SDK or protocol conformance fixtures. For v1, enum values include `ROLE_USER`, `TASK_STATE_INPUT_REQUIRED`, `TASK_STATE_COMPLETED`, and similar fully qualified forms.

## Consequences

**Benefits**
- A standard interface for agent-to-agent discovery and task invocation, avoiding a bespoke client protocol for the first release.
- Natural support for ambiguity via input-required tasks and structured results via artifacts.
- Easier A2A-client interoperability tests and clearer product positioning: "Make your REST API callable by other agents."

**Costs**
- Even read-only requests now require task state, caller-to-task access control, expiration, and protocol-conformance work.
- A2A does not solve application authorization, entity resolution, deterministic parameter extraction, or API transaction safety.
- An HTTP+JSON binding is not equivalent to supporting JSON-RPC or gRPC A2A clients; advertise only the binding implemented.
- The earlier custom REST response contract becomes a deferred convenience interface, not the initial source of truth.

## Implementation sequence (A2A-first)

1. **Contract and scaffold:** Pin the A2A spec/SDK version; minimal HTTP+JSON A2A server, Agent Card, schema fixtures and interoperability smoke tests with a real A2A client.
2. **Tasks and trust boundaries:** Single-instance bounded task store, Task retrieval, caller authorization, TTL, terminal-state and input-required continuation tests.
3. **Gateway foundation:** Sample OpenAPI billing API, importer/allowlist, trusted origin configuration and deterministic fake router.
4. **Semantic execution:** Jev adapter, constrained argument resolver, `plan`/read-only `execute` core service and task-state/artifact mapping.
5. **Release evidence:** End-to-end A2A client demo, negative and security tests, provider comparisons, metrics and documentation. **No proprietary HTTP execution endpoint required for v0.1.**

## Validation and references

- [A2A v1.0 specification](https://a2a-protocol.org/v1.0.0/specification/) (binding, Agent Cards, task lifecycle and security)
- [A2A releases](https://github.com/a2aproject/A2A/releases) (pin a tested patch version; don't silently track `latest`)
- [UtterRoute SPEC](SPEC.md) (security and execution invariants)
- [SystemOne patterns](SYSTEMONE_PATTERNS.md) (optional inference runtime; no protocol coupling)

**Non-goal:** This ADR does **not** implement A2A or claim conformance yet. It changes the intended delivery order and specifies what will need to be built and tested.
