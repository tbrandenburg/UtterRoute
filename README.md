# UtterRoute

**Route intent, not endpoints.**

UtterRoute is a proposed **A2A-first, open-source natural-language gateway for REST APIs**. Register an OpenAPI-described service and make its safe, read-only operations discoverable and callable by other AI agents through the [Agent2Agent (A2A) protocol](https://a2a-protocol.org/).

**A2A messages in. Validated API results out.**

> **Project status: planning / v0.1 specification.** The repository currently contains design documents and an MIT license. The A2A agent, CLI, Docker image, and endpoints shown below are **not implemented or available to install yet**. We welcome feedback on scope and architecture before coding begins.

## Why UtterRoute?

OpenAPI tells developers **how** to call an API. UtterRoute aims to let other **agents discover what a service can do** and send ordinary-language tasks without knowing endpoint URLs and query strings.

The goal is not to build another general chat agent. It is a narrow, inspectable runtime that turns one instruction into **at most one** validated API request. For the first release that means **read-only, allowlisted GET** operations with explicit error handling when the intent or arguments are unclear.

### Example (target v0.1 A2A behavior — illustrative only)

An A2A client discovers the UtterRoute Agent Card at `/.well-known/agent-card.json`, then sends a natural-language instruction using the A2A **HTTP+JSON** binding. For example, assume a registered billing API supports `GET /invoices?status=...` with an allowlisted `listInvoices` operation:

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
    "parts": [{"text": "List unpaid invoices"}]
  }
}
```

UtterRoute would return an A2A **Task** with a structured result artifact containing the selected API operation, validated arguments, upstream HTTP status and bounded response. The client could retrieve that Task with `GET /tasks/{id}`.

If an invoice identifier is missing, the Task enters `TASK_STATE_INPUT_REQUIRED`; the client can supply the missing value in another `SendMessage` referencing the task. To preview a plan with **no business API call**, clients can set `message.metadata["utterroute.mode"] = "plan"`.

These examples describe the **intended v0.1 contract, not a working API**. The separate custom `POST /v1/execute` endpoint is **secondary and deferred to v0.2+**. See [ADR-001](docs/ADR-001-A2A-FIRST.md) and [SPEC.md](docs/SPEC.md).

## How it is intended to work

```text
A2A client agent
    |
    +--> Discover /.well-known/agent-card.json
    +--> SendMessage /message:send
    |        |
    |        v
    |    UtterRoute task manager
    |        |
    |        v
    |    Registered OpenAPI operations + GET allowlist
    |        |
    |        v
    |    Jev-first routing + argument/schema validation
    |        |
    |        +--> plan: no upstream call
    |        +--> execute: at most one approved GET
    |        |
    |        v
    +<-- Task artifact / input-required / rejected
    |
    +--> GetTask /tasks/{id}
