# RFC-001: Job Run Lifecycle Context

- **Status:** Draft
- **Context:** Job Run Lifecycle
- **Depends On:** RFC-000
- **Consumers:** Scheduling, Redis Delivery, Worker Runtime, Monitoring, Query & Investigation

## 1. Summary

This context owns durable execution identity, job-run state, retry lineage, and worker execution attempts.

The key modeling decision is:

> A policy-level retry creates a new `run_id` linked to the failed run through `parent_run_id` and grouped by a stable `execution_chain_id`.

A worker/delivery re-attempt does **not** necessarily create a new run. `attempt_id` remains the identity of an individual worker ownership/execution attempt for one run.

This produces three distinct levels:

```text
Job Definition
      │
      ▼
Execution Chain
      │
      ├── Run 1
      │     └── Attempt(s)
      │
      ├── Run 2 [retry of Run 1]
      │     └── Attempt(s)
      │
      └── Run 3 [retry of Run 2]
            └── Attempt(s)
```

The model is intended to make retry history explicit without overloading one `run_id` with several policy-level executions.

## 2. Goals

- provide durable execution-chain and run identity,
- make every retry independently addressable,
- preserve parent/child retry lineage,
- distinguish policy retries from delivery/worker attempts,
- track authoritative run execution state,
- support retry visibility and retry exhaustion,
- distinguish terminal execution state from monitoring annotations,
- tolerate duplicate delivery and duplicate command signals,
- provide stable correlation keys for monitoring and investigation.

## 3. Non-Goals

- implementing a workflow DAG,
- treating retry lineage as general job dependency orchestration,
- owning Redis pending-entry state,
- determining queue health,
- determining worker health,
- storing raw worker logs,
- classifying PII.

## 4. Domain Model

### JobDefinition

```text
job_definition_id
name
version
default_queue
enabled
```

### ExecutionChain

An execution chain groups the initial run and policy-level retries caused by the same original trigger.

```text
execution_chain_id
job_definition_id
created_at
source_run_id?        # when created from rerun/replay of older work
```

The chain is primarily a lineage/correlation boundary. Its operational outcome may be derived from the runs in the chain rather than treated as a second execution state machine.

### JobRun

One policy-level execution of a job.

```text
run_id
execution_chain_id
job_definition_id
parent_run_id?
retry_index
trigger_type
retry_reason?
scheduled_for?
created_at
current_state
correlation_id?
trace_id?
```

Suggested `trigger_type` values:

```text
SCHEDULED
API
MANUAL
AUTOMATIC_RETRY
MANUAL_RETRY
REPLAY
```

### Attempt

One worker ownership/execution attempt for one run.

```text
attempt_id
run_id
attempt_number
worker_id?
state
claimed_at?
started_at?
finished_at?
failure_category?
```

A run normally has one attempt, but more than one attempt may exist when the delivery/runtime layer must recover from uncertain ownership, reclaim, or redelivery. That recovery is different from a retry-policy decision.

## 5. Run State Model

A single run does not return from terminal failure to queued execution.

```text
CREATED
  │
  ├── delayed/scheduled ───────► SCHEDULED
  │                                │
  └────────────────────────────────┘
                                   ▼
                                QUEUED
                                   │
                                   ▼
                                RUNNING
                              ┌────┴─────┐
                              ▼          ▼
                          SUCCEEDED    FAILED
```

Additional terminal states may include:

```text
CANCELLED
TIMED_OUT
DEAD
```

A retry is represented by a **new child run**, not by moving the failed run back to `QUEUED`.

```text
run-101 FAILED
    │
    │ retry policy allows retry
    ▼
run-102 SCHEDULED
parent_run_id = run-101
execution_chain_id = same chain
retry_index = 1
    │
    ▼
QUEUED → RUNNING → ...
```

`LOST`, `STUCK`, and `SLA_VIOLATED` should normally be monitoring interpretations, not authoritative execution states.

## 6. Attempt State Model

```text
CREATED
   │
   ▼
CLAIMED
   │
   ▼
STARTED
   │
 ┌─┴───────────────┐
 ▼                 ▼
SUCCEEDED        FAILED
```

An implementation may add states such as:

```text
ABANDONED
CANCELLED
TIMED_OUT
```

`ABANDONED` can be useful when ownership is lost and transport recovery creates another attempt for the **same run**. A policy retry after a run failure creates a **new run**.

## 7. Retry Lineage

Example:

```text
execution_chain_id = chain-55

run-101
├── parent_run_id = null
├── retry_index = 0
└── FAILED
      │
      ▼
run-102
├── parent_run_id = run-101
├── retry_index = 1
└── FAILED
      │
      ▼
run-103
├── parent_run_id = run-102
├── retry_index = 2
└── SUCCEEDED
```

Monitoring/query clients should be able to derive an overall presentation such as:

```text
SUCCEEDED_AFTER_2_RETRIES
```

without rewriting any historical run state.

## 8. Invariants

1. `execution_chain_id` is globally unique.
2. `run_id` is globally unique.
3. `attempt_id` is globally unique.
4. `attempt_number` is unique within one run.
5. `retry_index` starts at `0` for the root run.
6. A policy-level retry creates a new `run_id`.
7. An automatic/manual retry within the same logical execution retains the same `execution_chain_id`.
8. A retry run references the run that caused it through `parent_run_id`.
9. A child retry has `retry_index = parent.retry_index + 1`.
10. A failed parent run remains terminal after its retry child is created.
11. A run cannot be `SUCCEEDED` without at least one succeeded attempt.
12. Terminal run states must not silently transition back to running.
13. Duplicate retry commands must not create duplicate retry children for the same retry decision.
14. Monitoring annotations must not overwrite authoritative execution state.

