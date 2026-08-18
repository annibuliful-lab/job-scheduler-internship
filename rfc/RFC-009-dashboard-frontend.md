# RFC-009: Dashboard Frontend

- **Status:** Draft
- **Area:** Operations Dashboard / Frontend Experience
- **Depends On:** RFC-005, RFC-006, RFC-007, RFC-008, RFC-010
- **Primary Consumers:** Operators, developers, support engineers

## 1. Summary

This RFC defines the operator-facing dashboard frontend for the distributed job scheduling monitoring platform.

The dashboard is a presentation and investigation surface over the Query & Investigation context. It does **not** own job, worker, queue, schedule, alert, or PII truth and must not derive authoritative state independently from backend contracts.

Its primary responsibility is to turn monitoring data into an operational experience that helps a user move from:

```text
Is something wrong?
```

to:

```text
What is affected?
When did it become abnormal?
Which run / attempt / worker / queue is involved?
What evidence supports the diagnosis?
Is the data current and complete enough to trust?
What should I investigate next?
```

## 2. Context Boundary

```text
Monitoring / PII / Alerts
          │
          ▼
Query & Investigation Context
          │
          │ read APIs / stream of read updates
          ▼
┌──────────────────────────────┐
│ Dashboard Frontend           │
│                              │
│ navigation                   │
│ visualization                │
│ investigation workflow       │
│ filter / URL state           │
│ freshness presentation       │
│ permission-aware rendering   │
└──────────────────────────────┘
```

The frontend must never:

- read PostgreSQL directly,
- inspect Redis directly,
- determine authoritative run state from client-side heuristics,
- expose data hidden only with CSS or client-side filtering,
- silently replace backend health classifications with its own interpretation.

## 3. Goals

- provide an immediate system-health overview,
- support fast drill-down from aggregate anomaly to affected runs,
- make execution state and monitoring health visually distinct,
- expose freshness, partial-data, and degraded-monitoring conditions,
- provide efficient search and filtering across runs and operational entities,
- preserve investigation context while navigating between screens,
- provide shareable deep links for investigations,
- present PII findings without unnecessary PII exposure,
- remain usable during partial backend degradation,
- provide consistent loading, empty, stale, partial, error, and permission-denied states,
- support accessible operational visualization without relying only on color,
- support real-time whole-platform monitoring with explicit connection, replay, fallback, and resynchronization states.

## 4. Non-Goals

The dashboard is not intended to become:

- a general-purpose BI/dashboard builder,
- a generic log explorer,
- a replacement for Grafana or a full metrics platform,
- a workflow editor,
- a Redis administration console,
- a PostgreSQL administration console,
- a raw PII browsing tool,
- a source of authoritative domain state,
- a mobile-first operational product in the initial release.

Operator mutations such as retry, rerun, replay, cancel, pause, or redrive are outside the initial dashboard contract unless defined by a dedicated command/audit RFC.

## 5. Product Principles

### 5.1 Overview First, Evidence One Click Away

The first view should communicate current operational health without requiring the operator to know a run identifier.

Any abnormal aggregate should lead to the underlying evidence.

Example:

```text
Queue backlog: 1 degraded queue
        │
        ▼
reports queue
        │
        ▼
oldest waiting runs
        │
        ▼
run detail / timeline
```

### 5.2 Facts and Interpretations Must Look Different

The UI must distinguish authoritative facts from derived monitoring interpretations.

Example:

```text
Execution state: RUNNING
Monitoring health: RUN_LOST
```

`RUN_LOST` must not visually replace `RUNNING` as if they were the same state dimension.

### 5.3 Freshness Is Part of Correctness

A dashboard showing old data without saying so is misleading.

The UI must expose freshness when data is delayed, partial, or unavailable.

Examples:

```text
Updated 3s ago
Projection lag: 1.2s
Redis queue sample: 18s old
Monitoring degraded: event ingestion delayed
```

### 5.4 Investigation Context Must Survive Navigation

Time range, filters, search terms, sort order, and selected operational scope should be preserved where practical when an operator drills into a detail screen and returns.

### 5.5 URLs Are Investigation Artifacts

Important views should have stable, shareable URLs.

