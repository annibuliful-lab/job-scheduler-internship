# RFC-010: Real-Time Monitoring and Live Observation

- **Status:** Draft
- **Area:** Real-Time Observation / Monitoring Delivery
- **Depends On:** RFC-000, RFC-005, RFC-006, RFC-007, RFC-008, RFC-009
- **Primary Product Focus:** Yes
- **Primary Consumers:** Dashboard Frontend, operators, automated operational clients

## 1. Summary

This RFC defines real-time monitoring for the distributed job scheduling monitoring platform.

Real-time monitoring means that operational changes across the project can become visible to an authorized operator shortly after they are observed, without requiring manual refresh and without making the live transport a new source of truth.

The capability covers the project as a whole:

```text
job runs / retry chains
worker attempts
Redis queues and delivery health
workers and capacity
schedules and drift
alerts
PII findings and scan health
monitoring ingestion / projection health
platform component health
```

The durable Monitoring and Query contexts remain authoritative. Real-time observation is a low-latency delivery layer over those facts and projections.

## 2. Definition of Real Time

This product requires **operational near-real-time**, not hard real-time processing.

Under normal operating conditions, important monitoring changes should become visible to a connected dashboard within a small bounded delay. An initial product target is:

```text
observed operational change
        ↓
monitoring ingestion
        ↓
projection / derived health update
        ↓
live publication
        ↓
operator-visible update

p95 target: <= 5 seconds
```

The exact SLO is configurable and should be validated against deployment size and load.

The product does not promise:

- deterministic sub-second delivery,
- global total ordering across all resources,
- exactly-once live delivery,
- uninterrupted browser connectivity.

## 3. Goals

- make important changes visible without manual refresh,
- observe the health of the entire project from one operational surface,
- stream run, queue, worker, schedule, alert, PII, and monitoring-health changes,
- expose component-instance health for project services/processes,
- support reconnect and bounded replay after temporary disconnects,
- preserve correctness when events are duplicated, delayed, or delivered out of order,
- provide snapshot reconciliation when the live cursor is no longer recoverable,
- expose live-channel health and lag to the operator,
- keep live updates authorization-aware and PII-safe,
- avoid overwhelming clients with high-frequency low-value signals,
- preserve the durable investigation history independently of the live channel.

## 4. Non-Goals

This context does not:

- replace durable monitoring storage,
- become the authoritative job-state store,
- replace Prometheus, OpenTelemetry, or a general APM platform,
- stream unrestricted raw logs to every browser,
- guarantee exactly-once delivery,
- guarantee a single global order for all distributed events,
- execute retry/cancel/redrive commands,
- turn Redis Pub/Sub or an SSE connection into historical truth,
- expose raw PII through live notifications.

## 5. Core Principle: Snapshot Is Truth, Stream Is Change Notification

The dashboard must not reconstruct authoritative system state only from an open stream.

The expected model is:

```text
Query Snapshot
     │
     │ authoritative read model + freshness + observation watermark
     ▼
Dashboard
     │
     │ live changes after watermark
     ▼
Live Observation Stream
```

When confidence in the stream is lost:

```text
stream gap / cursor expired / ordering ambiguity
                 │
                 ▼
           RESYNC_REQUIRED
                 │
                 ▼
        fetch fresh snapshot
                 │
                 ▼
          resume live stream
```

The product prefers temporary staleness with an explicit warning over silently presenting an incorrect reconstructed state.

## 6. Context Boundary

```text
Domain Events / Runtime Signals
             │
             ▼
      Monitoring Context
      durable events + projections
             │
             ├───────────────┐
             │               │
             ▼               ▼
      Query Snapshot      Live Change Publisher
             │               │
             │               ▼
             │        Real-Time Gateway
             │               │
             └───────┬───────┘
                     ▼
              Dashboard Frontend
```

### Monitoring & Observability owns

- durable facts,
- projections,
- anomaly interpretation,
- freshness/completeness,
- component-health projections.

