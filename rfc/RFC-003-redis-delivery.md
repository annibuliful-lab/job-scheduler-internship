# RFC-003: Redis Delivery Context

- **Status:** Draft
- **Context:** Redis Delivery
- **Depends On:** RFC-000, RFC-001
- **Consumers:** Worker Runtime, Monitoring

## 1. Summary

This context defines how executable work is transported and coordinated through Redis.

Redis is a **runtime delivery substrate**. It is not the durable source of historical monitoring truth.

The architecture should assume at-least-once delivery and possible worker/process failure.

## 2. Goals

- distribute jobs across independent workers,
- support multiple workers per queue,
- expose queue and pending-delivery facts,
- support acknowledgement after processing,
- enable recovery from abandoned deliveries,
- preserve correlation identifiers.

## 3. Non-Goals

- maintaining complete job history,
- serving as the only persistence layer,
- determining business success,
- deriving operational alerts,
- storing unredacted PII for investigation.

## 4. Redis Primitive

Redis Streams with consumer groups are a natural fit because they expose concepts such as:

```text
stream
consumer group
consumer
message id
pending entries
delivery count
acknowledgement
```

However, the RFC defines required semantics rather than forcing a specific implementation detail.

## 5. Delivery Envelope

A delivery envelope should contain only what is required for execution and correlation.

Conceptual form:

```json
{
  "message_id": "redis-or-logical-id",
  "run_id": "run-123",
  "attempt_id": "attempt-1",
  "job_type": "generate_report",
  "payload_ref": "...",
  "payload": {},
  "created_at": "...",
  "correlation_id": "...",
  "trace_id": "..."
}
```

Whether payload is embedded or referenced must be decided with PII and payload-size considerations.

## 6. Delivery Lifecycle

```text
PUBLISHED
    │
    ▼
AVAILABLE
    │
    ▼
CLAIMED
    │
    ├── worker alive + processing
    │
    ▼
ACKNOWLEDGED
```

Abnormal branch:

```text
CLAIMED
   │
   ▼
worker disappears
   │
   ▼
pending / reclaim candidate
   │
   ▼
REDELIVERED
```

## 7. Critical Distinction

```text
message claimed
      ≠
attempt started
      ≠
attempt succeeded
      ≠
message acknowledged
```

Monitoring must preserve these boundaries.

## 8. Acknowledgement Semantics

The implementation must define when acknowledgement occurs.

Typical choices:

- after handler success,
- after terminal execution state is durably recorded,
- after retry scheduling is durably recorded.

Acknowledging before the relevant durable state is recorded can create lost work.

## 9. At-Least-Once Delivery

The system must assume:

- duplicate deliveries can occur,
- a worker can crash after completing work but before acknowledgement,
- a message can be reclaimed,
- multiple workers may temporarily appear related to the same run.

Therefore job handlers or the execution layer need an idempotency strategy.

## 10. Pending Delivery Monitoring

Monitoring requires access to facts such as:

```text
queue/stream name
message id
consumer
idle duration
delivery count
pending count
oldest pending age
consumer count
```

These facts may be collected periodically instead of persisted per-message forever.

## 11. Queue Health Signals

The delivery context should expose raw signals, not final alert judgments.

Examples:

```text
queue_depth
oldest_available_age
pending_count
oldest_pending_age
active_consumer_count
delivery_rate
ack_rate
```

Monitoring decides whether these imply:

```text
QUEUE_BACKLOG
QUEUE_AGE_HIGH
QUEUE_NO_CONSUMER
```

## 12. Reclaim Semantics

If a worker disappears while owning a pending delivery, the system may reclaim it after a configurable idle threshold.

Reclaiming must not automatically mean the previous attempt failed.

Monitoring may observe:

```text
attempt state: RUNNING
worker heartbeat: missing
delivery reclaimed
```

and flag a consistency/anomaly condition.

## 13. Redis Outage

Redis unavailability must be distinguishable from application job failure.

Potential states:

```text
dispatch unavailable
queue inspection unavailable
acknowledgement delayed
worker cannot fetch work
```

Monitoring should report degraded visibility if Redis cannot be inspected.

## 14. PII Considerations

Redis may contain job payloads.

Requirements:

- payload retention should be minimized,
- Redis access must be restricted,
- monitoring should not duplicate raw payloads unnecessarily,
- PII policy should influence whether payload is embedded, encrypted, referenced or redacted,
- debug tooling must not casually expose entire messages.

## 15. Open Questions

1. Redis Streams vs another Redis pattern?
2. One stream per logical queue or shared stream with routing fields?
3. How are priority jobs represented?
4. Who performs pending-entry reclaim?
5. What acknowledgement boundary is required?
6. Should payloads live directly in Redis or through durable payload references?
7. How should poison messages be represented?