A shared URL should be able to preserve enough state to reproduce an investigation, such as:

```text
time range
queue
job type
execution state
monitoring annotation
worker
PII type
alert severity
sort
```

Sensitive values must not be placed in URLs.

### 5.6 Progressive Disclosure

Overview screens should summarize; detail screens should expose evidence.

Avoid showing every event, metric, field, and internal identifier on the first screen.

## 6. Information Architecture

The initial top-level navigation should include:

```text
Overview
Runs
Queues
Workers
Schedules
Alerts
PII Findings
Monitoring Health
Components
```

A user should not need to understand the backend bounded contexts to use the product.

## 7. Global Dashboard Context

The frontend should provide a consistent global context where applicable:

```text
time range
timezone
auto-refresh state
last refresh / data freshness
global search
active filter count
```

Recommended initial time ranges:

```text
15m
1h
6h
24h
7d
custom
```

The exact set is not normative.

All absolute timestamps should expose timezone context. Relative timestamps such as `3m ago` should provide the absolute timestamp on demand.

## 8. Overview Dashboard

The overview should answer:

```text
Is the distributed job system healthy right now?
What changed recently?
Where should I investigate first?
```

Minimum sections:

### 8.1 Run Health

```text
succeeded
failed
running
waiting / queued
retrying execution chains
lost / stuck annotations
failure rate
```

### 8.2 Queue Health

```text
healthy / degraded queue count
queue depth
oldest pending age
backlog trend
queues with no consumers
```

### 8.3 Worker Health

```text
online
degraded
offline
capacity utilization
workers with active lost-run candidates
```

### 8.4 Scheduling Health

```text
late starts
missed schedules
schedule drift
recent recurring-run failures
```

### 8.5 Alerts

```text
active alerts by severity
new alerts
acknowledged alerts if supported
highest-impact alert groups
```

### 8.6 PII

```text
runs with findings
findings by type
policy actions
scan failures / incomplete scans
```

### 8.7 Monitoring Health

```text
event-ingestion health
projection freshness
Redis inspection freshness
alert-evaluation freshness
known monitoring gaps
```

Monitoring health must not be hidden at the bottom when the dashboard's data cannot be trusted.

### 8.8 Project Components

```text
API
scheduler
Redis delivery
workers
monitoring ingestor/projector
PII scanner
alert evaluator
query API
real-time gateway
```

The overview should show healthy/degraded/offline component counts and allow drill-down to component evidence without becoming a generic infrastructure inventory.

### 8.9 Live Activity

A bounded sanitized activity feed may show meaningful changes such as run starts/completions, worker health changes, queue degradation, alert changes, and monitoring degradation. Heartbeat noise and raw logs should not be streamed directly.

## 9. Runs Experience

### 9.1 Run List

The list should support:

```text
search
filter
sort
cursor pagination
refresh
row-to-detail navigation
```

Minimum columns:

```text
run_id
retry index
job type
queue
execution state
monitoring health
attempt count
latest worker
created / started time
duration
PII status
active alerts
```

Long identifiers may be visually shortened but the full value must remain copyable.

### 9.2 Run Detail

Recommended sections:

The header should expose `execution_chain_id`, `parent_run_id`, retry index, and links to previous/next retry runs when available.


```text
Summary
Retry lineage / execution chain
Timeline
Attempts
Scheduling
Redis Delivery
Worker Activity
PII Findings
Alerts / Evidence
Sanitized Logs
Trace / Correlation
Monitoring Completeness
```

The first viewport should communicate:

```text
what is the execution state?
is monitoring reporting an abnormal condition?
how old is this state?
is the monitoring data fresh?
what is the latest important event?
```

### 9.3 Timeline

Timeline events should show at minimum:

```text
timestamp
event type
source / context
attempt
worker when applicable
short safe summary
late-arrival marker when applicable
```

The timeline should support grouping or filtering noisy event classes such as heartbeats without deleting them from the underlying evidence.

## 10. Queue Experience

Queue screens should show:

```text
current depth
pending
oldest age
consumer count
throughput
completion / acknowledgement rate
backlog trend
sampling freshness
active alerts
affected runs
```

A queue anomaly should provide direct navigation to runs contributing to that anomaly.

