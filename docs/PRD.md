# UtterRoute — Product Requirements Document (PRD)

**Release:** v0.1 (proposed)  
**Status:** Draft for discussion; no implementation is shipped  
**Last updated:** 2026-10-18  
**Related:** [Technical specification](SPEC.md) · [Project README](../README.md)

## 1. Summary

UtterRoute is a self-hostable, MIT-licensed **natural-language API gateway**. Developers register an OpenAPI-described REST service, send a plain-language request to one endpoint, and receive either a validated execution plan, an executed read-only API result, or a clear explanation of why the request cannot be executed.

**Promise:** *Route intent, not endpoints.* Natural language in; validated, traceable API calls out.

For v0.1, use a **Jev-first, provider-pluggable** selection layer. Jev selects among permitted operations; deterministic code handles schema validation, authorization, argument checks, and HTTP execution. The router must abstain when it cannot establish a safe, unambiguous call. Jev is a hosted proprietary model; UtterRoute's own code remains MIT-licensed. A TypeSafe API key is required for the first Jev adapter, and third-party data handling must be documented.

## 2. Problem and customer

### Problem

An OpenAPI description documents the syntax of an API but does not let application builders invoke it reliably with ordinary language. LLM tool calling can work, yet often requires bespoke prompts, orchestration, credentials handling, validation, and failure management per API.

### Initial users

- **Primary:** Developers embedding natural-language read operations into a vertical SaaS product or internal tool.
- **Secondary:** API/platform engineers testing a natural-language interface to an existing REST API.
- **Not the v0.1 buyer:** Nontechnical end users looking for a general chatbot or autonomous agent platform.

### Job to be done

> When my team already operates a documented REST API, I want to register it once and submit plain-language read requests, so I can add a controlled natural-language interface without building a bespoke agent.

## 3. Product principles

1. **Safety over apparent cleverness:** Do not guess missing identifiers, invent values, or silently execute a different operation.
2. **Deterministic enforcement:** An AI selection never bypasses allowed operations, identity/policy, or schema validation.
3. **Explainability by default:** Return the operation, arguments, status, and a request ID. Keep sensitive detail out of logs.
4. **Open core in full:** Local/self-hosted mode, routing, validation, and tracing are open source; no artificial paywall on safety.
5. **Model independence:** Jev is the first adapter, not the external API or a permanent architectural requirement.
6. **Small and composable:** Build a gateway, not a full conversation agent, workflow builder, or tool marketplace.

## 4. v0.1 scope

### In scope (P0)

| ID | Requirement | Acceptance criteria |
| --- | --- | --- |
| PRD-01 | Import one OpenAPI 3.x specification from a local file at startup | A sample specification is ingested, resolved, and validated; malformed specs fail with a helpful error |
| PRD-02 | Build a normalized operation registry | Stable operation IDs, human-readable summaries, query/path parameter schema, and HTTP metadata are available internally |
| PRD-03 | Explicit allowlist for executable operations | Only configured operations are eligible; **GET only** for v0.1, regardless of other operations in the specification |
| PRD-04 | Natural-language request endpoint | `POST /v1/execute` accepts a connection ID, request text, and `plan` or `execute` mode |
| PRD-05 | Jev-first intent routing | Select from eligible candidates and an explicit no-match/abstain option; no arbitrary URL construction |
| PRD-06 | Basic argument resolution | Support supported scalar path/query parameters (enum, boolean, numeric, ISO date, unambiguous literal string); reject missing/ambiguous required values |
| PRD-07 | Schema and URL validation | No request leaves the gateway unless operation, arguments, destination, and access rules pass validation |
| PRD-08 | Read-only execution | `execute` forwards validated GET requests to preconfigured origins and returns bounded response data and metadata |
| PRD-09 | Plan mode | `plan` returns intended operation and redacted parameter preview with **no outbound business API call** |
| PRD-10 | Abstention and errors | No-match, low confidence, missing parameter, ambiguous value, denied action, and upstream failure yield machine-readable responses |
| PRD-11 | Local developer experience | Docker-based start, environment-configured provider key and service credentials, sample API, and reproducible demonstration |
| PRD-12 | Basic observability | Structured request ID, selected operation, outcome, duration and redacted diagnostic fields, with no secrets or raw provider credentials |

### Nice-to-have only if P0 is complete (P1)

- Deterministic shortlisting before Jev selection for larger catalogs.
- Optional CLI command to inspect imported operations.
- Minimal routing evaluation harness with versioned labeled requests.
- Pluggable second router for reproducible offline/local test runs (not a promise of equivalent semantics).

### Explicitly out of scope

