# Mainframe Abstraction Platform — Design Detail
*Companion to summary.md — focused on Layer 3 (screen-client libraries) and Layer 4 (mainframe integration platform)*
*Last updated: August 4, 2026*

This document captures design decisions from a working thread that dug into how the screen-client libraries and the mainframe integration platform actually interact. It assumes the context in `summary.md` (problem statement, layered architecture, ownership principles) and should be read alongside it.

## Deployment Model: Platform as a Standalone Service (Option B)

Decided: the platform is a **standalone, network-addressable service**, not a library embedded in each domain service.

- Domain services call the platform remotely (REST and/or events — see below).
- The platform is the one that actually talks to each mainframe, including through the API Gateway.
- At the gateway, mainframe-bound calls present as coming from the **platform's own service identity** (one cert/service account), not from individual domain services.
- Per-domain attribution for audit/logging is preserved via an on-behalf-of header/claim (e.g. `X-Origin-Domain`) that the platform forwards, separate from gateway-level auth.

This is what makes centralized window-awareness, circuit-breaking, and idempotency state possible — there is one process managing that state, not N independent library instances that would otherwise need to share state through a database anyway.

## Screen-Client Libraries: Where They Live

Screen-clients remain **domain-owned code, deployed as part of each domain service** (a library/package), not components running inside the platform. This holds under Option B — the screen-client runs inside the domain service's process and makes a network call out to the platform.

- Screen-client responsibilities: request building, response parsing, error translation — all mainframe-screen-specific.
- Platform responsibilities: durability, retry, window-awareness, idempotency, circuit breaking, protocol execution, observability — all transport/reliability-specific, with zero knowledge of what a payload *means*.
- The platform treats `screenPayload` as an opaque blob. It does not need to know screen/transaction type — window-awareness is scoped **per mainframe system**, not per screen.

### OpenAPI / codegen implications

Where a mainframe exposes an OpenAPI spec (REST channel only — not applicable to TCP/IP screens), codegen splits into two concerns:

1. **Mainframe's spec → models only** (request/response types), not a transport client. The generated transport client pointed at the mainframe's host no longer applies, since the domain service doesn't call the mainframe directly.
2. **Platform's own spec → full generated client** (`invoke`, `getStatus`, `resume`). This is now the actual network-called API domain teams codegen against.

TCP/IP screens have no OpenAPI spec to begin with; those screen-clients are presumably hand-built or generated from copybooks/schemas already, and that doesn't change with the platform in the middle.

## The `invoke` Contract

### Entry Point 1 — REST
```
POST /invocations
{
  targetSystem:        string,          // e.g. "mainframe-1"
  protocol:            "tcpip" | "rest",
  screenPayload:        bytes/base64,    // opaque to the platform
  idempotencyKey:       string,          // scoped per screen-step, caller-generated
  sagaCorrelationId:    string,          // ties together one saga's invocation chain
  executionMode:        "async" | "sync-fail-fast"   // default: "async"
}
```

### Entry Point 2 — Event (async mode only)
```
topic: platform.invocations.submit
{
  targetSystem, protocol, screenPayload,
  idempotencyKey, sagaCorrelationId
  // no executionMode — submitting by event implies async
}
```

Both entry points write to the **same outbox record** before anything else happens. Same idempotency check, same window check, same downstream pipeline — the entry point only affects how the call got in the door, never its guarantees.

`sync-fail-fast` is REST-only. There is no way to get a synchronous immediate answer from a fire-and-forget event submission without reintroducing polling under a different name.

### Execution Modes

**`async`** (default):
- Window closed → queue durably, dispatch on reopen
- Transient failure → retry per policy
- Completion signaled via event; `getStatus` available as fallback

**`sync-fail-fast`** (REST only):
- Window closed → reject immediately, do not queue
- Mainframe unreachable / circuit open → reject immediately
- Transient error → reject immediately, no platform-level retry
- Still durably records the outbox entry with the idempotency key before attempting dispatch — needed to detect ambiguous outcomes if the caller retries manually