## 11. Worker Experience

Worker screens should show:

```text
status
last heartbeat
version
queues
capacity
active attempts
utilization
recent completions
recent failures
active alerts
infrastructure metadata when available
```

A worker marked offline should expose affected active attempts and suspected lost runs.

## 12. Schedule Experience

Schedule screens should show:

```text
definition
next expected occurrence
recent occurrences
associated runs
creation drift
start drift
missed / delayed occurrences
recent result distribution
```

## 13. Alerts Experience

The alerts view should support:

```text
severity
alert type
status
entity type
entity identifier
first seen
last seen
evidence summary
related runs / workers / queues / schedules
```

The frontend should prefer evidence-driven navigation over standalone alert cards with no path to root cause.

## 14. PII Findings Experience

PII findings should display only authorized, minimized information:

```text
type
source
field path
confidence
policy action
run
attempt
detector/version
policy name/version
rule ID
mask strategy when applicable
time
```

Raw PII must not be returned to the frontend merely so the frontend can hide it.

Permission-denied fields should be omitted or explicitly represented as unavailable according to the API contract.

The PII area should also expose the active policy name/version/checksum, revision history, detector inventory, safe rule summaries, policy validation failures, and scanner instances whose active version differs from the authoritative policy.

If policy editing is added, read-only investigation and policy mutation permissions must remain separate. Candidate policy JSON must be validated before an activation action is enabled.

## 15. Monitoring Health Experience

Monitoring health deserves its own view because monitoring is the product.

It should expose:

```text
event ingestion lag
projection lag
queue observation age
alert evaluation lag
PII scan lag / scanner health
active PII policy version
PII policy reload/compile health
scanner policy-version drift
known gaps
last successful observation per subsystem
```

A degraded monitoring subsystem should produce a visible warning on dependent views.

Example:

```text
Queue data may be stale.
Last successful Redis inspection: 2m 14s ago.
```

## 16. Filtering and Search

Minimum filters should align with RFC-008:

```text
run_id
job_type
queue
execution_state
monitoring_annotation
worker_id
schedule_id
time range
duration range
attempt count
failure category
PII type
PII scan status
alert type
alert severity
```

Filter behavior should be predictable:

- active filters are visible,
- filters can be cleared individually or together,
- invalid combinations produce actionable feedback,
- changing filters does not silently expand the requested time range,
- filter state should be encoded into the URL when safe.

## 17. Real-Time Update Semantics

RFC-010 defines the live observation contract. The dashboard should use a snapshot + stream model rather than treating an open connection as authoritative state.

Baseline browser behavior:

```text
load current Query snapshot + live watermark
        ↓
connect SSE after watermark
        ↓
apply focused safe summaries / invalidate aggregate queries
        ↓
on disconnect: reconnect and replay
        ↓
if cursor expired: RESYNC_REQUIRED → fetch fresh snapshot
```

The UI must expose live state such as:

```text
CONNECTING
LIVE
RECONNECTING
RESYNCING
POLLING_FALLBACK
STALE
PAUSED
```

Manual refresh remains available. Auto/live updates can be paused. Pausing live updates must make the loss of freshness obvious and should require reconciliation before returning to `LIVE`.

User-facing semantics are normative:

```text
failed live refresh does not erase previously loaded data
stale data is marked stale
replay gaps are never silently skipped
new updates do not unexpectedly destroy investigation position
complex filtered lists are invalidated/refetched rather than reimplemented client-side
focused active resources may receive compact summary updates
```

The dashboard must distinguish live transport health from monitoring/source freshness.

Example:

```text
Live: connected
Projection: 1s old
Redis sample: 41s old — STALE
```

Polling is the fallback when streaming is unavailable. Exact fallback intervals are configurable and must respect backend capacity.

## 18. Frontend Data States

Every major data surface must intentionally handle:

```text
INITIAL_LOADING
REFRESHING
EMPTY
READY
STALE
PARTIAL
ERROR
FORBIDDEN
```

`EMPTY` and `ERROR` must not look the same.

`PARTIAL` should be used when some dependent data is unavailable but the view remains useful.

Example:

```text
Run lifecycle is available.
Redis delivery evidence is temporarily unavailable.
```

