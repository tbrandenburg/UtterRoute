# Future native HTTP interface (secondary; not in v0.1)

> **Archived reference, not current implementation scope.** Use [PRD](../PRD.md), [ARCH](../ARCH.md), and [PLAN](../PLAN.md) for the active decisions and implementation steps.

**Status:** Deferred, non-normative draft for v0.2 or later  
**Related:** [A2A-first ADR](../adr/ADR-001-A2A-FIRST.md) · [v0.1 A2A SPEC](../ARCH.md) · [PRD](../PRD.md)

This preserves the initial proposal for `POST /v1/execute` so the community can review a future **optional convenience HTTP interface**. **UtterRoute v0.1 is A2A-first:** it implements an A2A Agent Card, `SendMessage`, and `GetTask` before any custom API. This draft is **not** an implemented API, a v0.1 deliverable, or an alternative authorization path.

The future native HTTP adapter must invoke the same internal routing, policy, validation, and execution service as A2A. It cannot bypass task/caller policies, allow additional HTTP methods by default, or introduce untrusted target URLs. The older examples below are preserved as a design starting point; their status codes, default mode, schema, and field naming require fresh review when scheduled.

---

## Archived v0.1 HTTP proposal (now v0.2+ design sketch)

### `GET /healthz`

Reports local process liveness; does not contact the Jev provider or upstream API.

### `POST /v1/execute`

Content type: `application/json`.

```json
{
  "connection_id": "billing-demo",
  "request": "List unpaid invoices",
  "mode": "plan"
}
```

| Field | Required | Behavior |
| --- | --- | --- |
| `connection_id` | Yes | Exact name of a preconfigured connection, not a URL |
| `request` | Yes | Nonempty natural-language instruction within configured size bounds |
| `mode` | Yes | `plan` or `execute`; absence must not default to execution |

#### Example: plan response (`200`)

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
  "execution": { "eligible": true, "will_call_upstream": false },
  "routing": { "engine": "jev", "score": 0.92 }
}
```

The example assumes a registered API with `GET /invoices` and supported `status` enum. `score` is a model selection score, **not** an independently calibrated probability of request correctness. Preview values may be redacted if the deployment marks a parameter sensitive.

#### Example: execute response (`200`)

```json
{
  "status": "completed",
  "request_id": "req_example2",
  "connection_id": "billing-demo",
  "operation": {
    "operation_id": "listInvoices",
    "method": "GET"
  },
  "result": {
    "upstream_status": 200,
    "content_type": "application/json",
    "body": {
      "invoices": [{ "id": "inv_1", "status": "unpaid" }]
    }
  },
  "duration_ms": 231
}
```

A completed gateway call is not a claim that domain-level postconditions have been verified. Non-2xx upstream **business** statuses should be represented in `result.upstream_status` with a clear completion status; do not confuse them with routing failures. Upstream responses must be type/size bounded; binary payloads are out of scope.

#### Example: clarification (`422`)

```json
{
  "status": "needs_clarification",
  "request_id": "req_example3",
  "error": {
    "code": "MISSING_PARAMETER",
    "message": "A required invoice identifier could not be resolved.",
    "fields": ["invoice_id"]
  }
}
```

#### Error and status codes

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | `INVALID_REQUEST` | Invalid JSON, empty text or bad mode |
| 401 | `UNAUTHENTICATED` | Optional external/internal gateway auth denied |
| 403 | `OPERATION_NOT_ALLOWED` | Forbidden/non-GET/unallowlisted operation |
| 404 | `UNKNOWN_CONNECTION` | Unknown or invisible connection |
| 422 | `NO_MATCH` | No trustworthy operation |
| 422 | `MISSING_PARAMETER` | Unresolved required value |
| 422 | `AMBIGUOUS_ARGUMENT` | Conflicting plausible parameter values |
| 422 | `UNSUPPORTED_ARGUMENT` | Input schema/extraction beyond v0.1 |
| 429 | `RATE_LIMITED` | Quota or rate policy applied |
| 502 | `UPSTREAM_ERROR` | Upstream transport/protocol failure |
| 503 | `ROUTER_UNAVAILABLE` | Routing provider unavailable; nothing executed |
| 504 | `UPSTREAM_TIMEOUT` | Upstream timed out |
| 500 | `INTERNAL_ERROR` | Sanitized unexpected server error |

Non-success payloads use `status: rejected`, `needs_clarification`, or `failed`, plus `request_id` and structured `error`. Never include stack traces, credentials, raw model prompts or private request bodies in public errors.

