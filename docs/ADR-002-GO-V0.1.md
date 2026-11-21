# ADR-002: Use Go for the UtterRoute v0.1 reference implementation

**Status:** Proposed for acceptance  
**Date:** 2026-11-21  
**Scope:** v0.1 implementation language and runtime (not a change to product goals)  
**Related:** [ADR-001: A2A-first](ADR-001-A2A-FIRST.md) · [PRD](PRD.md) · [SPEC](SPEC.md) · [SystemOne patterns](SYSTEMONE_PATTERNS.md)

## Context

UtterRoute v0.1 must expose an **A2A v1 HTTP+JSON agent**, accept `SendMessage`, manage scoped tasks and clarification, import a local OpenAPI-described REST API, select one allowlisted GET operation using Jev, validate scalar arguments, then produce a Task with a structured result artifact.

The initial PRD/SPEC suggested Python + FastAPI without a binding decision. We evaluated **Go** and **Rust** as native compiled implementation options. Both now offer official A2A v1 SDK support, and neither requires us to implement A2A protocol bindings from scratch.

This project is primarily a **network-facing orchestration and policy gateway**, not a local inference engine. We want small reviewable implementation PRs, straightforward cross-platform binaries, predictable I/O and a contributor-friendly codebase.

## Decision

**Implement UtterRoute v0.1 in Go**, using the [official A2A Go SDK v2](https://github.com/a2aproject/a2a-go) for A2A messages, Agent Cards, Task types, and the initial HTTP+JSON server adapter. Use Go's standard HTTP packages and [kin-openapi](https://github.com/getkin/kin-openapi) for importing/validating the supported OpenAPI 3.0/3.1 subset. Keep Jev and a future SystemOne connection as **out-of-process HTTP adapters** behind a minimal, provider-neutral `Router` interface.

The SDK's current [`go.mod`](https://github.com/a2aproject/a2a-go/blob/main/go.mod) requires **Go 1.26 or newer**. The first code PR must pin a tested A2A Go SDK v2 release in `go.mod` and pin the corresponding Go toolchain in CI. Do **not** follow the SDK's `main` branch at build time. Record the exact tested A2A protocol/SDK version combination when implementation is introduced.

**This is a Go-only reference implementation decision, not a mandate that API clients or downstream integrations use Go.** The external protocol remains A2A. The later native HTTP adapter, if shipped, invokes the same core.

## Evidence and trade-offs

| Criterion | Go (selected) | Rust (considered) |
| --- | --- | --- |
| Official A2A v1 integration | [a2a-go/v2](https://github.com/a2aproject/a2a-go), with REST/HTTP+JSON, JSON-RPC, gRPC and client/server APIs | [a2a-rs](https://github.com/a2aproject/a2a-rs), with protocol crates and Axum REST/JSON-RPC server |
| OpenAPI 3.0/3.1 loading | [kin-openapi](https://github.com/getkin/kin-openapi) provides importer, validation and typed schema facilities | Several crates; requires a specific parser/validator evaluation |
| Gateway concurrency | Standard library `net/http`, goroutines, contexts, familiar profiling | Tokio/Axum, excellent performance but greater language/runtime learning curve |
| Distribution | Single compiled binary and small container | Single compiled binary and small container |
| Future local inference | Use SystemOne via HTTP or introduce a separate service | Easier to integrate with Rust-native inference libraries directly |
| Primary trade-off | Go GC, less low-level control, latest official SDK requires a new Go toolchain | More complex ownership/async idioms and potentially slower onboarding |

There is **no benchmark evidence** that Go is faster or more correct for UtterRoute. We choose it for implementation simplicity and fit to the I/O-bound workload. Reconsider Rust only after a measured need for embedded local inference, hard memory ceilings or other concrete constraints.

## Reference stack

| Responsibility | Proposed library / technique | Boundary |
| --- | --- | --- |
| A2A messages and server | `github.com/a2aproject/a2a-go/v2/a2asrv` using `a2asrv.NewHandler` and `a2asrv.NewRESTHandler` | A2A HTTP+JSON is **the only v0.1 public business protocol** |
| HTTP serving | Standard `net/http` + context timeouts | No independent `/v1/execute` surface in v0.1 |
| Task persistence | SDK [`a2asrv/taskstore`](https://github.com/a2aproject/a2a-go/tree/main/a2asrv/taskstore) or a compatible bounded wrapper | Authenticate callers, bind task owner, enforce TTL/size, continuation and replay rules |
| OpenAPI importer | `github.com/getkin/kin-openapi/openapi3` | Support a **strict subset** of local OpenAPI 3.0/3.1; disable remote reference fetching |
| Inference | `net/http` client to TypeSafe Jev; optional later SystemOne adapter | Router returns registered operation ID or abstention; no model-defined method/URL |
| Input and output types | Standard Go types + explicit validation | Scalar arguments only; canonical operation IDs |
| Upstream REST execution | Separate hardened `http.Client`/transport | Static configured origins, GET allowlist, DNS/SSRF protection, no redirects, bounded responses |
| Config | Local YAML or JSON, environment variables for secrets | Exact format to settle with first implementation; never accept secret values in prompts |
| Telemetry | `log/slog` and request IDs | Redacted structured logs; exclude raw prompts and provider secrets |
| Testing | `testing`, `httptest`, race detector and independent A2A client | Wire conformance **and** end-to-end operation safety tested separately |
| Distribution | Go toolchain, Docker multi-stage build | Pin versions and ship a minimal self-hosted image |

**SDK security caveat:** The SDK's in-memory task store supports an optional caller `Authenticator`; when absent, its default user identity is empty. That **is not suitable as tenant/caller authorization**. UtterRoute must explicitly bind a verified caller identity before any task create/read/update, and implement bounded retention (e.g., a policy-aware wrapper if the SDK does not provide TTL itself). Do not assume an SDK implementation guarantees task isolation, public deployment security, or safe replay behavior.

**Licensing:** UtterRoute remains MIT-licensed. The official A2A Go SDK is Apache-2.0-licensed, compatible with an MIT project when its applicable copyright/license/notice obligations are respected. No A2A or model library code is copied into this ADR. Model weights and hosted inference terms are independent.

## Architecture and boundaries

```text
A2A HTTP+JSON
     |
     v
internal/a2a    -- SDK adapter, Agent Card, SendMessage, GetTask
     |
     v
internal/tasks  -- authorized caller, task TTL, continuation, no replay
     |
     v
internal/gateway
     +--> internal/openapi      -- imported, allowlisted GET operations
     +--> internal/router       -- interface; Jev HTTP adapter first
     +--> internal/arguments    -- scalar extraction and schema checks
     +--> internal/policy       -- caller/operation/destination checks
     +--> internal/execution    -- one verified outbound business GET
     |
     v
A2A Task / Artifact
```

The **business gateway** does not import A2A SDK types. Translate at the `internal/a2a` boundary; expose a core use-case API that can later be reused by another ingress adapter. In particular, the A2A Task state is never a substitute for operation authorization.

### Minimal typed router contract (illustrative)

```go
type Candidate struct {
    ID          string
    Description string
}

type RoutingDecision struct {
    OperationID string // empty means abstain
    Score       *float64
    Reason      string
}

type Router interface {
    Route(ctx context.Context, instruction string, candidates []Candidate) (RoutingDecision, error)
}
```

Router inputs are already permission-filtered candidates. Jev results remain untrusted. Validate the chosen ID against the exact supplied candidate set, then separately resolve/validate arguments and execution policy. A test fake implements the same interface without contacting external providers.

For larger catalogs, the documented SystemOne/Jev-compatible Choice subset may cap the number of options. Do not silently truncate candidates; implement a measured candidate-retrieval stage or abstain until supported. See [SystemOne patterns](SYSTEMONE_PATTERNS.md).

## Implementation sequence

1. **PR: Go scaffold + A2A hello world.** Add `go.mod`/pinned SDK and toolchain, `cmd/utterroute`, `internal/a2a`, Agent Card, `SendMessage`, `GetTask`, basic task with a deterministic artifact, tests and CI. Verify protocol interop using an independent A2A client.
2. **PR: Task isolation/lifecycle.** Implement caller binding, bounded/TTL task store, continuation, terminal-state/idempotency guards, authentication tests and fail-closed error responses.
3. **PR: OpenAPI registry + mock billing.** Parse local spec, reject unsupported features, assemble stable candidate registry with explicit GET allowlist and fixed destinations.
4. **PR: Routing/validation/execution.** Jev adapter, scalar argument resolution, single GET executor, plan-only metadata behavior, task states/artifacts and no-upstream-call tests.
5. **PR: Release hardening.** OpenAPI security tests, SSRF/DNS/redirect handling, rate limits, provider failures, A2A conformance, full routing benchmark, Docker and operator docs.

**Explicit non-goals for v0.1:** native inference embedded into the Go process, an independent HTTP execution endpoint, write operations, multi-step workflows, durable distributed task queues, streaming or multi-protocol A2A support.

## Acceptance criteria

- [ ] First implementation PR pins official `a2a-go/v2` and the required Go toolchain in `go.mod` and CI.
- [ ] An independent A2A v1 client can discover the Agent Card, send a message, retrieve the Task, and continue an input-required task.
- [ ] Core routing/execution tests do **not** depend on A2A types or hosted model access.
- [ ] Unknown operations, invalid parameters, ambiguous requests and prohibited writes never call an upstream business API.
- [ ] Per-task at-most-one-GET semantics, task ownership, TTL and replay constraints have automated tests (including `go test -race`).
- [ ] The model and business HTTP clients use independent, controlled destinations, timeouts and credential sources.
- [ ] `README` and configuration examples accurately distinguish proposal from shipped features until implementation lands.

## Follow-up decisions

Before the first implementation PR, settle only the minimum details not answered by this ADR: exact pinned A2A Go SDK release/toolchain, local config encoding, and the task-store TTL/limits. These should be captured in code/tests rather than reopen the Go-vs-Rust decision.

## Sources

- [Official A2A Go SDK and examples](https://github.com/a2aproject/a2a-go)
- [A2A Go module declaration](https://github.com/a2aproject/a2a-go/blob/main/go.mod)
- [SDK in-memory task store and Authenticator](https://github.com/a2aproject/a2a-go/blob/main/a2asrv/taskstore/inmemory.go)
- [Official A2A Rust SDK](https://github.com/a2aproject/a2a-rs)
- [kin-openapi and supported OpenAPI versions](https://github.com/getkin/kin-openapi)
- [A2A specification](https://a2a-protocol.org/v1.0.0/specification/)
