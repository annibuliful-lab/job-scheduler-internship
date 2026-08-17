# RFC-000: Domain Boundaries and Integration Model

- **Status:** Draft
- **Context:** Platform-wide
- **Decision Type:** Domain architecture
- **Related RFCs:** RFC-001 through RFC-010

## 1. Summary

This RFC defines the bounded contexts for the distributed job scheduling monitoring platform and the rules used to integrate them.

The product is monitoring-first. The system needs enough scheduling, Redis delivery and worker execution behavior to create a realistic distributed execution environment, but those capabilities must not dominate the product architecture.

## 2. Motivation

Without explicit domain boundaries, the implementation is likely to collapse into a single "job" model containing:

- schedule configuration,
- Redis message details,
- worker execution state,
- retry policy,
- monitoring state,
- alert state,
- PII findings,
- UI-specific fields.

That creates ambiguous ownership and makes failures difficult to reason about.

This RFC separates the concepts according to why they change and who owns the truth.

## 3. Bounded Contexts

### 3.1 Job Run Lifecycle

Owns:
- logical job runs,
- execution attempts,
- execution state,
- retry run lineage (`execution_chain_id`, `parent_run_id`),
- run-attempt relationship,
- run identity,
- attempt identity.

Does not own:
- Redis consumer-group internals,
- dashboards,
- PII classification,
- alert rules.

### 3.2 Scheduling

Owns:
- schedule definitions,
- one-time schedules,
- recurring schedules,
- next due time,
- schedule occurrence identity,
- creation of due logical runs.

Does not own:
- execution completion,
- worker assignment,
- Redis acknowledgement.

### 3.3 Redis Delivery

Owns:
- runtime dispatch,
- queue/stream naming,
- delivery envelope,
- claim/acknowledgement mechanics,
- pending/re-delivery mechanics,
- Redis-specific identifiers.

Does not own:
- business success/failure,
- historical monitoring retention.

### 3.4 Worker Runtime

Owns:
- worker identity,
- worker runtime metadata,
- worker capacity,
- worker heartbeat,
- execution heartbeat,
- job handler protocol.

Does not decide:
- whether a run is operationally "lost",
- whether a queue is "degraded",
- whether PII violates policy.

Those are monitoring interpretations.

### 3.5 Monitoring & Observability

Owns:
- durable lifecycle event history,
- event normalization,
- current-state projections for investigation,
- run timelines,
- queue health projections,
- worker health projections,
- anomaly evidence,
- monitoring completeness/gaps,
- project component-health projections,
- monitoring/live-pipeline freshness evidence.

This is the central context of the product.

### 3.6 PII Detection

Owns:
- PII detectors,
- detection policies,
- classification,
- findings,
- masking/redaction decisions,
- scan status.

It does not own the job's execution state unless a policy explicitly instructs the execution context to block processing.

### 3.7 Alerting

Owns:
- alert rules,
- severity,
- alert instances,
- acknowledgement,
- resolution.

Alerting consumes facts/derived signals from Monitoring and PII Detection.

### 3.8 Query & Investigation

Owns read-oriented models and user-facing investigation semantics:
- job search,
- dashboard summaries,
- run detail,
- queue views,
- worker views,
- PII-safe views.

It does not become the authoritative writer for upstream domains.

### 3.9 Dashboard Frontend

Owns:
- operator navigation,
- visualization,
- live/polling presentation state,
- investigation URL/filter state,
- safe frontend interaction behavior.

It does not own authoritative monitoring facts or derive backend health independently.

### 3.10 Real-Time Monitoring

Owns:
- low-latency monitoring change delivery,
- subscription semantics,
- bounded replay/resume cursors,
- stream backpressure/coalescing,
- live connection health,
- explicit resynchronization behavior.

It does not own durable monitoring truth. Query snapshots and Monitoring projections remain authoritative.

## 4. Shared Language

### Job Definition

Identifies a type of executable background work.

Example:

```text
generate_customer_report
```

### Execution Chain

Groups the initial run and policy-level retry runs originating from the same logical trigger.

### Run

One policy-level execution. A policy retry creates a new `run_id` in the same `execution_chain_id` and links it through `parent_run_id`.

### Attempt

One worker/delivery execution ownership attempt within a run. Infrastructure reclaim may create another attempt without creating a policy retry.

```text
execution-chain-1
├── run-123 FAILED
│   └── attempt-1
└── run-124 SUCCEEDED  parent=run-123
    └── attempt-2
```

### Delivery

Redis-specific transfer of work to a consumer.

Delivery is not equivalent to execution.

### Claim

A worker has obtained responsibility for a delivery.

### Worker

Independent process capable of executing one or more job types.

### Monitoring Event

An immutable observation about something that happened.

### Alert

A derived operational condition requiring attention.

### PII Finding

A structured observation that content probably contains a PII category.

## 5. Integration Rules

### 5.1 No Cross-Context Table Ownership

One context must not update another context's private persistence tables.

Use:
- commands,
- APIs,
- published events,
- replicated read models.

### 5.2 Stable IDs Cross Contexts

The following identifiers are cross-context correlation keys:

```text
job_definition_id
execution_chain_id
run_id
attempt_id
schedule_id
schedule_occurrence_id
worker_id
queue_name
correlation_id
trace_id
```

Not every event requires every ID.

### 5.3 Domain Events Are Facts

Preferred event naming:

```text
run.created
run.queued
attempt.claimed
attempt.started
attempt.heartbeat
attempt.failed
attempt.succeeded
run.retry_scheduled

worker.registered
worker.heartbeat

schedule.occurrence_due

pii.scan_completed
pii.detected

alert.opened
alert.resolved
```

Events should represent facts that occurred, not UI instructions.

### 5.4 At-Least-Once Is Assumed

Consumers must assume duplicate events/messages may arrive.

Every cross-context event requires:
- `event_id`,
- producer identity,
- timestamp,
- schema version.

Consumers must handle duplicate `event_id` safely.

## 6. Source-of-Truth Matrix

| Concept | Source of Truth |
|---|---|
| Run identity | Job Run Lifecycle |
| Attempt state | Job Run Lifecycle |
| Schedule definition | Scheduling |
| Redis delivery state | Redis Delivery / Redis runtime |
| Worker heartbeat | Worker Runtime |
| Historical lifecycle timeline | Monitoring |
| PII finding | PII Detection |
| Alert lifecycle | Alerting |
| Search projection | Query & Investigation |

## 7. Monitoring-First Boundary

The following capabilities are intentionally secondary:

- advanced DAG workflows,
- task dependency graphs,
- distributed transactions,
- arbitrary workflow compensation,
- complex business orchestration.

The system may evolve later, but those concerns must not distort the initial monitoring design.

## 8. Failure Classification Rule

The platform must distinguish:

```text
business execution failure
worker/runtime failure
Redis/infrastructure failure
scheduler failure
PII scanning failure
monitoring ingestion failure
alert-delivery failure
```

A failure in one context must not automatically overwrite another context's state.

Example:

```text
Job result:     SUCCEEDED
PII scan:       FAILED
Monitoring:     PARTIAL
Alert delivery: FAILED
```

All four facts may be true simultaneously.

## 9. Open Questions

1. Which integration boundaries should initially be synchronous versus event-driven?
2. Is the first implementation a modular monolith or multiple deployables?
3. What event schema/versioning strategy will be used?
4. Which timestamps are producer-generated versus canonicalized during ingestion?
5. Should Redis Delivery expose Redis Streams details directly to Monitoring, or normalize them first?
