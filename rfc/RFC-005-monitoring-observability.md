# RFC-005: Monitoring and Observability Context

- **Status:** Draft
- **Context:** Monitoring & Observability
- **Depends On:** RFC-000 through RFC-004
- **Primary Product Focus:** Yes
- **Consumers:** Alerting, Query & Investigation, PII Detection integrations

## 1. Summary

This context is the core of the product.

It reconstructs a reliable operational view of distributed jobs from signals produced by:

- Job Run Lifecycle,
- Scheduling,
- Redis Delivery,
- Worker Runtime,
- PII Detection.

It provides durable history, timelines, health projections and anomaly evidence.

## 2. Goals

- reconstruct a complete run timeline,
- retain monitoring history independently of Redis,
- expose queue and worker health,
- distinguish facts from derived health interpretations,
- identify stuck/lost/delayed execution,
- identify missing or contradictory signals,
- provide trustworthy investigation data.

## 3. Non-Goals

- executing jobs,
- owning Redis acknowledgement,
- deciding PII classification rules,
- sending notification messages,
- becoming a general-purpose log platform.

## 4. Core Design

Monitoring receives immutable observations and builds projections.

```text
Domain Events / Runtime Signals
             │
             ▼
      Monitoring Ingestion
             │
             ▼
      Durable Event Store
             │
             ├──────────────┐
             ▼              ▼
      Current State      Time-series /
      Projections        Aggregate Data
             │
             ▼
      Investigation API
```

## 5. Monitoring Event Envelope

Minimum fields:

```text
event_id
event_type
schema_version
occurred_at
ingested_at
producer
```

Correlation fields where applicable:

```text
execution_chain_id
run_id
parent_run_id
attempt_id
worker_id
schedule_id
schedule_occurrence_id
queue_name
trace_id
correlation_id
```

## 6. Event Immutability

Historical facts should not be overwritten.

Example:

```text
10:00 attempt.started
10:05 worker heartbeat missing
10:06 delivery reclaimed
10:07 attempt.started by worker-b
```

The system should not rewrite the first start event merely because later evidence changed interpretation.

## 7. Projections

### Run Projection

Contains:
- current execution state,
- latest attempt,
- attempt count,
- timestamps,
- active monitoring annotations,
- PII summary,
- alert summary.

### Worker Projection

Contains:
- last heartbeat,
- active attempts,
- capacity,
- derived health.

### Queue Projection

Contains:
- queue depth,
- pending count,
- oldest message age,
- consumer count,
- throughput estimates,
- derived health.

### Schedule Projection

Contains:
- expected occurrences,
- associated run IDs,
- creation/start drift,
- missed occurrence candidates.

## 8. Fact vs Interpretation

Example facts:

```text
attempt.started_at = 10:00
last_attempt_heartbeat = 10:05
worker_last_heartbeat = 10:05
current_time = 10:20
```

Possible interpretation:

```text
RUN_LOST
```

The interpretation must carry evidence and rule version.

This enables later explanation:

```text
Why was this marked lost?
```

## 9. Monitoring Annotations

Suggested annotations:

```text
RUN_STUCK
RUN_LOST
RUN_SLA_VIOLATED
RUN_DUPLICATE_SUSPECTED
SCHEDULE_DELAYED
SCHEDULE_MISSED
QUEUE_BACKLOG
QUEUE_AGE_HIGH
QUEUE_NO_CONSUMER
WORKER_DEGRADED
WORKER_OFFLINE
MONITORING_GAP
```

These are separate from authoritative execution states.

## 10. Stuck Detection

A candidate stuck attempt may be identified when:

```text
now - last_execution_heartbeat > threshold
```

while worker heartbeat remains healthy.

Evidence should include:

```text
attempt_id
last_execution_heartbeat
worker_last_heartbeat
threshold
rule_id
```

## 11. Lost Detection

A candidate lost attempt may use combined evidence:

```text
attempt state = RUNNING
execution heartbeat missing
worker heartbeat missing
no terminal event observed
```

Redis pending/reclaim facts may strengthen the conclusion.

A lost classification should avoid pretending that a business failure occurred.

## 12. Duplicate Execution Detection

Possible signal:

```text
same run
+ multiple active attempts
+ overlapping execution windows
```

This should be a monitoring warning unless the execution model explicitly allows parallel attempts.

## 13. Schedule Monitoring

For each occurrence:

```text
expected_at
run_created_at?
run_queued_at?
run_started_at?
```

Derived metrics:

```text
creation_drift
queue_drift
start_drift
```

`SCHEDULE_MISSED` requires configurable grace windows.

## 14. Queue Monitoring

Raw queue samples should be timestamped.

Example:

```text
sample_at
queue_name
depth
pending
oldest_age
consumers
delivery_rate
ack_rate
```

Backlog detection should ideally consider trend, not only a fixed threshold.

## 15. Monitoring Gaps

The product must admit when observation is incomplete.

Examples:

- event ingestion unavailable,
- Redis inspection unavailable,
- worker heartbeat channel unavailable,
- projection consumer behind.

Expose states such as:

```text
COMPLETE
DEGRADED
PARTIAL
UNKNOWN
```

where useful.

## 16. Ordering

Distributed event timestamps can disagree.

The system should retain both:

```text
occurred_at
ingested_at
```

and, where possible:

```text
producer_sequence
stream_position
```

Projection logic must not rely solely on wall-clock ordering.

## 17. Persistence

Redis is not sufficient for durable monitoring history.

A durable database should retain:

```text
monitoring_events
run_projection
attempt_projection
worker_projection
queue_samples
schedule_projection
anomaly_evidence
```

Exact table names are implementation-specific.

## 18. Retention

Different data classes may require different retention:

- immutable lifecycle events,
- queue samples,
- logs,
- PII findings,
- alert history.

Retention must be configurable and documented.

## 19. Metrics

Examples:

```text
runs_created_total
runs_completed_total
runs_failed_total
attempts_total
queue_latency_seconds
execution_duration_seconds
schedule_start_drift_seconds
workers_online
queue_depth
queue_oldest_age_seconds
monitoring_ingestion_lag_seconds
monitoring_event_failures_total
```

Do not use `run_id` or `attempt_id` as general metric labels.

## 20. Open Questions

1. Event store vs ordinary relational event table?
2. How should projection rebuild work?
3. What is the maximum acceptable monitoring ingestion delay?
4. Which anomalies are continuously evaluated versus evaluated on event arrival?
5. How much Redis state should be sampled historically?
6. What evidence is retained when a derived annotation resolves?
7. How should late-arriving events revise current projections?