### Responses

**`sync-fail-fast`:**
```
200: { invocationId, status: "succeeded", result }
409: { invocationId, status: "rejected", reason: "window_closed", retryAfter }
409: { invocationId, status: "rejected", reason: "circuit_open", retryAfter }
502: { invocationId, status: "rejected", reason: "mainframe_error" }
504: { invocationId, status: "rejected", reason: "ambiguous" }   // no ack received — caller should call getStatus rather than blindly retry
```

**`async`:**
```
REST:  202 { invocationId, status: "queued" }
Event: no synchronous response — the accepted event is the ack
```

### Event Lifecycle (async calls)

```
1. platform.invocations.accepted
   { invocationId, sagaCorrelationId }
   — emitted immediately on durable receipt, for ALL async calls
     (REST and event-submitted alike, for consistency — even though
     REST callers also get this info synchronously in the 202 body)

2. platform.invocations.completed  { invocationId, sagaCorrelationId, status: "succeeded", result }
   platform.invocations.failed     { invocationId, sagaCorrelationId, status: "failed", reason }
   — emitted once, when the call actually resolves
```

### Fallback (all modes)
```
GET /invocations/{invocationId}   → current status — reconciliation, debugging,
                                     or a saga that missed an event
```

## Naming: `sagaCorrelationId`

Renamed from `correlationId` to `sagaCorrelationId` to avoid collision with the org's other, already-overloaded uses of `correlationId` (request tracing, message-bus correlation, etc.). Applies wherever the original field appeared: the invoke contract, all platform events, and the saga state store.

## High-Level: Remaining Platform Components

These were discussed at a high level in this thread; each needs a follow-up deep-dive pass.

### Window / Availability Status
Not a static calendar lookup alone — a **composite, dynamically-maintained status per `targetSystem`**, fed by three sources:
- **Defined windows** (static config) — known recurring/scheduled batch outages
- **Active health checks** — platform polls each mainframe ("are you up") on a regular interval, independent of the calendar; catches unplanned outages and windows that overran
- **Event-driven status** — for mainframes that emit their own up/down signals, the platform subscribes and updates status immediately

All three resolve to one current status per system, which is what the window-check pipeline step queries — it doesn't matter which source produced it.

**Open question:** should an early "available" signal (health check or event) actually resume dispatch before the calendar's official window-open time, or does the calendar remain authoritative for *when* a window officially opens regardless of what health checks say? This is a real behavioral decision, not just a reporting nuance — revisit before implementation.

### Retry / Circuit Breaker
- Retries apply only to transient failures (timeouts, connection errors) — not mainframe-returned business errors, which the screen-client/saga interprets instead
- One breaker per mainframe connection (not per domain), so one degraded system doesn't throttle calls to the others
- Not yet defined: backoff strategy, trip thresholds, half-open probe behavior

### Observability
- Every invocation traced end-to-end (submit → queue/hold → dispatch → completion), tagged with `invocationId` and `sagaCorrelationId`
- Rolled up per-domain and per-mainframe on a shared dashboard
- Not yet defined: what's alertable (breaker trips, queue depth, hold duration) vs. logged only

### `resume()`
- Takes a `sagaCorrelationId`, returns saga-level state from the saga state store: which steps completed (with results), what's outstanding
- Used when a saga itself is interrupted (e.g. domain service restart mid-saga), distinct from `getStatus` which is scoped to one `invocationId`
- Not yet detailed: exact response shape

## Open Questions Carried Forward

1. Window calendar: authoritative source for schedules — does one already exist in the org (ops team, config, mainframe-published API), or does the platform become system of record?
2. Early-availability behavior: does the platform dispatch early on a positive health signal, or does the static calendar still gate window-open timing?
3. Retry/breaker specifics: backoff algorithm, failure-count/time-window trip thresholds, reset/half-open behavior
4. Observability: alertable vs. logged-only signal list
5. `resume()` response shape
6. `getStatus` response shape — mirror the completed/failed event payload, or leaner?
