# RFC-004: Worker Runtime Context

- **Status:** Draft
- **Context:** Worker Runtime
- **Depends On:** RFC-001, RFC-003
- **Consumers:** Job Run Lifecycle, Monitoring

## 1. Summary

The Worker Runtime context defines the behavior of independent worker processes that consume Redis-delivered jobs and execute handlers.

Workers are not goroutines hidden inside the API server. They are independently observable runtime participants.

## 2. Goals

- unique worker identity,
- multiple workers consuming the same logical queue,
- configurable concurrency,
- worker heartbeats,
- execution heartbeats,
- clear execution protocol,
- graceful shutdown,
- observable handler outcomes.

## 3. Non-Goals

- owning alert severity,
- deciding whether the system is healthy,
- maintaining the monitoring read model,
- owning schedule definitions.

## 4. Worker Identity

Each worker requires a stable runtime identifier.

Recommended metadata:

```text
worker_id
instance_id
hostname
process_id
version
started_at
queues
concurrency
deployment_metadata
```

`worker_id` must be unique for concurrently active processes.

## 5. Worker Registration

On startup, a worker announces:

```text
identity
version
supported job types
queues
concurrency
start time
```

The runtime must not assume graceful deregistration always occurs.

Worker disappearance is detected via heartbeat expiry.

## 6. Worker Heartbeat

Conceptual payload:

```json
{
  "worker_id": "worker-7",
  "occurred_at": "...",
  "running_attempts": 4,
  "capacity": 8,
  "version": "v1.4.0"
}
```

Heartbeat frequency and health thresholds are separate concepts.

The worker only emits facts. Monitoring interprets:

```text
ONLINE
DEGRADED
OFFLINE
```

## 7. Execution Protocol

```text
Receive delivery
      │
      ▼
Validate envelope
      │
      ▼
Claim / establish attempt
      │
      ▼
Emit attempt.started
      │
      ▼
Execute handler
      │
 ┌────┴──────────────┐
 ▼                   ▼
success             failure
 │                   │
 ▼                   ▼
record success      record failure
 │                   │
 └────────┬──────────┘
          ▼
ack / retry handling
```

The exact ordering between durable state writes and Redis acknowledgement must be specified jointly with RFC-003.

## 8. Execution Heartbeat

Long-running jobs should support attempt-level heartbeats.

Conceptual event:

```text
attempt_id
worker_id
occurred_at
optional_progress
```

Execution heartbeat differs from worker heartbeat.

A worker can be alive while one handler is stuck.

## 9. Capacity

Workers should report:

```text
configured_concurrency
active_attempts
available_slots
```

Monitoring may derive utilization.

No worker should claim unbounded work without deliberate configuration.

## 10. Graceful Shutdown

On shutdown:

1. stop claiming new jobs,
2. allow running attempts a grace period,
3. emit shutdown intent if possible,
4. do not acknowledge unfinished work merely to empty the queue,
5. allow Redis reclaim/re-delivery if execution cannot complete.

The system must also work when none of these steps can occur because of hard crash.

## 11. Handler Contract

Each job handler should produce a structured outcome:

```text
SUCCESS
RETRYABLE_FAILURE
NON_RETRYABLE_FAILURE
```

with:

```text
error_category?
error_code?
sanitized_message?
result_reference?
```

The handler must not be required to understand alert rules or monitoring UI concerns.

## 12. Duplicate Execution

Workers must assume the same logical work can be delivered more than once.

Possible protection layers:

- idempotency key,
- durable attempt ownership,
- business-level deduplication,
- compare-and-set transitions.

The final strategy must be documented by implementation.

## 13. Logging

Logs should be structured and correlated with:

```text
run_id
attempt_id
worker_id
job_type
trace_id?
```

Raw PII must not be intentionally logged.

Where log forwarding is integrated with PII scanning, scanning/redaction should occur before monitoring persistence.

## 14. Failure Semantics

Examples:

### Handler returns error

```text
attempt → FAILED
```

### Process crashes

Do not fabricate a handler error.

Observed facts may instead be:

```text
last execution heartbeat = old
worker heartbeat = missing
Redis pending entry = still present
```

Monitoring can derive `RUN_LOST`.

### Worker is alive but handler heartbeat stops

Monitoring may derive `RUN_STUCK`.

## 15. Open Questions

1. Should worker identity survive process restart?
2. Where is concurrency enforced?
3. Are execution heartbeats mandatory above a duration threshold?
4. How are cancellation signals delivered?
5. How does a worker prove ownership of an attempt before changing terminal state?
6. Should workers send monitoring events directly or only through the Job Run context?
