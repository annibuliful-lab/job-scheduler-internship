# RFC-002: Scheduling Context

- **Status:** Draft
- **Context:** Scheduling
- **Depends On:** RFC-000, RFC-001
- **Consumers:** Job Run Lifecycle, Monitoring, Query & Investigation

## 1. Summary

This context owns definitions that determine **when a logical job run should be created**.

It does not own job execution.

The monitoring product must be able to compare expected schedule behavior with actual run behavior.

## 2. Goals

- one-time scheduling,
- recurring scheduling,
- deterministic schedule occurrences,
- prevention/detection of duplicate occurrence creation,
- calculation of next due occurrence,
- emission of schedule facts for monitoring,
- visibility of schedule drift and missed occurrences.

## 3. Non-Goals

- executing jobs,
- claiming Redis messages,
- owning retry policy or retrying failed runs,
- determining worker health.

## 4. Domain Model

### ScheduleDefinition

```text
schedule_id
job_definition_id
schedule_type
schedule_expression
timezone
enabled
created_at
updated_at
```

### ScheduleOccurrence

Represents one expected firing.

```text
schedule_occurrence_id
schedule_id
expected_at
created_run_id?
status
```

Possible occurrence status:

```text
PENDING
RUN_CREATED
SKIPPED
MISSED
```

Whether `MISSED` is stored here or derived by Monitoring remains an implementation decision.

## 5. Schedule Types

Minimum required types:

```text
ONE_TIME
RECURRING
```

Potential later extension:

```text
INTERVAL
CALENDAR
EXTERNAL_TRIGGER
```

## 6. Core Invariant

For one schedule occurrence, the scheduler must not intentionally create multiple logical runs.

A useful idempotency boundary is:

```text
(schedule_id, expected_at)
```

or an explicit:

```text
schedule_occurrence_id
```

## 7. Scheduling Flow

```text
Schedule Definition
       │
       ▼
Determine occurrence due
       │
       ▼
Create occurrence
       │
       ▼
Request Job Run creation
       │
       ▼
Associate run_id with occurrence
```

The scheduler must be resilient to process restart between these steps.

## 8. Schedule Drift

Monitoring needs at least:

```text
expected_at
run_created_at
run_queued_at
run_started_at
```

Possible derived metrics:

```text
creation_drift = run_created_at - expected_at
start_drift    = run_started_at - expected_at
```

The product should state clearly which drift is shown in the UI.

## 9. Missed Occurrence

Conceptually:

```text
current_time >
expected_at + allowed_delay
```

and no valid run exists.

The exact interpretation belongs to Monitoring because a scheduler may be temporarily unavailable and later recover.

## 10. Scheduler Restart / Catch-Up

The implementation must define behavior when the scheduler is unavailable across one or more due times.

Possible policies:

```text
CATCH_UP_ALL
CATCH_UP_LATEST
SKIP_MISSED
```

This RFC does not force one global policy.

The selected policy must be visible per schedule.

## 11. Events

Suggested events:

```text
schedule.created
schedule.updated
schedule.enabled
schedule.disabled
schedule.occurrence_due
schedule.occurrence_run_created
schedule.occurrence_skipped
```

Monitoring should not need to inspect scheduler-internal tables to determine what was expected.

## 12. Timezone Requirements

A recurring schedule must explicitly define its timezone or explicitly use a documented system default.

DST and timezone changes must not be hidden implementation details.

For every occurrence, store the resolved absolute timestamp:

```text
expected_at
```

rather than requiring later reconstruction from cron text.

## 13. Failure Semantics

Scheduler errors are not job failures.

Example:

```text
Schedule should fire at 09:00
Scheduler unavailable until 09:20
No run exists
```

This is a scheduling health issue.

It must not fabricate:

```text
run.status = FAILED
```

because no run ever executed.

## 14. Monitoring Requirements

The context must expose enough information to answer:

- What schedules exist?
- Which are enabled?
- What should run next?
- What should have run recently?
- Was a run created for each expected occurrence?
- How late was run creation?
- How late did execution actually begin?

## 15. Open Questions

1. Cron syntax, interval model, or both?
2. Who calculates recurring occurrences: application code or Redis?
3. What catch-up policy is the default?
4. How many future occurrences are materialized?
5. Is `MISSED` a scheduling state or a monitoring annotation?