- Arbitrary POST/PUT/PATCH/DELETE calls, and write approvals or transaction rollback.
- Chaining multiple API operations, autonomous plans, stateful agents, or entity lookup across endpoints.
- GraphQL, MCP server ingestion, voice input, and browser/UI automation.
- Automatic OAuth connection setup, hosted multi-tenant control plane, visual UI, billing, or managed cloud.
- General arbitrary JSON-body synthesis, arbitrary string extraction, or guaranteed resolution of natural-language references.
- Claims of guaranteed correctness, end-to-end subsecond performance, or fully offline Jev inference.

## 5. Core experience

### First run

1. Developer provides an OpenAPI file, explicitly configured origin, and a named connection with authentication configured outside the natural-language request.
2. Gateway validates and registers eligible GET operations.
3. Developer calls `POST /v1/execute` in `plan` mode with a request such as “List unpaid invoices.”
4. Gateway returns the operation (for example `listInvoices`), resolved parameters (`status=unpaid`), and whether it is executable.
5. The same request in `execute` mode calls the configured upstream API and returns the result.

**Important:** The example assumes the registered API actually exposes a `status=unpaid` query parameter. No cross-API customer-ID resolution is implied.

### Failure behavior

- “Cancel invoice 42” → rejected for v0.1: non-GET operation.
- “Fetch the invoice for Alex” with multiple possible customer IDs → `needs_clarification`, no upstream business API call.
- “List unpaid invoices” with no semantically matching operation → `NO_MATCH`.
- Provider unavailable → explicit routing failure; never pick an unverified operation to keep the request moving.

## 6. API-facing behavior

Modes:

- **`plan`**: Resolve and validate without executing the upstream business API.
- **`execute`**: Re-run the same policy/validation checks and perform one allowlisted GET.

Responses must distinguish `planned`, `completed`, `needs_clarification`, and `rejected` / `failed`. A plan is **not** authorization to skip revalidation at execute time. There are no conversational side effects.

See [SPEC.md](SPEC.md) for proposed request/response shapes and error codes. Interfaces are contracts to implement, not available endpoints today.

## 7. Success metrics and validation

The primary metric is **correct, safe end-to-end execution**, not the router's top-1 classification rate.

| Area | Release evaluation target (initial, to validate) |
| --- | --- |
| Operation routing | ≥95% on a held-out set of supported, unambiguous single-step cases |
| Full request success | ≥90% on the same supported cases, including parameter correctness and API result |
| Unsafe side effects | Zero non-GET requests; zero unauthorized destinations in integration/security tests |
| Ambiguous requests | Abstain or request clarification rather than execute the wrong operation |
| Latency | Report p50/p95 independently for routing, parameter resolution and end-to-end gateway overhead; compare against a small-LLM baseline |
| Developer setup | A fresh developer can follow documentation from clone to sample `plan` and `execute` within 10 minutes once dependencies/keys are available |

Targets are hypotheses, not measurements. Benchmark samples must include paraphrases, out-of-domain requests, missing parameters, swapped operations, provider failures, adversarial instructions, and identical inputs with reordered routing candidates. Evaluate on fixtures with known ground truth and preserve regression data.

## 8. Milestones

1. **Foundation:** OpenAPI import, eligibility policy, registry, mock upstream and Docker wiring.
2. **Reasoning and compilation:** Jev adapter, operation selection, scalar argument resolver and schema checks.
3. **Gateway runtime:** `plan` / `execute`, timeout handling, redacted traces and stable errors.
4. **Release gate:** Sample app, integration/security tests, route-quality baseline, documentation and versioned configuration.

## 9. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Jev may choose a plausible but wrong operation | Restrict choices to eligible catalog; explicit abstain; benchmark and calibrate thresholds |
| Jev is not an arbitrary structured-output generator | Keep parameter parsing/validation in code; abstain rather than fabricate unsupported data |
| API docs are incomplete or misleading | Allow a curated operation allowlist and human-edited descriptions; fail closed |
| SSRF and credential leakage from imported specifications | Pin outbound origins, block redirects to untrusted hosts, prevent arbitrary URLs/headers, and redact secrets |
| Imported spec or upstream response includes prompt-injection text | Treat descriptions/responses as untrusted data; never grant authority over policies |
| Provider cost, latency, outage or licensing changes | Provider interface, bounded timeouts, version pinning, no silent risky fallback |
| GTM becomes too broad | Begin with one use case and a small set of business APIs; validate with embedding developers |

## 10. Decisions still open

- First reference implementation language and framework (proposal: Python + FastAPI, pending maintainer decision).
- Whether `plan` should expose raw argument values or a redacted preview by default.
- Which OpenAPI schema features and remote references are supported in the first importer.
- Default routing-confidence policy; must be calibrated from fixtures rather than treated as a probability of business success.
- Whether API calls require a local developer token by default, even for localhost deployments.

**Definition of done:** A new contributor can run the documented demo against a mock OpenAPI REST server, plan and execute an allowlisted GET through Jev, observe a trace, and demonstrate that ambiguous/denied inputs do **not** execute a backend operation.
