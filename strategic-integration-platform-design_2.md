# Strategic Integration Platform — Design Document
*Bridging modern experiences/domain APIs with legacy mainframe systems of record*
*Last updated: August 4, 2026*

This document consolidates the original architecture decisions with the detailed design work on the Strategic Integration Platform (formerly referred to as the "Mainframe Abstraction Platform") and its interaction with domain-owned screen-client libraries.

---

## Problem Statement

A large financial institution has modern web/mobile experiences (customer-facing and advisor/banker-facing) backed by domain APIs, which in turn must integrate with 3-4 legacy mainframe systems of record. These mainframes:

- Remain the **legal book of record** for regulatory/audit purposes
- Support **writes only during non-batch windows** (writes unavailable during batch processing)
- Support writes via **TCP/IP and REST channels only** (not MQ/events/files)
- Emit **raw SQL Server CDC replication** (mirrored tables, not domain events/change signals)
- Behave like **green-screen terminals** — a single business transaction (e.g., "create account") requires multiple sequential screen-calls, each depending on the prior one's response
- Are hosted in some cases by third parties, with similar access patterns across the 3-4 systems

The business requirement: experiences must be secure, reliable, scalable, and available 24/7 with real-time (or near-real-time) data — even though the back-end systems were not built for this.

Eventual consistency is acceptable for writes. Multiple experiences and multiple domain-owned APIs, built by separate teams across the organization, all need to interact with these same mainframes.

## Core Architectural Decision: Two Distinct, Separable Designs

### 1. Strategic Integration Platform
A shared, cross-cutting, protocol-agnostic **durable invocation platform** that domain teams use to reliably call mainframe screens/transactions. Scope:

- Durable invocation primitive: persist a call (target system, protocol, payload, idempotency key), execute it, track status, retry on transient failure
- **Window-awareness**: knows each mainframe's batch-window calendar; queues calls durably when the target system is in a batch window, dispatches when the window reopens
- **Protocol execution**: maintains TCP/IP and REST clients/connections to each mainframe; domain teams choose the channel per call, platform handles the plumbing
- **State persistence for multi-step sequences**: supports a saga correlation ID so a domain's multi-call sequence can resume mid-way after an interruption (e.g., a batch window closing between calls)
- **Idempotency guarantees** so retries/resumes can't double-apply changes on the mainframe
- **Circuit breaking / bulkheading** per mainframe connection
- **Observability**: unified logging/tracing/status dashboard across all domain teams' mainframe calls

**Explicitly out of scope for the platform:**
- Business/domain logic (which screens to call, in what order, with what data) — owned by domain teams
- Compensation logic for partial failures — owned by domain teams
- Inter-domain communication (REST/GraphQL/Kafka between domain APIs) — uses standard modern resilience patterns, does not route through this platform, since domain APIs don't have batch-window/green-screen fragility

### 2. Domain Services CQRS Implementation
How domain-owned APIs are structured to reconcile sub-second response expectations with multi-second/minute mainframe orchestration. Key decision: **queries and commands must be architecturally separate.**

- **Queries (reads)**: synchronous, sub-second, served from a real-time projection store (see Read Path below) — never a live mainframe call in the request path
- **Commands (writes)**: asynchronous — the API accepts the request, validates it, kicks off a saga, and immediately returns (e.g., `202 Accepted`) with a tracking ID. The saga executes against screen-client libraries + the Strategic Integration Platform, updating status (`submitted → in_progress → confirmed/failed`) as it progresses. Callers poll a status endpoint or receive a push (websocket/SSE/notification/event) on completion.
- A domain's bounded context is likely **multiple APIs**, not a single CRUD-style API (e.g., account lifecycle, account maintenance, statement history, product enrollment may all be distinct APIs within one domain).
- Open question carried into design: whether to standardize a common status/tracking API shape across domains, or let each domain team design it independently.

## Ownership & Governance Principles (apply to both designs)

- **Data ownership defines domain boundaries**, not which mainframe hosts the data. A single domain may own screen-clients spanning multiple mainframes.
- **Screen-client libraries** (request building, response parsing, error translation per mainframe screen/transaction) are owned **per domain area**, not per mainframe and not by the platform team. Whichever domain owns a piece of data owns the screen-clients that read/write it.
- **No domain bypasses another domain's ownership.** If Domain B needs data owned by Domain A, Domain B calls Domain A's API — it never calls Domain A's screen-clients or the Strategic Integration Platform directly for that data.
- Inter-domain communication uses whatever modern pattern fits (REST, GraphQL, Kafka, etc.) with standard resilience practices for that protocol — separate concern from mainframe integration.

