# UtterRoute

**Route intent, not endpoints.**

UtterRoute is a proposed **open-source natural-language API gateway**. Register an OpenAPI-described REST service, describe the read operation you want in plain language, and receive either a validated execution plan, the result of an allowed API call, or a clear refusal/clarification.

**Natural language in. Validated API calls out.**

> **Project status: planning / v0.1 specification.** The repository currently contains design documents and an MIT license. The gateway, CLI, Docker image, and HTTP endpoints shown below are **not implemented or available to install yet**. We welcome feedback on scope and architecture before coding begins.

## Why UtterRoute?

OpenAPI tells developers **how** to call an API. UtterRoute aims to help clients say **what they want done** without constructing endpoint URLs and query strings.

The goal is not to build another general chat agent. It is a narrow, inspectable runtime that turns one instruction into **at most one** validated API request. For the first release that means **read-only, allowlisted GET** operations with explicit error handling when the intent or arguments are unclear.

### Example (target v0.1 behavior — illustrative only)

Given a registered API with `GET /invoices?status=...` and an allowlisted `listInvoices` operation:

```http
POST /v1/execute
Content-Type: application/json

{
  "connection_id": "billing-demo",
  "request": "List unpaid invoices",
  "mode": "plan"
}
```

Proposed response:

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
  "execution": {
    "eligible": true,
    "will_call_upstream": false
  },
  "routing": {
    "engine": "jev",
    "score": 0.92
  }
}
```

With `"mode": "execute"`, the gateway would revalidate all parameters and policies, issue the GET to the configured origin, and return the bounded upstream response. The example score is illustrative, **not** a guarantee of execution correctness.

If the request is ambiguous, the required arguments cannot be determined, or the operation is not permitted, UtterRoute must **not execute** the request. `plan` does not contact the upstream business API (but can contact the configured inference provider).

## How it is intended to work

```text
Instruction
    |
    v
Registered OpenAPI operations + operator allowlist
    |
    v
Jev-first intent selection (including abstention)
    |
    v
Constrained argument extraction + schema validation
    |
    v
Execution policy
    +--> plan: return proposed operation, no upstream request
    +--> execute: one validated GET to a configured origin
    |
    v
Result or explicit refusal / clarification + request ID
```

**Jev-first, not Jev-locked:** [TypeSafe.ai's Jev](https://typesafe.ai/) is the proposed initial semantic selection adapter. It is a separate hosted, proprietary inference service requiring a TypeSafe API key. UtterRoute itself is MIT-licensed, and the gateway architecture is intended to support other routing backends later. We do **not** assume Jev can synthesize arbitrary JSON arguments or verify API-side effects.

## Related projects and interoperability

[SystemOne by codesoda](https://github.com/codesoda/systemone) is an independent MIT-licensed **decision-model runtime**, not a natural-language API executor. It exposes a Jev-compatible `POST /v1/systemone` interface for typed Choice/Noul/Score decisions and supports both hosted Jev and alternative local/hosted backends. UtterRoute's distinct responsibilities are OpenAPI ingestion, safe operation/argument compilation, policy, and upstream execution.

Its [documented wire subset](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) and [backend/runtime documentation](https://github.com/codesoda/systemone#about-the-project) are useful reference material for a future optional inference adapter and comparison tests. **No SystemOne dependency or compatibility is implemented in UtterRoute v0.1 yet.** Model predictions, calibration, limits and licensing for downloaded weights vary by backend; a Jev-compatible API does **not** mean model equivalence or locally hosted TypeSafe Jev weights. See [SPEC.md](docs/SPEC.md#optional-systemone-interoperability-reference) for proposed integration constraints. UtterRoute is not affiliated with or endorsed by SystemOne or TypeSafe.

## v0.1 goals

- Import **local OpenAPI 3.x** descriptions and expose only explicitly allowed GET operations.
- Implement `POST /v1/execute` with required `connection_id`, `request`, and `mode: plan | execute`.
- Select among eligible operations using Jev with an explicit no-match/abstain path.
- Extract and validate **simple scalar** path/query parameters; refuse ambiguous or missing required values.
- Never let a model or a user prompt select an arbitrary URL, HTTP method, authentication header, or secret.
- Support bounded upstream GET execution with structured outcomes, timeouts, redacted tracing, and request IDs.
- Provide a reproducible Docker-based example and basic evaluation/security tests **when the implementation lands**.

### Not part of v0.1

Writes (`POST`/`PUT`/`PATCH`/`DELETE`), autonomous multi-step workflows, MCP/GraphQL integration, browser agents, arbitrary request body generation, per-user OAuth, hosted multi-tenancy and production-grade delegated authorization are future possibilities—not launch features.

v0.1 is intended for **local development and controlled evaluation**, not an internet-exposed production gateway.

## Project docs

- **[Product requirements — v0.1](docs/PRD.md):** Audience, problems, scope, user stories, release criteria and risks.
- **[Technical specification — v0.1](docs/SPEC.md):** Proposed architecture, request/response contracts, importer constraints, policy, security and testing.
- **[Reusable SystemOne patterns](docs/SYSTEMONE_PATTERNS.md):** Upstream architecture references, practical reuse map, compatibility limits, tests, and deferred integration.

The docs are proposals open for review. Examples above describe intended interfaces, not currently working APIs.

## Initial development roadmap

1. Mock billing API, validated OpenAPI importer, configuration and explicit operation registry.
2. Router interface + Jev adapter, abstention behavior, constrained parameter handling.
3. Plan/execute HTTP service, upstream GET execution, safe logging and network protections.
4. Docker demo, unit/integration/security tests, evaluation fixtures and first tagged `v0.1.0` release.

The priority is **correct end-to-end behavior and safe refusal**, not a model-only routing benchmark. We intend to report routing accuracy, argument correctness, safe abstention, overall execution success, latency and comparison to a small-LLM tool-calling baseline.

## Security model

UtterRoute aims to fail closed. Only developer-configured upstream origins and operations can be called; no model-produced URL or credentials are trusted. A low-confidence or ambiguous request should result in clarification/refusal rather than execution. Planning may send natural-language instructions and selected operation descriptions to the inference provider, which should be considered when handling sensitive data.

Until authentication, authorization, networking and deployment controls have been implemented and tested, **do not expose future development builds on a public network or use them for privileged production operations**. The design explicitly excludes tenant-level delegated authorization in v0.1. See [SPEC.md](docs/SPEC.md) for the planned safeguards.

## Contributing

Early contributions are most useful as design review, representative OpenAPI examples (with no secrets), annotated natural-language evaluation cases, and critiques of the [PRD](docs/PRD.md) or [SPEC](docs/SPEC.md).

Please open a GitHub issue describing the use case and expected behavior before proposing substantial scope expansion. We want to keep the first usable release small and dependable.

## License

[MIT](LICENSE) © 2026 Tom Brandenburg. The gateway's open-source license does **not** license third-party model weights or grant free use of TypeSafe's hosted Jev service. API-provider usage, data-processing terms, and fees are governed separately by their providers.