## 19. Visualization Semantics

Visual status must not rely on color alone.

Status should use a combination of:

```text
label
icon / shape when useful
text
color
```

Charts should:

- expose units,
- expose time range,
- avoid misleading zero baselines where inappropriate,
- distinguish missing data from zero,
- make sampling gaps visible,
- support textual summaries for critical status.

## 20. Performance Requirements

The frontend should remain responsive for operational datasets by using backend-bounded queries.

Requirements:

- no unbounded run/event fetches,
- cursor/keyset pagination for large lists,
- bounded timeline retrieval with progressive loading where needed,
- cancellation of obsolete requests when filters change rapidly,
- avoid requesting high-cardinality aggregates the current screen does not use,
- preserve useful cached data during background refresh when safe.

Concrete latency targets should be defined after expected dataset size is established.

## 21. Security and Authorization

Authorization is enforced by backend services.

The frontend may use permissions to improve navigation and avoid offering impossible actions, but frontend checks are not security boundaries.

Examples:

```text
runs.read
workers.read
queues.read
schedules.read
alerts.read
logs.read
pii.findings.read
pii.masked.read
```

The frontend must not persist sensitive API responses to long-lived browser storage without explicit need and review.

PII, credentials, tokens, and sensitive error content must not be included in client analytics or frontend error-reporting payloads.

## 22. Accessibility and Device Scope

The initial product should be desktop-first but remain usable at narrower widths.

At minimum:

- keyboard navigation for primary workflows,
- visible focus state,
- semantic headings and controls,
- accessible labels for status indicators,
- no color-only health communication,
- tables remain inspectable without requiring hover-only interactions.

## 23. Frontend Observability

The frontend itself should emit enough sanitized telemetry to diagnose operator-facing failures.

Useful signals include:

```text
page load / route failure
API request failure by endpoint class
query latency
frontend exception count
stale-data banner frequency
permission-denied navigation
live-update disconnects
```

Do not use `run_id`, `attempt_id`, raw search text, PII, or unrestricted URL query strings as high-cardinality metrics labels.

## 24. Saved and Shareable Investigations

The initial requirement is shareable URL state.

A later release may support named saved views such as:

```text
Failed settlement jobs — 24h
Queues with backlog
PII BLOCK findings
Offline workers with active attempts
```

Saved views must store safe filter definitions, not raw PII values.

## 25. Compatibility with RFC-008

RFC-008 owns the query/read-model contract.

RFC-009 owns how those read models are presented and navigated.

Changes that require new backend facts, filters, aggregates, or consistency guarantees must be proposed against RFC-008 rather than implemented as frontend-only inference.

## 26. Acceptance Criteria

The dashboard frontend is acceptable for the initial release when an operator can:

- determine whether the system and monitoring pipeline appear healthy,
- drill from an unhealthy aggregate to affected entities and runs,
- search and filter historical runs,
- distinguish execution state from monitoring health,
- inspect a run timeline and attempts,
- inspect queue, worker, schedule, alert, and PII views,
- see when displayed data is stale, partial, or degraded,
- preserve investigation filters and time range during drill-down,
- share a safe deep link that reproduces an investigation view,
- inspect PII findings without receiving unauthorized raw values,
- understand empty, error, stale, partial, and forbidden states,
- use critical workflows without relying on color alone,
- observe active run, queue, worker, schedule, alert, PII-summary, monitoring-health, and component changes without manual refresh,
- understand whether live observation is connected, reconnecting, resynchronizing, in polling fallback, paused, or stale,
- recover from a live-stream interruption without silently losing continuity,
- distinguish a healthy live connection from stale source/projection data.

## 27. Open Questions

1. What freshness target should the overview dashboard guarantee under normal operation?
2. Which views should receive compact live summaries versus invalidation/refetch notifications under RFC-010?
3. Which charts are necessary for the first release versus later operational-intelligence phases?
4. Should named saved views be personal, shared, or both?
5. Should dashboard layout customization be supported, or should the product keep a fixed operational layout initially?
6. What browser support policy is required?
7. When operator commands are introduced, should they live in the same dashboard or a separately permissioned control surface?
8. Should the frontend provide export functionality, and which fields are safe to export?