### Query & Investigation owns

- current safe snapshots,
- filters/search,
- snapshot watermarks,
- historical evidence.

### Real-Time Monitoring owns

- subscription semantics,
- low-latency change fanout,
- replay cursor/resume semantics,
- backpressure/coalescing,
- stream connection health,
- resynchronization signaling.

### Dashboard Frontend owns

- live connection lifecycle,
- presentation of incoming change,
- preserving operator context,
- reconciliation/refetch behavior,
- visible live/stale/disconnected state.

## 7. Observable Resource Types

Minimum live resource classes:

```text
PLATFORM_SUMMARY
RUN
EXECUTION_CHAIN
ATTEMPT
QUEUE
WORKER
SCHEDULE
ALERT
PII_FINDING_SUMMARY
PII_POLICY_ACTIVATED
PII_POLICY_RELOAD_FAILED
PII_POLICY_DRIFT_DETECTED
MONITORING_HEALTH
COMPONENT
```

Not every resource requires the same update frequency.

High-frequency heartbeats should normally update a worker/component projection and publish meaningful summarized changes rather than forwarding every heartbeat to browsers.

## 8. Whole-Project Component Monitoring

The platform should observe its own runtime components in addition to business job execution.

Initial component types:

```text
API
SCHEDULER
REDIS_DELIVERY
WORKER
MONITORING_INGESTOR
MONITORING_PROJECTOR
PII_SCANNER
ALERT_EVALUATOR
QUERY_API
REALTIME_GATEWAY
DASHBOARD_BACKEND   # only if a server-side frontend runtime exists
```

A component instance should expose safe operational metadata such as:

```text
component_instance_id
component_type
instance_name
version
build_revision
started_at
last_heartbeat_at
status
status_reason
observed_at
```

Optional summarized dependency health:

```text
postgres: HEALTHY
redis: DEGRADED
monitoring_ingestion: HEALTHY
```

The component model is scoped to this project. It is not intended to become a generic infrastructure inventory.

## 9. Component Health Semantics

Suggested derived health values:

```text
HEALTHY
DEGRADED
UNHEALTHY
OFFLINE
UNKNOWN
```

Health is an interpretation and should carry evidence.

Example:

```text
component_type: MONITORING_PROJECTOR
status: DEGRADED
reason: PROJECTION_LAG_HIGH
observed_lag_ms: 12750
threshold_ms: 5000
```

A missing heartbeat must account for startup, shutdown, drain, and expected maintenance when those signals are available.

## 10. Live Update Envelope

Illustrative envelope:

```json
{
  "live_event_id": "01K...",
  "schema_version": 1,
  "published_at": "2026-08-17T04:45:02Z",
  "observed_at": "2026-08-17T04:45:01Z",
  "resource_type": "RUN",
  "resource_id": "run-123",
  "resource_revision": 42,
  "change_type": "UPSERT",
  "change_seq": 928318,
  "payload_mode": "SUMMARY",
  "payload": {},
  "freshness": {
    "status": "FRESH",
    "projection_lag_ms": 900
  }
}
```

The exact JSON schema is implementation-specific, but the contract needs enough information for:

- deduplication,
- per-resource ordering,
- reconnect/resume,
- stale-update rejection,
- safe reconciliation.

## 11. Change Types

Suggested change classes:

```text
UPSERT
DELETE
INVALIDATE
HEALTH_CHANGED
RESYNC_REQUIRED
HEARTBEAT
```

`INVALIDATE` means:

```text
The resource/query may have changed; fetch the current snapshot.
```

This is often preferable for aggregates and filtered lists where sending full deltas would force the browser to duplicate backend query logic.

## 12. Payload Strategy

The system may use three payload modes:

### SUMMARY

Small safe resource snapshot sufficient to update a focused detail view.

### INVALIDATION

No business payload; tells the client which query/resource should be refreshed.

### HEALTH_ONLY

Contains only health/freshness state required for a global status surface.