## Read Path (data freshness)

CDC (raw SQL Server replication) → **normalization/projection service** (transforms raw replicated rows into canonical domain model, owned per domain) → event backbone → real-time projection store that domain query APIs read from. Nightly reconciliation against EOD/BOD files to catch replication drift. EOD/BOD values are retained **in addition to**, not instead of, real-time data, where the business still needs beginning/end-of-day values.

Neither CDC nor the daily EOD/BOD file transfers pass through the Strategic Integration Platform or the screen-client libraries — they are inbound read-side data feeds, a separate mechanism and ownership path from the write/invocation side, even though both originate from the same mainframes.

## Write Path (summary)

Command → durable outbox with idempotency key → window-aware dispatcher (immediate dispatch if window open; durable queue-and-resume if closed) → domain-owned saga executes ordered screen-calls via the Strategic Integration Platform → status updates surfaced to caller via tracking ID → mainframe response is the only thing that ever means "confirmed," since the mainframe remains the legal book of record.

## Layered Architecture (full stack, top to bottom)

1. **Experience layer** — web/mobile/advisor UIs, talk only to their BFFs
2. **Domain APIs** — one or more per bounded context; own their data end-to-end; communicate with each other via standard modern protocols
3. **Screen-client libraries** — owned per domain, one client per mainframe screen/transaction that domain's data touches, regardless of which of the 3-4 mainframes it lives on
4. **Strategic Integration Platform** — cross-cutting durable invocation, retry, idempotency, window-awareness, circuit breaking, observability; runs as a distributed integration runtime (sidecar or embedded library) alongside each domain service, backed by a lightweight central coordination service for shared state
5. **Read-side projection** — CDC → normalization → event backbone → real-time store, owned per domain, feeding query APIs
6. **Mainframes** — sole legal book of record; all writes eventually land here, even if asynchronously

---

## Strategic Integration Platform — Design Detail

*Focused on Layer 3 (screen-client libraries) and Layer 4 (the platform itself), and how they interact.*

### Deployment Model: Distributed Data Plane, Centralized Control Plane

A single centralized service that proxies all mainframe I/O was considered and ruled out: routing every domain team's mainframe calls through one process creates a central point of failure and a scaling bottleneck. The design instead separates the **data plane** (the actual bytes going to/from the mainframe) from the **control plane** (the shared state that governs whether/how a call proceeds):

- An **integration runtime**, deployed alongside every domain service instance, holds the actual TCP/IP and REST protocol clients and **talks to the mainframe directly** — no network hop through a central process. Mainframe I/O now scales with domain service replica count instead of one shared service's capacity.
- The runtime subscribes to a lightweight status broadcast (window state, circuit breaker state) per `targetSystem` and caches it **locally, in-memory**. Window-checks and circuit-breaker-checks become local cache reads, not remote calls — this is what actually removes the bottleneck.
- The runtime checks idempotency against a shared, fast key-value store before dispatch (a KV lookup, not a proxy of the mainframe conversation itself).
- The runtime emits invocation lifecycle events (accepted/dispatched/completed/failed) to the central tracking store **asynchronously** — the tracking store is a read model built from events, not a gate the call passes through live.
- When a window is closed, the runtime holds the call in a **local durable queue**, retrying dispatch on its own next local check.

**A much smaller central coordination service** now owns only:
- The window calendar and health-check polling, broadcasting status changes to every runtime instance
- Circuit-breaker signal aggregation, if trips should be a shared signal protecting the mainframe holistically rather than purely local per-instance (open design choice — see below)
- The saga/correlation state store and invocation tracking store, populated by events, queried for `getStatus`/`resume`/the dashboard — a read-heavy service, not in the hot path of any mainframe call
- The idempotency ledger, as its own lightweight, highly-available store (e.g. Redis/DynamoDB-class), not bundled into a monolithic platform process

