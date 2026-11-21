# UtterRoute

**Resolve capabilities, not endpoints.**

UtterRoute is a proposed **A2A-first, open-source capability resolver** across existing REST APIs **and remote A2A agents**. Register operator-approved capabilities, choose one safely, and dispatch through a minimal protocol adapter. It is **not** a general HTTP/API gateway or agent orchestrator.

**A2A messages in. One approved capability out.**

> **Project status: planning / v0.1 specification.** The repository currently contains design documents and an MIT license. The A2A agent, CLI, Docker image, and endpoints shown below are **not implemented or available to install yet**. We welcome feedback on scope and architecture before coding begins.

## Why UtterRoute?

Existing APIs and A2A agents already know **how** to handle requests. UtterRoute focuses on deciding **which approved capability** a caller may invoke, without building another API gateway, traffic proxy or agent-hosting platform.

- **Direct path:** An explicitly named, authorized capability or unambiguous configured rule bypasses inference entirely.
- **Optional semantic path:** Jev (and potentially SystemOne later) may resolve ambiguous requests when evaluated and configured. No model key is required for direct routing.
- **Two target kinds:** A configured **REST GET** operation or a trusted, operator-approved **remote A2A capability**. A remote agent may have side effects; it requires its own capability approval policy.
- **Existing infrastructure stays in place:** Use Kong, Envoy, a cloud API gateway or a plain reverse proxy for ingress, TLS, WAF/DDoS defense and general traffic controls. UtterRoute enforces its own capability/task authorization and outbound constraints.

The first release aims to demonstrate **one REST operation and one independent A2A agent**, with exactly one selected target per incoming task. The differentiator to validate is selection quality, ease of integration and acceptable additional latency—not generic HTTP throughput. See [ADR-003](docs/ADR-003-CAPABILITY-RESOLVER.md).

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

UtterRoute would deterministically select the registered billing capability, validate the arguments and return an A2A **Task** with a result artifact containing the upstream status and bounded response. The client could retrieve that Task with `GET /tasks/{id}`. **No Jev call is necessary when the configured rule uniquely identifies the capability.** A different request may select an operator-approved remote A2A research capability, which UtterRoute invokes through a standard outbound `SendMessage` and tracks as a child Task.

If an invoice identifier is missing, the Task enters `TASK_STATE_INPUT_REQUIRED`; the client can supply the missing value in another `SendMessage` referencing the task. To preview a plan with **no downstream REST or A2A call**, clients can set `message.metadata["utterroute.mode"] = "plan"`.

These examples describe the **intended v0.1 contract, not a working API**. The separate custom `POST /v1/execute` endpoint is **secondary and deferred to v0.2+**. See [ADR-001](docs/ADR-001-A2A-FIRST.md) and [SPEC.md](docs/SPEC.md).

## How it is intended to work

```text
A2A client -> existing ingress / gateway (optional)
                    |
                    v
             UtterRoute A2A adapter
                    |
                    v
          Caller-approved capability catalog
          +-----------------+-----------------+
          | REST GETs       | A2A agents     |
          +-----------------+-----------------+
                    |
                    v
       Explicit / deterministic resolution
                    |
              ambiguous only
                    v
       Optional Jev / SystemOne selection
                    |
                    v
           Policy-checked dispatch
                 /         \
                v           v
     REST: one GET       Remote A2A: one SendMessage
                \           /
                 A2A Task/artifact to caller
```

A2A is the **first public interface**. For **known** capability IDs, direct invocation may be preferable; UtterRoute's purpose is adding discoverability, eligibility and useful selection—not forcing every call through inference.

