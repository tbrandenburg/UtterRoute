# UtterRoute

**Resolve capabilities, not endpoints.**

UtterRoute is a proposed **A2A-first, open-source capability resolver** connecting existing REST APIs and independently running A2A agents. It selects **one eligible, operator-approved capability** and dispatches through a small REST or A2A adapter. It is **not** a general API gateway, reverse proxy or multi-agent orchestrator.

> **Status: planning / documentation only.** This repository does not yet contain a runnable Go service, CLI, Docker image, or implemented A2A endpoints. The examples describe the intended behavior, not a released product.

## The problem

An agent can call a known REST endpoint or a known A2A agent directly. UtterRoute aims to help when it needs **discoverable, caller-authorized selection** across several such capabilities—without bespoke integration for every agent and **without** paying model-inference latency for a known route.

- **Explicit / deterministic first:** A caller selects an authorized capability ID or matches an unambiguous configured rule. **Zero Jev/SystemOne calls.**
- **Optional semantic selection:** Later, an evaluated Jev or SystemOne adapter may choose **one eligible capability ID or abstain** when a deterministic decision is not possible. It never creates a workflow.
- **Two baseline target types:** An allowlisted, validated **REST GET** or one approved outbound **A2A agent delegation**. A remote agent may have side effects; UtterRoute does not control its internal execution.
- **Proposed third integration mode (later):** A Go app embeds an UtterRoute SDK, registers a reviewed typed **`local_handler`** function and exposes the existing A2A interface. No new wire protocol or cross-process Go calls. This remains a post-S4 experiment, not implemented today.
- **Existing infrastructure:** Reuse current gateways/ingress for TLS, WAF/DDoS and generic API management. UtterRoute still enforces caller/task/target authorization, egress policy and bounded work.

```text
Calling A2A agent
       |
       v
UtterRoute A2A interface (Agent Card, SendMessage, GetTask)
       |
       v
Caller-authorized Capability Catalog
       |
       v
Resolver: explicit -> deterministic -> optional semantic -> abstain
       |
       +--> REST Target: one approved GET
       |
       +--> A2A Target: one approved SendMessage / correlated child task
       |
       +--> Local Handler (later embedded mode only): one Go function
       |
       v
Caller-owned A2A Task / Artifact or clarification / refusal
```

**Product terminology:** *Capability* = local selectable unit, *Target* = configured REST API or A2A agent, *Resolver* = selector. The A2A protocol's `AgentSkill` / Agent Card `skills` are descriptive wire metadata, **not** UtterRoute authorization units.

## Current validation: two-target spike

The first implementation experiment will connect one billing API GET and one independently running research A2A agent. A third-party A2A client must discover, invoke and retrieve results from both through UtterRoute. The deterministic path needs **no model provider key**.

Measure direct target invocation versus UtterRoute for latency, routing correctness, safe refusal and developer setup effort. The experiment ends with an evidence-based **continue / narrow to a library / stop** decision. See the active [implementation plan](docs/PLAN.md) for S0–S4, acceptance tests and limits. The same plan describes the **separate optional embedded Go SDK experiment (SDK-0–SDK-3)** after the baseline. This is a **hypothesis to validate**, not evidence that the architecture is already performant.

## Project documentation

| Document | Authority |
| --- | --- |
| [PRD](docs/PRD.md) | Product positioning, non-goals, five user journeys, requirements and adoption criteria |
| [ARCH](docs/ARCH.md) | Architecture, Go boundaries, A2A/REST contracts, security and technical verification |
| [AGENTS](AGENTS.md) | Coding-agent/contributor rules, terminology, working practices and scope constraints |
| [PLAN](docs/PLAN.md) | Two-target S0–S4 execution slices, benchmarks and optional follow-up Go SDK-0–SDK-3 experiment |
| [ADRs](docs/adr/ADR-003-CAPABILITY-RESOLVER.md) | Decision history: A2A-first, Go, capability resolver product boundary |
| [Archive](docs/archive/SYSTEMONE_PATTERNS.md) | Deferred SystemOne and custom HTTP design notes; **not** active implementation requirements |

**Documentation ownership:** PRD owns *why/what*; ARCH owns *how*; AGENTS owns *contributor instructions*; PLAN owns *next execution and measurements*. ADRs explain historical rationale; archive references are nonbinding.

## Development and contributions

The agreed reference implementation direction is **Go**, using the official pinned [A2A Go SDK](https://github.com/a2aproject/a2a-go), standard `net/http`, and [kin-openapi](https://github.com/getkin/kin-openapi) for a constrained REST importer. Versions, development commands and tests will be added in the first implementation PR.

Contributions should start with the [PRD](docs/PRD.md), [ARCH](docs/ARCH.md) and [PLAN](docs/PLAN.md). Prefer a small, tested A2A vertical slice over new gateway or orchestration features. Report unclear product boundaries or unsafe dispatch behavior through GitHub issues.

## License

[MIT](LICENSE) © 2026 Tom Brandenburg. UtterRoute's MIT license does not license third-party models, hosted inference services or target systems; their own terms apply.
