# RFC-008: Query and Investigation Context

- **Status:** Draft
- **Context:** Query & Investigation
- **Depends On:** RFC-005, RFC-006, RFC-007
- **Primary Consumers:** Operators, developers, support engineers

## 1. Summary

This context defines the read-side contracts and investigation projections required to investigate distributed job behavior safely.

It is intentionally optimized for operational questions rather than transactional writes. Presentation, navigation, visualization, and browser behavior are owned by RFC-009 Dashboard Frontend.

## 2. Core Question

The query API and read models should provide enough evidence for an investigation client to answer:

```text
What happened?
Where did it happen?
When did it happen?
Which worker was involved?
What happened before and after it?
Is the problem execution, scheduling, Redis, worker health, PII, or monitoring itself?
```

## 3. Goals

- fast run search,
- complete chronological timeline data,
- queue investigation read models,
- worker investigation read models,
- schedule investigation read models,
- PII-safe findings,
- alert/evidence queries,
- explicit execution-state and monitoring-health dimensions,
- freshness and partial-data metadata required by RFC-009.

## 4. Non-Goals

- owning transaction state,
- mutating Redis directly,
- becoming a general SQL explorer,
- exposing unrestricted raw payloads.

## 5. Primary Query Surfaces

### 5.1 Overview Aggregate

Should include:

```text
runs by state
failure rate
retry count
queue health
worker health
schedule delay
PII findings
active alerts
monitoring ingestion health
```

### 5.2 Run List

Suggested fields:

```text
run_id
execution_chain_id
parent_run_id
retry_index
job_type
queue
execution_state
monitoring_health
attempt_count
latest_worker
created_at
started_at
duration
PII_status
active_alert_count
```

### 5.3 Run Detail

Sections:

```text
Summary
Retry lineage / execution chain
Timeline
Attempts
Scheduling
Redis delivery
Worker activity
PII findings
Alerts
Sanitized logs
Correlation / trace information
```

### 5.4 Queue Detail

```text
depth
pending
oldest age
consumer count
throughput
ack rate
recent backlog history
active alerts
affected runs
```

### 5.5 Worker Detail

```text
status
last heartbeat
version
queues
capacity
active attempts
recent completions
recent failures
active alerts
```

### 5.6 Schedule Detail

```text
schedule definition
next expected occurrence
recent occurrences
associated runs
creation drift
start drift
missed/delayed occurrences
```

### 5.7 PII Findings

Display:

```text
type
source
field path
confidence
policy action
run
attempt
detector
time
```

Raw values should not be shown by default.

## 6. Timeline

The timeline should merge facts from different contexts.

Example:

```text
10:00:00 schedule.occurrence_due
10:00:00 run-101 run.created          chain-55 retry=0
10:00:00 run-101 run.queued
10:00:01 run-101 delivery.claimed
10:00:01 run-101 attempt.started
10:00:02 run-101 pii.detected
10:00:11 run-101 attempt.heartbeat
10:00:45 run-101 attempt.failed
10:00:45 run-101 run.failed
10:00:45 run-102 run.retry_scheduled  parent=run-101 retry=1
10:05:45 run-102 run.queued
10:05:46 run-102 delivery.claimed
10:05:46 run-102 attempt.started
10:06:13 run-102 attempt.succeeded
10:06:13 run-102 run.succeeded
```

Late-arriving events should remain visible and should not be silently discarded.

## 7. Search Filters

Minimum filters:

```text
run_id
execution_chain_id
parent_run_id
job_type
queue
execution_state
monitoring_annotation
worker_id
schedule_id
created time
started time
duration range
attempt count
failure category
PII type
PII scan status
alert type
alert severity
```

## 8. Query Examples

The system should support investigation equivalent to:

```text
failed report jobs in the last 24 hours
running jobs older than 10 minutes
execution chains with more than 3 retry runs
runs with multiple worker/delivery attempts
runs executed by worker-7
runs affected by worker-offline alerts
runs with national-ID findings
schedules delayed more than 5 minutes
queues with increasing backlog
```

## 9. Read Model Strategy

The query layer should read projections optimized for investigation rather than performing expensive joins over raw event history for every screen.

Raw events remain available for detailed timeline reconstruction and audit/debug use.

## 10. Consistency Expectations

Read models may be eventually consistent.

Query responses should expose monitoring freshness so consuming clients can present it accurately:

```text
last updated 3s ago
projection lag 1.2s
```

For critical operator actions, stale data must not be presented as guaranteed current truth.

For live-capable views, responses should also expose an observation watermark/cursor that can be used by RFC-010 to subscribe to changes occurring after the returned snapshot without a silent blind gap.

Example shape:

```json
{
  "data": {},
  "freshness": {},
  "live": {
    "watermark": 928318
  }
}
```

## 11. PII-Safe Access

Authorization should separate:

```text
job.read
job.logs.read
worker.read
queue.read
alerts.read
pii.findings.read
pii.policy.read
pii.masked_value.read
```

The query layer should apply redaction before data leaves the backend.

Client-only hiding is insufficient.

## 12. Pagination and Scale

Run/event search must support bounded pagination.

Avoid:
- loading all event history at once,
- unbounded wildcard searches,
- high-cardinality aggregation without limits.

## 13. API Shape

Illustrative read endpoints:

```text
GET /runs
GET /runs/{run_id}
GET /runs/{run_id}/timeline
GET /runs/{run_id}/attempts
GET /execution-chains/{execution_chain_id}
GET /execution-chains/{execution_chain_id}/timeline

GET /queues
GET /queues/{queue_name}

GET /workers
GET /workers/{worker_id}

GET /schedules
GET /schedules/{schedule_id}

GET /pii/findings
GET /alerts
GET /monitoring/health
GET /components
GET /components/{component_instance_id}
```

Exact paths are not normative.

## 14. Investigation Safety

The platform must avoid turning debugging into an uncontrolled PII access path.

Default query representations should prefer:

```text
payload size
payload schema/field paths
redacted preview
PII finding metadata
```

rather than entire raw payloads.

## 15. Open Questions

1. How much raw event history is exposed through the UI?
2. Should logs be stored internally or linked to an external log platform?
3. What search backend is appropriate at expected scale?
4. Which aggregate read models are required for the first release?
5. Which operator commands, if any, should be added through a separately audited command boundary rather than this read context?
6. What read-model freshness is acceptable?
7. What exact snapshot/watermark mechanism should guarantee gap-free handoff to RFC-010 live subscriptions?