The live layer should not indiscriminately stream complete run payloads, raw errors, logs, or PII-bearing data.

## 13. Ordering

The system does not require a total order across all resources.

It does require deterministic conflict handling for one resource.

Each live resource update should carry a monotonic or comparable `resource_revision`.

Client rule:

```text
incoming revision <= rendered revision
        ↓
ignore as duplicate/stale
```

Cross-resource causal relationships should be investigated using durable monitoring timelines rather than inferred solely from live arrival order.

## 14. Delivery Semantics

Live delivery is at-least-once.

Clients must tolerate:

```text
duplicate update
late update
out-of-order update
connection loss
reconnect
cursor expiration
server restart
```

Exactly-once live delivery is not required.

## 15. Subscription Scopes

A client should subscribe only to the operational scope it needs.

Examples:

```text
platform.summary
runs
run:{run_id}
execution-chain:{execution_chain_id}
queues
queue:{queue_name}
workers
worker:{worker_id}
schedules
alerts
pii.summary
monitoring.health
components
```

The implementation may use a smaller internal topic set and filter server-side.

Authorization applies to the effective subscription, not only to the initial page route.

## 16. Initial Snapshot and Watermark

A safe live view should establish a snapshot before applying incremental updates.

Conceptually:

```text
GET /overview
→ data
→ freshness
→ live_cursor / observation_watermark

CONNECT /live?after=<cursor>
```

The snapshot/cursor contract must avoid a blind window where a change can happen after snapshot generation but before the live connection begins and then be missed.

Possible implementations are defined in TECH-010.

## 17. Reconnect and Replay

Clients should reconnect automatically using bounded backoff.

If the live transport supports a resume cursor:

```text
disconnect at cursor 100
reconnect after cursor 100
receive 101..latest
```

Replay retention is intentionally bounded.

If the requested cursor is too old:

```text
RESYNC_REQUIRED
```

The client then fetches a fresh snapshot.

## 18. Live Connection State

The dashboard must expose a connection state such as:

```text
CONNECTING
LIVE
DEGRADED
RECONNECTING
STALE
RESYNCING
PAUSED
```

The connection badge is not decorative. It communicates whether the operator is currently seeing updates.

Example:

```text
LIVE — last update 1s ago
```

versus:

```text
RECONNECTING — last confirmed update 47s ago
```

## 19. Backpressure and Event Storms

The live path must protect both the gateway and browser during bursts.

Allowed strategies include:

- coalesce multiple updates to the same resource,
- send `INVALIDATE` rather than thousands of row patches,
- summarize heartbeat changes,
- cap per-client buffered updates,
- disconnect and require resync when a client falls too far behind,
- apply subscription/topic rate limits,
- prioritize health/terminal-state changes over low-value activity updates.

The system must not silently drop important changes while still claiming the connection is fully synchronized.

## 20. Event Priority

Suggested priority classes:

### Critical operational change

```text
RUN_FAILED
RUN_LOST
WORKER_OFFLINE
QUEUE_NO_CONSUMER
ALERT_OPENED critical/high
MONITORING_DEGRADED
PII_POLICY_VIOLATED
```

### Normal state change

```text
RUN_STARTED
RUN_SUCCEEDED
WORKER_CAPACITY_CHANGED
QUEUE_DEPTH_CHANGED
SCHEDULE_OCCURRENCE_CREATED
```

### High-frequency informational signal

```text
heartbeat
progress sample
queue sample
```

High-frequency signals should normally be projected/aggregated before browser fanout.

## 21. Real-Time Activity Feed

The dashboard may expose a sanitized activity feed showing meaningful project events.

Examples:

```text
11:45:01 report run-123 started on worker-4
11:45:04 queue reports became DEGRADED
11:45:07 worker-2 became OFFLINE
11:45:09 alert QUEUE_BACKLOG opened
11:45:12 run-125 succeeded
```

The activity feed is an operational convenience, not a replacement for the canonical run timeline.

It must not include unrestricted logs or raw PII.