**Semantic selection is optional:** [TypeSafe.ai's Jev](https://typesafe.ai/) is the first proposed semantic engine, with [SystemOne](https://github.com/codesoda/systemone) a future possible adapter. UtterRoute works with deterministic routing without either provider, and does not embed model weights. [ADR-003](docs/ADR-003-CAPABILITY-RESOLVER.md) defines the build-vs-integrate boundary.

## Related projects and interoperability

[SystemOne by codesoda](https://github.com/codesoda/systemone) is an independent MIT-licensed **decision-model runtime**, not a natural-language API executor. It exposes a Jev-compatible `POST /v1/systemone` interface for typed Choice/Noul/Score decisions and supports both hosted Jev and alternative local/hosted backends. UtterRoute's distinct responsibilities are OpenAPI ingestion, safe operation/argument compilation, policy, and upstream execution.

Its [documented wire subset](https://github.com/codesoda/systemone/blob/main/compat/jev-http-v1.schema.json) and [backend/runtime documentation](https://github.com/codesoda/systemone#about-the-project) are useful reference material for a future optional inference adapter and comparison tests. **No SystemOne dependency or compatibility is implemented in UtterRoute v0.1 yet.** Model predictions, calibration, limits and licensing for downloaded weights vary by backend; a Jev-compatible API does **not** mean model equivalence or locally hosted TypeSafe Jev weights. See [SPEC.md](docs/SPEC.md#optional-systemone-interoperability-reference) for proposed integration constraints. UtterRoute is not affiliated with or endorsed by SystemOne or TypeSafe.

## v0.1 goals

- Expose a bounded A2A v1 HTTP+JSON Agent Card, `SendMessage`, `GetTask`, caller-scoped Tasks and artifacts.
- Register one local OpenAPI **GET** operation and one **approved A2A agent capability**, each with a stable capability ID and trusted target reference.
- Apply caller eligibility **before** selection; use explicit/deterministic resolution by default, with zero inference calls on that path.
- Dispatch **one** approved target per task: validated REST GET **or** remote A2A `SendMessage`, preserving child Task/context and bounded continuation.
- Treat remote agent capabilities as potentially side-effecting and deny unapproved/unknown-risk agent capabilities by default.
- Abstain or request clarification when no unambiguous authorized capability exists; plan-only metadata sends **zero** calls to either kind of target.
- Preserve minimal local trust, workload bounds and outbound safety; rely on existing API gateways/ingress for commodity network protections.
- Measure direct vs deterministic dispatch overhead, correctness and the developer experience. Add **optional** semantic inference only if it demonstrates value.

### Explicitly outside the product boundary

No generic HTTP reverse proxy, API management, WAF/TLS gateway, developer portal, auth/token issuance service, traffic splitting, full agent orchestrator or bundled inference runtime. The custom `POST /v1/execute` API, REST writes, multi-agent fan-out, distributed durable tasks and A2A streaming remain deferred.

v0.1 targets local/controlled evaluation, **not** public production deployment.

## Project docs

- **[Product requirements — v0.1](docs/PRD.md):** Audience, problems, scope, user stories, release criteria and risks.
- **[Technical specification — v0.1](docs/SPEC.md):** A2A Agent Card, SendMessage, tasks, artifacts, importer constraints, policy, security and testing.
- **[Two-target validation spike](docs/PLAN.md):** Implementation slices S0–S4, REST/A2A fixtures, safety gates, benchmark design and continue/library/stop criteria.
- **[ADR-001 — A2A-first interface](docs/ADR-001-A2A-FIRST.md):** Why A2A ships first, task lifecycle, wire binding and scope boundaries.
- **[ADR-002 — Go implementation decision](docs/ADR-002-GO-V0.1.md):** Why Go over Rust for v0.1, proposed SDK/tooling, alternatives, and implementation sequence.
- **[ADR-003 — Capability resolver scope](docs/ADR-003-CAPABILITY-RESOLVER.md):** REST/A2A target model, deterministic-first selection, build-vs-integrate boundary and adoption validation.
- **[Future native HTTP design](docs/HTTP_API_FUTURE.md):** Archived custom `POST /v1/execute` proposal, deferred to v0.2+.
- **[Reusable SystemOne patterns](docs/SYSTEMONE_PATTERNS.md):** Upstream architecture references, practical reuse map, compatibility limits, tests, and deferred integration.

The docs are proposals open for review. Examples above describe intended interfaces, not currently working APIs.

## Initial development roadmap

**Execution plan:** [Two-target validation spike](docs/PLAN.md) provides the actionable, reviewable S0–S4 implementation slices, independent-client demo, explicit scope boundaries and performance/adoption decision gate. The outline below is directional; the spike checklist is authoritative for the first code PRs.

**Implementation direction:** [Go](https://go.dev/) with official [A2A Go SDK v2](https://github.com/a2aproject/a2a-go), [kin-openapi](https://github.com/getkin/kin-openapi) and thin standard HTTP/A2A clients. No application code has shipped yet.

1. **A2A skeleton:** Agent Card, `SendMessage`, `GetTask`, bounded caller-owned Task lifecycle and a real A2A client.
2. **Two target kinds:** Register one billing GET operation and one independently running A2A agent capability, each with configured eligibility and a deterministic match.
3. **Dispatch:** One approved target per task, independent REST/A2A adapter tests, child-task correlation and replay prevention.
4. **Validate the wedge:** Compare direct invocation against the deterministic UtterRoute path for latency, correctness and developer effort; publish honest results.
5. **Semantic experiment (optional):** Add hosted Jev selection only if it improves held-out ambiguous capability selection enough to justify its latency and cost.

Do not ship a general HTTP gateway, model-dependent fast path or `/v1/execute` API as a prerequisite. If the selection value is not measurable, consider a small reusable resolver library rather than growing the service.

## Security model

UtterRoute aims to fail closed. A2A caller authentication, authorization to retrieve/continue tasks, and task retention rules are separate from target credentials. Only developer-configured REST origins/GET operations and explicitly approved remote A2A agents and their registered capabilities may be called; no model-produced URL or credentials are trusted. A low-confidence or ambiguous request should result in clarification/refusal rather than execution. Planning may send natural-language instructions and selected operation descriptions to the inference provider, which should be considered when handling sensitive data.

Until authentication, authorization, networking and deployment controls have been implemented and tested, **do not expose future development builds on a public network or use them for privileged production operations**. The design explicitly excludes tenant-level delegated authorization in v0.1. See [SPEC.md](docs/SPEC.md) for the planned safeguards.

## Contributing

Early contributions are most useful as design review, representative OpenAPI examples (with no secrets), annotated natural-language evaluation cases, and critiques of the [PRD](docs/PRD.md) or [SPEC](docs/SPEC.md).

Please open a GitHub issue describing the use case and expected behavior before proposing substantial scope expansion. We want to keep the first usable release small and dependable.

## License

[MIT](LICENSE) © 2026 Tom Brandenburg. The gateway's open-source license does **not** license third-party model weights or grant free use of TypeSafe's hosted Jev service. API-provider usage, data-processing terms, and fees are governed separately by their providers.
