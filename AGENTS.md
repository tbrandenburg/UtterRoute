# Contributor and coding-agent instructions

UtterRoute is currently **documentation-only**. Do not claim an endpoint, binary, Docker image or SDK integration works until code and independent-client tests exist.

## Read these documents first

1. [Product requirements and user journeys](docs/PRD.md) — **why/what**, scope, success criteria and refusal boundaries.
2. [Architecture and technical contract](docs/ARCH.md) — **how**, protocols, authorization, invariants and implementation shape.
3. [Active implementation plan](docs/PLAN.md) — **what to build next**, S0–S4 gates and decision evidence.
4. [Decision history](docs/adr/ADR-003-CAPABILITY-RESOLVER.md) — historical rationale, not an alternative current spec.

**Authority:** Product decisions go in `PRD.md`; technical design in `ARCH.md`; the current spike's checklists in `PLAN.md`. Update those authoritative files when behavior changes; do not copy and maintain conflicting requirements in multiple places. New architectural trade-offs may warrant an ADR in `docs/adr/`. Deferred work and provider reference designs live in `docs/archive/`.

## Non-negotiable boundaries

- UtterRoute is an **A2A-first capability resolver**, **not** an API gateway, reverse proxy, API-management suite or multi-agent orchestrator.
- Use **Capability**, **Target** and **Resolver** for UtterRoute-owned concepts. `AgentSkill` and Agent Card `skills` are **protocol-defined metadata**, not UtterRoute capabilities or enforceable per-skill privileges.
- One incoming task selects **one authorized capability**: at most one permitted REST GET or one initial approved downstream A2A delegation in the S0–S4 baseline; a **post-S4 embedded Go SDK** may invoke one locally registered `local_handler` instead. Remote-agent continuation of the existing child task is not a second independent selection.
- Explicit and unambiguous deterministic routing must make **zero Jev/SystemOne calls**. Semantic inference is an optional, separately evaluated extension; when enabled it may return **one eligible ID or abstain**, never a call sequence.
- Filter by authenticated caller and operator policy **before** choosing a capability. Never trust prompts, client-supplied IDs/URLs, model output, OpenAPI `servers` or Agent Cards as authorization.
- The REST path is allowlisted GET-only; remote agents may have side effects and require separately approved delegation risk. Do not promise control of a remote agent's internal steps.
- Task ownership, replay handling, resource bounds, credential isolation, fixed egress, redirect/DNS/SSRF protection and safe refusals remain UtterRoute responsibilities even behind an existing gateway.
- The proposed `local_handler` is an **in-process Go function**, not a remote Go RPC. Reuse the A2A interface, policy and resolver. Start with reviewed synchronous read-only typed handlers and structured validated arguments; neither Jev nor models fabricate function parameters. Local handler timeouts are cooperative and the embedding application retains its own authorization and data-isolation responsibilities.
- For embedded handlers, borrow the **typed schema-validation discipline** of the official MCP Go SDK, **not MCP itself**: fail fast on unsupported/duplicate registrations, validate serialized outputs and inputs before execution, reject unknown handler-argument fields, keep A2A metadata separate, and map business/panic errors to sanitized **A2A-native** outcomes. Freeze registrations for the initial SDK; see [ARCH](docs/ARCH.md) and SDK tests in [PLAN](docs/PLAN.md).
- Do not add generic HTTP proxying, provider-specific API-management plugins, REST writes, workflow chaining, distributed queues, dynamic Go plugin loading, arbitrary function reflection or agent credential brokerage as spike work.

## Workflow for code changes

1. Work from the **current PLAN slice**. Finish S0–S4 before taking on SDK-0–SDK-3 or optional semantic evaluation, subject to the S4 continue/library/stop decision. Prefer one small, reviewed vertical slice with acceptance tests over abstract frameworks.
2. Keep the Go implementation simple: official pinned `a2a-go/v2` SDK, standard `net/http`, constrained `kin-openapi`, and narrow typed interfaces only when needed. Pin exact versions in the first code PR.
3. Keep A2A SDK types at the inbound/outbound adapters; resolver, catalog and validation should not depend on A2A wire models.
4. Test authorized success, abstention/clarification, wrong caller, duplicate/concurrent messages, slow/oversized targets, uncertain remote delivery and zero downstream execution on denial/plan mode.
5. Use an **independent** A2A client to verify the real A2A v1 HTTP+JSON contract. Do not assume the SDK by itself proves conformance or tenant isolation.
6. Record direct-vs-UtterRoute timings and routing outcomes before adding inference or infrastructure features.
7. Keep docs truthful about what is proposed versus implemented, and update `PRD`/`ARCH`/`PLAN` only where their authoritative contracts actually change.

## Commands and validation

There is **no `go.mod`, application source or runnable test suite yet**. Do not invent successful build/test output. Once S0 creates the Go module, document the exact local/CI commands here (format, unit tests, race tests, integration tests and independent A2A conformance), and run them for every implementation PR.

For documentation-only changes: verify Markdown links/anchors, keep links to decisions in `docs/adr/` and deferred references in `docs/archive/`, and confirm that only one steering document owns each requirement.

## Definition of a useful PR

Explain the user journey or architecture requirement addressed, the **specific PLAN exit criterion**, affected trust boundaries, negative-case tests, actual measured results (when code exists), any documentation updates, and what remains intentionally out of scope. Prefer explicit uncertainty and safe abstention over unverified functionality.