```

A2A is our **first public interface**. A future custom HTTP interface can reuse the same internal service. A2A itself uses HTTP here; we are prioritizing a **standard agent protocol** over a proprietary request envelope.

**Jev-first, not Jev-locked:** [TypeSafe.ai's Jev](https://typesafe.ai/) is the proposed initial semantic selection adapter. It is a separate hosted, proprietary inference service requiring a TypeSafe API key. UtterRoute itself is MIT-licensed and the gateway architecture is intended to support other routing backends later. We do **not** assume Jev can synthesize arbitrary JSON arguments or verify API-side effects.

## Related projects and interoperability

[SystemOne by codesoda](https://github.com/codesoda/systemone) is an independent MIT-licensed **decision-model runtime**, not a natural-language API executor. It exposes a Jev-compatible `POST /v1/systemone` interface for typed Choice/Noul/Score decisions and supports both hosted Jev and alternative local/hosted backends. UtterRoute's distinct responsibilities are OpenAPI ingestion, safe operation/argument compilation, policy, and upstream execution.

Its [documented wire subset](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) and [backend/runtime documentation](https://github.com/codesoda/systemone#about-the-project) are useful reference material for a future optional inference adapter and comparison tests. **No SystemOne dependency or compatibility is implemented in UtterRoute v0.1 yet.** Model predictions, calibration, limits and licensing for downloaded weights vary by backend; a Jev-compatible API does **not** mean model equivalence or locally hosted TypeSafe Jev weights. See [SPEC.md](docs/SPEC.md#optional-systemone-interoperability-reference) for proposed integration constraints. UtterRoute is not affiliated with or endorsed by SystemOne or TypeSafe.

## v0.1 goals

- Expose a compliant **A2A Agent Card** advertising a bounded read-only API skill and the HTTP+JSON binding.
- Handle A2A **`SendMessage`** and **`GetTask`** requests; maintain a bounded local task store with caller isolation and retention limits.
- Map missing/ambiguous arguments to input-required tasks that can be continued; return API results as A2A artifacts.
- Import local OpenAPI 3.x documents and allow **only explicitly enabled GET** operations at fixed developer-configured origins.
- Select operations using Jev with an explicit no-match/abstain path; extract and validate constrained scalar path/query values.
- Support optional plan-only metadata without an upstream business API call; execute at most one allowed GET per task.
- Never let a model or A2A request dictate arbitrary URLs, methods, secrets or upstream credentials.
- Provide a reproducible A2A-client-to-mock-billing demo and honest conformance/security/evaluation tests **when implementation lands**.

### Not part of v0.1

The custom `POST /v1/execute` endpoint, API writes, autonomous multistep workflows, MCP/GraphQL, streaming, task cancellation/push, multi-instance durable tasks, OAuth delegation, hosted multitenancy and additional A2A wire bindings are deferred. v0.1 targets local development and controlled evaluation rather than public production deployment.

## Project docs

- **[Product requirements — v0.1](docs/PRD.md):** Audience, problems, scope, user stories, release criteria and risks.
- **[Technical specification — v0.1](docs/SPEC.md):** A2A Agent Card, SendMessage, tasks, artifacts, importer constraints, policy, security and testing.
- **[ADR-001 — A2A-first interface](docs/ADR-001-A2A-FIRST.md):** Why A2A ships first, task lifecycle, wire binding and scope boundaries.
- **[Future native HTTP design](docs/HTTP_API_FUTURE.md):** Archived custom `POST /v1/execute` proposal, deferred to v0.2+.
- **[Reusable SystemOne patterns](docs/SYSTEMONE_PATTERNS.md):** Upstream architecture references, practical reuse map, compatibility limits, tests, and deferred integration.

The docs are proposals open for review. Examples above describe intended interfaces, not currently working APIs.

## Initial development roadmap

1. **A2A first:** Pinned A2A v1 contract/SDK, Agent Card, `SendMessage`, `GetTask`, and independent-client conformance fixtures.
2. **Task foundation:** Bounded task store, caller ownership, context/task continuity, input-required clarification and replay/TTL tests.
3. **Gateway foundation:** Mock billing API, OpenAPI importer, operation allowlist, trusted-origin config and deterministic fake router.
4. **Jev execution:** Jev adapter, scalar argument resolution, internal plan/read-only execution, A2A artifact/status mapping and telemetry.
5. **Release:** Docker demo, A2A interoperability tests, integration/security fixtures and comparative evaluation.

Only **after the A2A v0.1 release** would we add an optional proprietary HTTP `/v1/execute` adapter sharing the same core service.

The priority is **safe end-to-end agent invocation**, not a routing-only benchmark. We intend to measure task/contract conformance, accuracy, refusal, argument correctness, execution success, latency and comparisons with a small-LLM baseline.

## Security model

UtterRoute aims to fail closed. A2A caller authentication, authorization to retrieve/continue tasks, and task retention rules are separate from configured upstream API credentials. Only developer-configured upstream origins and operations can be called; no model-produced URL or credentials are trusted. A low-confidence or ambiguous request should result in clarification/refusal rather than execution. Planning may send natural-language instructions and selected operation descriptions to the inference provider, which should be considered when handling sensitive data.

Until authentication, authorization, networking and deployment controls have been implemented and tested, **do not expose future development builds on a public network or use them for privileged production operations**. The design explicitly excludes tenant-level delegated authorization in v0.1. See [SPEC.md](docs/SPEC.md) for the planned safeguards.

## Contributing

Early contributions are most useful as design review, representative OpenAPI examples (with no secrets), annotated natural-language evaluation cases, and critiques of the [PRD](docs/PRD.md) or [SPEC](docs/SPEC.md).

Please open a GitHub issue describing the use case and expected behavior before proposing substantial scope expansion. We want to keep the first usable release small and dependable.

## License

[MIT](LICENSE) © 2026 Tom Brandenburg. The gateway's open-source license does **not** license third-party model weights or grant free use of TypeSafe's hosted Jev service. API-provider usage, data-processing terms, and fees are governed separately by their providers.