**Deployment packaging of the runtime — sidecar vs. embedded library — is supported as either, not a forced choice.** Screen-clients always talk to "the integration runtime" through one local interface contract; they don't know or care whether that's an in-process function call (embedded library) or a localhost call to a colocated process (sidecar). Two ways to keep both modes consistent: build one core implementation with two packagings (most consistent, more upfront engineering investment), or maintain independent per-mode implementations governed by a shared conformance test suite (cheaper to start, only stays consistent if the suite is actually enforced). Sidecar is the presumed default (polyglot, independently upgradable without a domain team's release cycle); embedded library is an explicit escape hatch for teams where sidecar resource overhead doesn't work.

**Gateway identity:** if using the sidecar packaging, the sidecar can hold the platform's shared service identity/cert — distributed via existing secrets infra, kept out of domain application code — so mainframe-bound calls can still present a consistent "platform" identity at the gateway even though execution is now distributed across every pod. This keeps a centralized gateway identity without routing every call through one shared process. (An embedded library instance would need its own path to this credential, which may weaken this guarantee — see open questions.)

**Resilience posture: the central coordination service being unreachable must not stop domain services from making mainframe calls.**
- **Window/circuit status:** the runtime dispatches on its last-known cached status, or attempts the call blind if no cached status exists yet (e.g. cold start during an outage). This is lower-risk than it sounds: the mainframe itself is the actual enforcement point for batch windows — a dispatch during an actually-closed window is rejected by the mainframe and handled like any other transient failure by the existing retry policy. The platform's window-awareness was always an efficiency optimization, not the thing preventing an incorrect write.
- **Circuit breaking specifically needs more nuance than blanket fail-open:** if central aggregation is unreachable, each runtime instance should fall back to tracking its own recent failures against that mainframe rather than assuming "no signal = healthy" — otherwise a genuinely struggling mainframe gets hammered by every instance simultaneously the moment the shared signal goes dark, which is the exact failure mode circuit breaking exists to prevent.
- **Idempotency-store unavailability is a different risk class**, flagged as an open action item below rather than folded into the same fail-open policy, since skipping that check risks double-applying a mainframe write — a correctness problem, not just a wasted-call problem.

### Screen-Client Libraries: Where They Live

Screen-clients remain **domain-owned code, deployed as part of each domain service** (a library/package), not components running inside the platform. The screen-client runs inside the domain service's process and calls the local integration runtime — an in-process function call if embedded-library mode, or a localhost call if sidecar mode — rather than a network hop to a remote platform service.

- Screen-client responsibilities: request building, response parsing, error translation — all mainframe-screen-specific.
- Runtime/platform responsibilities: durability, retry, window-awareness, idempotency, circuit breaking, protocol execution, observability — all transport/reliability-specific, with zero knowledge of what a payload *means*.
- The runtime treats `screenPayload` as an opaque blob. It does not need to know screen/transaction type — window-awareness is scoped **per mainframe system**, not per screen.

#### OpenAPI / codegen implications

Where a mainframe exposes an OpenAPI spec (REST channel only — not applicable to TCP/IP screens), codegen splits into two concerns:

1. **Mainframe's spec → models only** (request/response types), not a transport client. The generated transport client pointed at the mainframe's host no longer applies, since the domain service doesn't call the mainframe directly.
2. **Runtime's own spec → full generated client** (`invoke`, `getStatus`, `resume`). This is now the actual API domain teams codegen against — a local call (in-process for embedded-library mode, localhost for sidecar mode) rather than a call to a remote shared service.

TCP/IP screens have no OpenAPI spec to begin with; those screen-clients are presumably hand-built or generated from copybooks/schemas already, and that doesn't change with the runtime in the middle.

### The `invoke` Contract

The shape below is unchanged from the original design — what's changed is *where* it's implemented: this is now the local integration runtime's interface (a localhost call for sidecar mode, a direct function call for embedded-library mode), not a call to a remote standalone service. The two entry points still matter, since a domain service may still prefer submitting via its own local event bus over a direct call, even though both are now handled by a runtime instance colocated with that same domain service.

#### Entry Point 1 — Synchronous call (REST-shaped)
```
POST /invocations   (sidecar: localhost call · embedded library: direct function call)
{
  targetSystem:        string,          // e.g. "mainframe-1"
  protocol:            "tcpip" | "rest",
  screenPayload:        bytes/base64,    // opaque to the runtime
  idempotencyKey:       string,          // scoped per screen-step, caller-generated
  sagaCorrelationId:    string,          // ties together one saga's invocation chain
  executionMode:        "async" | "sync-fail-fast"   // default: "async"
}
```

#### Entry Point 2 — Event (async mode only)
```
topic: integration-runtime.invocations.submit
{
  targetSystem, protocol, screenPayload,
  idempotencyKey, sagaCorrelationId
  // no executionMode — submitting by event implies async
}
```

Both entry points write to the **same outbox record** before anything else happens. Same idempotency check, same window check, same downstream pipeline — the entry point only affects how the call got in the door, never its guarantees.

`sync-fail-fast` is REST-only. There is no way to get a synchronous immediate answer from a fire-and-forget event submission without reintroducing polling under a different name.

#### Execution Modes

**`async`** (default):
- Window closed → queue durably, dispatch on reopen
- Transient failure → retry per policy
- Completion signaled via event; `getStatus` available as fallback

**`sync-fail-fast`** (REST only):
- Window closed → reject immediately, do not queue
- Mainframe unreachable / circuit open → reject immediately
- Transient error → reject immediately, no platform-level retry
- Still durably records the outbox entry with the idempotency key before attempting dispatch — needed to detect ambiguous outcomes if the caller retries manually

#### Responses

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

#### Event Lifecycle (async calls)

```
1. integration-runtime.invocations.accepted
   { invocationId, sagaCorrelationId }
   — emitted immediately on durable receipt, for ALL async calls
     (synchronous-call and event-submitted alike, for consistency — even
     though synchronous callers also get this info in the immediate response)

2. integration-runtime.invocations.completed  { invocationId, sagaCorrelationId, status: "succeeded", result }
   integration-runtime.invocations.failed     { invocationId, sagaCorrelationId, status: "failed", reason }
   — emitted once, when the call actually resolves
```

#### Fallback (all modes)
```
GET /invocations/{invocationId}   → current status — reconciliation, debugging,
                                     or a saga that missed an event
```

### Naming: `sagaCorrelationId`

Renamed from `correlationId` to `sagaCorrelationId` to avoid collision with the org's other, already-overloaded uses of `correlationId` (request tracing, message-bus correlation, etc.). Applies wherever the original field appeared: the invoke contract, all platform events, and the saga state store.

### High-Level: Remaining Platform Components

These were discussed at a high level; each needs a follow-up deep-dive pass.

#### Window / Availability Status
Not a static calendar lookup alone — a **composite, dynamically-maintained status per `targetSystem`**, fed by three sources:
- **Defined windows** (static config) — known recurring/scheduled batch outages
- **Active health checks** — platform polls each mainframe ("are you up") on a regular interval, independent of the calendar; catches unplanned outages and windows that overran
- **Event-driven status** — for mainframes that emit their own up/down signals, the platform subscribes and updates status immediately

All three resolve to one current status per system, which is what the window-check pipeline step queries — it doesn't matter which source produced it.

**Open question:** should an early "available" signal (health check or event) actually resume dispatch before the calendar's official window-open time, or does the calendar remain authoritative for *when* a window officially opens regardless of what health checks say? This is a real behavioral decision, not just a reporting nuance — revisit before implementation.

#### Retry / Circuit Breaker
- Retries apply only to transient failures (timeouts, connection errors) — not mainframe-returned business errors, which the screen-client/saga interprets instead
- One breaker per mainframe connection (not per domain), so one degraded system doesn't throttle calls to the others
- Not yet defined: backoff strategy, trip thresholds, half-open probe behavior

#### Observability
- Every invocation traced end-to-end (submit → queue/hold → dispatch → completion), tagged with `invocationId` and `sagaCorrelationId`
- Rolled up per-domain and per-mainframe on a shared dashboard
- Not yet defined: what's alertable (breaker trips, queue depth, hold duration) vs. logged only

#### `resume()`
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
7. Whether to standardize a common status/tracking API shape across domains for the CQRS command side, or let each domain team design it independently
8. **[OPEN — ACTION ITEM] Idempotency-store failure behavior:** if the shared idempotency store is unreachable, should the runtime reject the call outright with a distinct reason (e.g. `reason: "idempotency_unverifiable"`), or is some other fallback acceptable? Unlike window/circuit status, this cannot simply fail open — skipping the check risks double-applying a mainframe write. Needs an explicit decision before implementation, not a default inherited from the fail-open policy on availability status.
9. Sidecar vs. embedded-library packaging: build once with two packagings, or maintain independent implementations against a shared conformance test suite? Also affects whether embedded-library instances get the same gateway-identity handling as sidecars, or need a separate credential path.
10. Circuit-breaker aggregation: centrally aggregated (holistic protection of the mainframe across all domains, dependent on the coordination service) vs. fully local per-runtime-instance (simpler, no shared dependency, less protective when one domain's traffic alone is enough to degrade a mainframe)
11. How locally-held durable queues (per runtime instance) reconcile into the central tracking store/dashboard for a fleet-wide view of pending calls

## Next Steps

Continue design work in two separate, focused threads:
1. Strategic Integration Platform (window calendar authoritative source and early-availability behavior, retry/circuit-breaker tuning, `resume()`/`getStatus` response shapes, observability alerting model)
2. Domain Services CQRS Implementation (status/tracking API shape, saga state modeling, read-model staleness handling, multi-API domain structuring)

This document should be provided as context at the start of both threads.