## 9. Commands

Conceptual commands include:

```text
CreateInitialRun
MarkRunQueued
CreateAttempt
MarkAttemptClaimed
MarkAttemptStarted
MarkAttemptSucceeded
MarkAttemptFailed
MarkRunSucceeded
MarkRunFailed
ScheduleRetryRun
CreateManualRetryRun
CancelRun
```

Command names are descriptive, not mandatory API names.

## 10. Events

Suggested events:

```text
run.created
run.scheduled
run.queued
run.cancelled
run.succeeded
run.failed
run.dead
run.retry_scheduled
run.retry_exhausted

attempt.created
attempt.claimed
attempt.started
attempt.heartbeat_observed
attempt.succeeded
attempt.failed
attempt.abandoned
```

A retry child should emit normal run lifecycle events. The linkage metadata makes its origin explicit.

Example child-run event:

```json
{
  "event_id": "evt-...",
  "event_type": "run.created",
  "schema_version": 1,
  "occurred_at": "2026-08-15T10:05:00Z",
  "execution_chain_id": "chain-55",
  "run_id": "run-102",
  "parent_run_id": "run-101",
  "retry_index": 1,
  "trigger_type": "AUTOMATIC_RETRY"
}
```

Example attempt event:

```json
{
  "event_id": "evt-...",
  "event_type": "attempt.started",
  "schema_version": 1,
  "occurred_at": "2026-08-15T10:05:01Z",
  "execution_chain_id": "chain-55",
  "run_id": "run-102",
  "attempt_id": "attempt-77",
  "worker_id": "worker-7"
}
```

## 11. Retry Semantics

Policy-level retry flow:

```text
run fails
   │
   ▼
retry decision
   │
   ├── not eligible ─────► run.retry_exhausted / no child
   │
   ▼
create child run
new run_id
same execution_chain_id
parent_run_id = failed run
   │
   ▼
SCHEDULED until retry_available_at
   │
   ▼
QUEUED → RUNNING
```

The failed parent remains queryable as a complete independent execution.

Monitoring must be able to answer:

- why the parent run failed,
- which worker/attempt executed it,
- why a retry was permitted,
- when the retry became eligible,
- which child run was created,
- when that retry run started,
- whether a later run in the chain eventually succeeded,
- when retry policy became exhausted.

The exact retry policy may live with the Job Run context or an execution-policy component, but its decisions must be observable.

## 12. Retry vs Attempt vs Rerun vs Replay

### Worker/Delivery Attempt

```text
same run_id
new attempt_id
```

Used for ownership/recovery semantics within the same policy-level execution.

### Automatic Retry

```text
same execution_chain_id
new run_id
parent_run_id = failed run
trigger_type = AUTOMATIC_RETRY
```

### Manual Retry

```text
same execution_chain_id
new run_id
parent_run_id = selected run
trigger_type = MANUAL_RETRY
```

A manual retry continues the existing logical execution lineage.

### Rerun

A rerun is a new logical execution initiated using an older run as context.

```text
new execution_chain_id
new run_id
source_run_id = old run
trigger_type = MANUAL
```

### Replay

A replay is a new logical execution that deliberately reconstructs prior inputs and/or configuration according to an explicit replay policy.

```text
new execution_chain_id
new run_id
source_run_id = old run
trigger_type = REPLAY
```

Rerun/replay must not be confused with automatic retry lineage.

## 13. Idempotency

At-least-once commands/events are expected.

Potential idempotency keys:

```text
create initial run  → producer_request_id
create attempt      → run_id + attempt_number
complete attempt    → attempt_id + completion_kind
schedule retry run  → parent_run_id + retry_index + retry_policy_decision_id
```

The implementation must define behavior for conflicting duplicate commands.

Example:

```text
attempt-1 already SUCCEEDED
then receives MarkAttemptFailed
```

This must be rejected or recorded as a conflicting signal; it must not silently rewrite history.

A repeated retry-scheduling command must return/reuse the already-created child run rather than create siblings accidentally.

## 14. Failure Categories

Suggested categories:

```text
APPLICATION_ERROR
INVALID_INPUT
DEPENDENCY_ERROR
TIMEOUT
WORKER_FAILURE
INFRASTRUCTURE_ERROR
UNKNOWN
```

These categories are intentionally broader than concrete exception types.

## 15. Monitoring Requirements

Every material state transition and retry-lineage decision must be externally observable.

Minimum run timestamps:

```text
created_at
scheduled_for?
queued_at
started_at
finished_at
```

Minimum retry metadata:

```text
execution_chain_id
parent_run_id
retry_index
trigger_type
retry_reason?
retry_available_at?
```

From attempts and run timestamps the monitoring context can derive:

```text
queue latency
claim-to-start latency
execution duration
run duration
retry delay
chain total duration
retry count
```

## 16. Open Questions

1. Should retry-policy ownership remain inside this context?
2. Should an eligible retry child be created immediately in `SCHEDULED`, or should only a durable retry intent be stored until it becomes due?
3. Can the Job Run context transition a run to `TIMED_OUT`, or should timeout first be a monitoring signal?
4. How are conflicting terminal signals reconciled?
5. Should cancellation be best-effort or guaranteed?
6. What worker-ownership/fencing rule determines whether a reclaimed delivery creates another attempt under the same run?
7. Should manual retry always continue the same execution chain, or may operators explicitly request a new rerun chain?