## 22. Live Platform Overview

The whole-project live overview should be able to display:

```text
Active runs
Queued/waiting runs
Retrying chains
Failed/lost/stuck changes
Queue depth / oldest age / consumers
Workers online/degraded/offline
Schedules due/late/missed
Open alerts
PII finding/scan/policy health
Monitoring ingestion lag
Projection lag
Live-stream lag
Project component health
```

An operator should be able to drill from a changing aggregate into the affected resource without losing investigation context.

## 23. Real-Time and Historical Consistency

A live update can arrive before a related historical query is refreshed.

The UI must not invent missing evidence.

Example:

```text
live signal: RUN_LOST
historical timeline query: still at previous projection revision
```

The UI may show:

```text
RUN_LOST — live update received
Evidence is refreshing…
```

until the durable read side catches up.

## 24. Monitoring the Real-Time Pipeline

The live system must monitor itself.

Useful metrics/signals:

```text
realtime_connections
realtime_connect_total
realtime_disconnect_total
realtime_publish_total
realtime_delivery_lag_ms
realtime_replay_total
realtime_resync_required_total
realtime_client_buffer_depth
realtime_dropped_or_coalesced_total
realtime_auth_rejection_total
realtime_gateway_heartbeat_age
```

High-cardinality resource IDs must not become metric labels.

## 25. Security and PII

The live transport must enforce the same or stricter security boundary as ordinary Query APIs.

Requirements:

- authenticate the connection,
- authorize subscription scopes,
- re-evaluate authorization on reconnect and token/session changes,
- serialize only fields the principal can read,
- never depend on client-side hiding for sensitive fields,
- do not put raw PII in stream topic names, URLs, metrics, or connection logs,
- sanitize live error messages,
- avoid raw job payload/result/log streaming by default.

## 26. Failure Semantics

### Query API healthy, live gateway unhealthy

Historical/current snapshots remain usable. Dashboard becomes `STALE` or polling-fallback mode.

### Live gateway healthy, monitoring projection delayed

The dashboard must expose monitoring degradation; a healthy socket does not mean healthy data.

### Redis live transport lost

If Redis is used for transient live fanout, durable history remains available from PostgreSQL. Clients resynchronize from Query snapshots when live transport returns.

### Browser disconnected

No job execution behavior changes. Live monitoring is observational.

### One project component disappears

Component health may become `OFFLINE`; this does not automatically imply every job handled by that component failed.

## 27. Fallback Behavior

A dashboard should remain operational when streaming is unavailable.

Fallback order may be:

```text
live stream
    ↓ unavailable
bounded polling
    ↓ unavailable
last successful snapshot + STALE warning
```

Fallback must be visible to the operator.

## 28. Acceptance Criteria

Real-time monitoring is acceptable when an authorized operator can:

- open the dashboard and see whether live observation is connected,
- observe active job-run state changes without manual refresh,
- observe worker online/degraded/offline changes,
- observe queue-health changes and backlog movement,
- observe schedule-health changes,
- observe alert open/update/resolve changes,
- observe PII finding/scan-health summaries safely,
- observe monitoring pipeline health and projection lag,
- observe health of the project runtime components,
- reconnect after a temporary network interruption without silently losing changes,
- receive an explicit resync signal when a replay gap cannot be recovered,
- continue investigating using snapshots when the live channel is unavailable,
- see when data is stale even if the browser connection itself is healthy,
- avoid exposure of unauthorized PII through the live path.

## 29. Open Questions

1. What production p95/p99 operator-visible latency targets should be committed after load testing?
2. How long should resumable live updates be retained?
3. Which overview aggregates should use compact payload updates versus invalidation/refetch?
4. Should multi-project/tenant scoping be introduced before externalizing the platform?
5. Which component dependency checks are useful without turning this product into generic infrastructure monitoring?
6. Should activity-feed retention exist separately, or should it always be reconstructed from monitoring events?
7. At what scale should the real-time gateway be separated into an independently deployable service?
