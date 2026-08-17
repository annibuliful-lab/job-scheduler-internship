# Product Requirements

## Distributed Job Scheduling Monitoring with Redis and PII Detection

**Document Status:** Draft
**Product Type:** Internal platform / developer tooling
**Primary Focus:** Distributed job execution monitoring
**Supporting Capabilities:** Redis-backed distributed job processing, scheduling visibility, PII detection, operator dashboard, real-time whole-platform observation
**Revision:** 1.3

---

# 1. Product Overview

The product provides centralized visibility into distributed background jobs executed by multiple workers using Redis as the coordination and messaging layer.

The primary purpose is **not to build another generic job scheduler**. Scheduling, queueing, retrying, and worker execution are treated as mechanisms whose behavior must be observable.

The system must answer questions such as:

- What jobs are scheduled?
- Which jobs are waiting?
- Which worker picked up a job?
- Is the job still running?
- How long has it been running?
- Is the worker still alive?
- Did a job finish successfully?
- Did it fail?
- Is it being retried?
- Has the job been delivered more than once?
- Has a job become stuck or lost?
- Is a queue becoming unhealthy?
- Is execution slower than expected?
- Is a scheduled job starting late?
- Is sensitive PII flowing through job payloads, results, or logs?
- Where was the PII detected?
- What type of PII was found?
- Can operators investigate a problem without exposing the sensitive data itself?
- Can operators see important state and health changes as they happen without manually refreshing?
- Is the monitoring platform itself healthy enough for live data to be trusted?
- Which project component is degraded or offline right now?

The product should provide a monitoring layer over the complete lifecycle:

```text
Producer
   │
   ▼
Scheduler / API
   │
   ▼
Redis
   │
   ├── Pending Job
   │
   ▼
Worker Pool
   │
   ├── Running
   ├── Heartbeat
   ├── Success
   ├── Failure
   └── Retry
   │
   ▼
Monitoring Platform
   │
   ├── Job State
   ├── Worker Health
   ├── Queue Health
   ├── Scheduling Health
   ├── Alerts
   ├── PII Detection
   ├── Project Component Health
   └── Real-Time Observation
```

---

# 2. Problem Statement

Background jobs become increasingly difficult to operate when processing is distributed across multiple workers.

Traditional application logs are insufficient because information about a single job may be spread across:

- API servers
- schedulers
- Redis
- multiple worker processes
- application logs
- retry runs and worker execution attempts
- different hosts or containers

Several operational problems become difficult to identify.

Examples include:

- jobs waiting indefinitely
- jobs processed significantly later than their intended schedule
- workers disappearing during execution
- duplicate processing
- repeated failures
- retry loops
- rapidly growing Redis queues
- workers consuming jobs slower than producers generate them
- executions exceeding expected duration
- jobs appearing as running even though the worker has disappeared
- sensitive data accidentally being included in job payloads or logs

Operators therefore need a system that reconstructs the complete lifecycle of every distributed job and highlights abnormal behavior.

---

# 3. Product Objective

Build a monitoring platform capable of reconstructing, observing, and analyzing distributed background-job execution using Redis.

The platform must provide six primary capabilities:

1. **Job lifecycle visibility**
2. **Distributed worker and Redis queue monitoring**
3. **Anomaly and failure detection**
4. **PII detection throughout job execution**
5. **Operator dashboard and investigation experience**
6. **Real-time whole-platform observation**

The product should allow an operator to move from:

```text
"Something looks wrong"
```

to:

```text
Which job?
Which queue?
Which worker?
Which attempt?
When did it become abnormal?
Why was it flagged?
Was sensitive information involved?
```

without manually correlating application logs.

---

# 4. Product Principles

## 4.1 Monitoring First

The product is primarily a monitoring platform.

Features such as retries, scheduling, and worker coordination are important because they generate operational states that must be understood.

The product does not need to become a full workflow engine.

---

## 4.2 Distributed Execution Is Expected

The architecture must assume:

```text
1 API / scheduler
        │
        ▼
      Redis
        │
        ├────────────┬────────────┐
        ▼            ▼            ▼
     Worker A     Worker B     Worker C
```

Workers must be independent processes.

The monitoring system must not depend on workers being:

- inside the same process
- on the same machine
- in the same container
- permanently available

---

## 4.3 Delivery Does Not Equal Execution

The system must distinguish:

```text
Job queued
Job delivered
Job claimed
Job started
Job running
Job completed
```

These are different states.

For example:

```text
Redis delivered message
        ≠
Job successfully completed
```

---

## 4.4 Monitoring Data Must Be Durable Enough for Investigation

Redis may contain transient runtime information.

Monitoring history must not disappear simply because:

- Redis trimmed a stream
- a message was acknowledged
- a worker restarted
- the Redis instance restarted
- the queue was cleared

Historical monitoring information therefore requires durable storage separate from ephemeral Redis state.

---

## 4.5 PII Detection Must Minimize PII Exposure

The detection system must avoid turning the monitoring platform itself into a new source of sensitive-data exposure.

The platform should prefer storing:

```text
PII detected: true
Type: national_id
Location: payload.customer.id_number
Confidence: 0.99
```

instead of:

```text
Detected value: 1234567890123
```

Raw sensitive values should not be stored unless explicitly permitted by policy.

## 4.6 Operational UI Must Represent Freshness and Uncertainty

The dashboard is part of the monitoring product and must not make stale or incomplete information appear authoritative.

The operator experience must distinguish:

```text
execution state
monitoring interpretation
data freshness
monitoring completeness
```

For example:

```text
Execution state: RUNNING
Monitoring health: RUN_LOST
Projection freshness: 3s
Redis observation: STALE — last successful sample 2m ago
```

The frontend must expose degraded monitoring when the evidence required to evaluate system health is delayed or unavailable.

## 4.7 Real-Time Is a Delivery Property, Not a New Source of Truth

The product should surface important operational changes shortly after they are observed, but the live connection must never replace durable monitoring history or authoritative read models.

The expected model is:

```text
Durable monitoring facts / projections
            │
            ├── Query snapshots for correctness
            │
            └── Live changes for low-latency awareness
```

If a live stream disconnects, falls behind, or loses replay history, the dashboard must resynchronize from a current snapshot and visibly communicate the degraded period.

The initial operational near-real-time target is that important monitoring changes become operator-visible within **5 seconds at p95 under normal load**, subject to validation and configuration for the deployment. This is not a hard real-time guarantee.

---

# 5. Scope

## 5.1 In Scope

The product covers:

### Distributed jobs

- immediate jobs
- scheduled jobs
- recurring jobs
- delayed jobs

### Redis monitoring

- queue depth
- pending jobs
- consumer activity
- worker consumption
- consumer lag
- message age
- delivery count
- unacknowledged work

### Job execution

- job creation
- scheduling
- queueing
- claiming
- execution
- success
- failure
- retry
- timeout
- stuck execution
- lost execution
- duplicate delivery

### Worker monitoring

- registration
- worker identity
- heartbeat
- active jobs
- last activity
- worker disappearance
- capacity

### PII monitoring

Detection in:

- job payload
- job metadata
- execution result
- error messages
- structured worker logs
- execution context

### Operational interfaces

- system-health overview dashboard
- job search and filtered run lists
- job execution detail and chronological timeline
- queue monitoring and backlog investigation
- worker health and capacity monitoring
- schedule health and drift investigation
- alert investigation
- PII findings
- monitoring-system health and data-freshness visibility
- shareable deep links that preserve safe investigation context
- consistent loading, empty, stale, partial, error, and forbidden states
- live state/health updates without manual refresh
- reconnect, bounded replay, and snapshot resynchronization
- live connection/fallback status
- project component health for API, scheduler, Redis delivery, workers, monitoring, PII, alerting, query, and real-time gateway components
- sanitized real-time activity feed for meaningful operational changes

---

# 6. Out of Scope

The initial product does not attempt to become:

- a general workflow/orchestration engine
- a BPM platform
- a distributed transaction system
- a replacement for Kubernetes
- a generic log-management platform
- a full SIEM
- an arbitrary event-processing platform
- a data-loss-prevention gateway
- a general-purpose BI/dashboard builder
- a direct Redis or database administration UI

Dependencies between arbitrary jobs such as:

```text
A → B → C → D
```

are not required unless later introduced as a separate workflow capability.

The system may expose retry configuration and retry history, but advanced retry orchestration is not the primary product goal.

---

# 7. Terminology

## Job

Logical unit of background work.

Example:

```text
generate_daily_report
```

---

## Execution Chain

One logical execution lineage originating from an initial trigger and including any policy-level retries.

```text
Job
 └── Execution Chain
      ├── Run 1
      ├── Run 2 [retry]
      └── Run 3 [retry]
```

The stable `execution_chain_id` allows operators to inspect the complete retry lineage without treating multiple executions as one mutable run.

---

## Job Run

One policy-level execution of a job.

```text
execution_chain_id = chain-001

run-001 → FAILED
run-002 → FAILED    parent_run_id = run-001
run-003 → SUCCEEDED parent_run_id = run-002
```

A retry creates a **new `run_id`** while retaining the same `execution_chain_id`.

---

## Attempt

A specific worker ownership/execution attempt for one run.

```text
run-002
 ├── attempt-1 → ownership lost / abandoned
 └── attempt-2 → execution completed
```

Multiple attempts under one run are reserved for delivery/runtime recovery semantics such as reclaim or redelivery. They are not the policy-level retry history.

---

## Queue

Redis-backed channel through which jobs are delivered to workers.

---

## Worker

Independent process capable of consuming and executing jobs.

---

## Schedule

Definition determining when a job should be created or executed.

---

## Heartbeat

Periodic signal proving that a worker or running execution remains alive.

---

## PII Finding

Detection that a field or piece of content probably contains personally identifiable information.

---

# 8. Core Job Lifecycle

The monitoring system must understand the following state model for a **single run**.

```text
CREATED
   │
   ▼
SCHEDULED
   │
   ▼
QUEUED
   │
   ▼
CLAIMED
   │
   ▼
RUNNING
   │
   ├───────────────┐
   │               │
   ▼               ▼
SUCCEEDED        FAILED
```

A failed run is terminal. If retry policy allows another execution, the platform creates a child run:

```text
run-001 FAILED
    │
    │ retry scheduled
    ▼
run-002 SCHEDULED
parent_run_id = run-001
execution_chain_id = same chain
    │
    ▼
QUEUED → RUNNING → ...
```

Retry waiting therefore belongs to the relationship between runs, not to a failed run transitioning back to queued execution.

Additional terminal or abnormal states may include:

```text
CANCELLED
TIMED_OUT
LOST
DEAD
```

Monitoring annotations may include:

```text
STUCK
SLA_VIOLATED
DUPLICATE_SUSPECTED
PII_DETECTED
```

An annotation does not necessarily replace the job's execution state.

For example:

```text
State: RUNNING
Alerts:
- SLA_VIOLATED
- PII_DETECTED
```

---

# 9. Job Identity Requirements

Every execution chain, job run, and worker execution attempt must have a globally unique identifier.

Required identifiers:

```text
job_definition_id
execution_chain_id
run_id
attempt_id
```

Retry lineage must also preserve:

```text
parent_run_id
retry_index
trigger_type
```

Where applicable:

```text
schedule_id
queue_name
worker_id
correlation_id
trace_id
```

A single `run_id` must remain visible throughout the execution of that run:

```text
Run creation
→ Redis delivery
→ Worker
→ Monitoring
```

A policy retry must create a new `run_id` while retaining the same `execution_chain_id`.

Example:

```text
execution_chain_id: chain-55

run-101:
  parent_run_id: null
  retry_index: 0
  attempt_id: a1
  result: FAILED

run-102:
  parent_run_id: run-101
  retry_index: 1
  attempt_id: a2
  result: SUCCEEDED
```

If delivery/runtime recovery creates another worker attempt for the same run, only the `attempt_id` changes.

---

# 10. Functional Requirements

# 10.1 Job Submission

The platform must support creation of background jobs containing:

- job type
- payload
- queue
- creation time
- scheduling information
- execution configuration
- metadata
- PII scanning configuration

Example conceptual request:

```json
{
  "jobType": "generate_customer_report",
  "queue": "reports",
  "payload": {},
  "metadata": {},
  "scheduledAt": null
}
```

The system must assign a `run_id`.

---

# 10.2 Immediate Jobs

The system must support jobs intended to execute as soon as worker capacity becomes available.

The monitoring system must record:

```text
created_at
queued_at
claimed_at
started_at
completed_at
```

This allows calculation of:

```text
queue latency
execution duration
total duration
```

---

# 10.3 Scheduled Jobs

The product must support monitoring of jobs intended to start at a future time.

Required data:

```text
scheduled_at
actual_queued_at
actual_started_at
```

The system must calculate schedule drift:

```text
schedule_drift =
actual_started_at - scheduled_at
```

Example:

```text
Scheduled: 09:00:00
Started:   09:04:31

Schedule drift = 4m31s
```

---

# 10.4 Recurring Jobs

The system should represent recurring jobs as:

```text
Schedule Definition
      │
      ├── Run 1
      ├── Run 2
      ├── Run 3
      └── Run 4
```

Each occurrence must receive its own `run_id`.

Operators must be able to inspect previous occurrences.

---

# 10.5 Queue Routing

Jobs must identify the logical queue used for processing.

Example:

```text
email
reports
pii_scan
file_processing
priority
```

Monitoring must be available separately for each queue.

---

# 11. Redis Requirements

Redis acts as the runtime messaging and coordination substrate.

The exact Redis primitive may be selected by the implementation, but the product requirements assume support for reliable distributed consumption semantics.

The implementation should be capable of representing:

```text
Producer
   │
   ▼
Redis
   │
   ├── Consumer A
   ├── Consumer B
   └── Consumer C
```

Redis integration must expose sufficient information for monitoring:

- message identity
- queue identity
- enqueue time
- consumer identity
- delivery time
- acknowledgement
- delivery count where available
- pending status
- message age

For implementations using Redis Streams, consumer groups naturally fit these requirements.

---

# 12. Queue Monitoring

The system must display queue-level health.

Required information includes:

```text
queue name
queued jobs
pending jobs
active workers
oldest pending job age
ingress rate
completion rate
failure rate
retry rate
```

Where available:

```text
consumer lag
pending-entry count
delivery count
```

Example:

| Queue    | Waiting | Running | Workers | Oldest | Status   |
| -------- | ------: | ------: | ------: | -----: | -------- |
| email    |       4 |       2 |       3 |     2s | Healthy  |
| report   |   1,230 |       5 |       2 |    18m | Degraded |
| pii-scan |       0 |       1 |       2 |     0s | Healthy  |

---

# 13. Queue Backlog Detection

The platform must detect abnormal queue growth.

Potential signals include:

```text
queue depth
oldest-message age
enqueue rate
completion rate
worker availability
```

Example:

```text
Producer rate: 100 jobs/min
Worker throughput: 40 jobs/min
```

The monitoring system should detect that backlog is increasing even before the queue becomes extremely large.

---

# 14. Worker Registration

Each worker must have a unique `worker_id`.

Worker metadata should include:

```text
worker_id
instance
hostname
process_id
version
started_at
queues
concurrency
last_heartbeat
```

Additional infrastructure metadata may include:

```text
pod
node
availability_zone
deployment_version
```

---

# 15. Worker Heartbeat

Workers must periodically emit heartbeat information.

The monitoring system must determine:

```text
ONLINE
DEGRADED
OFFLINE
```

Example policy:

```text
heartbeat interval: 10 seconds

< 20s       → ONLINE
20–60s      → DEGRADED
> 60s       → OFFLINE
```

Exact thresholds must be configurable.

---

# 16. Worker Capacity

The system must display worker execution capacity.

Example:

```text
worker-a
concurrency: 10
running: 8
available: 2
```

Aggregate views should show:

```text
report queue

Workers:            5
Execution capacity: 50
Currently running:  46
Utilization:        92%
```

---

# 17. Job Execution Tracking

When a worker receives a job, the platform must record:

```text
claimed_at
started_at
worker_id
attempt_id
```

While running, the system should be capable of receiving execution heartbeats or progress signals.

Example:

```json
{
  "runId": "run-123",
  "attemptId": "attempt-2",
  "progress": 54
}
```

Progress reporting is optional for job types that cannot calculate meaningful progress.

---

# 18. Job Completion

Successful executions must record:

```text
completed_at
duration
worker_id
attempt
result status
```

The product should avoid retaining entire result bodies unless configured to do so.

---

# 19. Job Failure

Failures must record:

```text
failure timestamp
failure category
error code
sanitized error message
worker
attempt
execution duration
retry eligibility
```

Failure categories should support at least:

```text
APPLICATION_ERROR
TIMEOUT
DEPENDENCY_ERROR
INVALID_INPUT
WORKER_FAILURE
INFRASTRUCTURE_ERROR
UNKNOWN
```

---

# 20. Retry Visibility

Policy-level retries must be visible as distinct runs in one execution chain.

Example:

```text
Execution Chain: chain-001

run-101
Retry index: 0
Worker: worker-1
Duration: 13s
Result: FAILED

run-102
Parent: run-101
Retry index: 1
Worker: worker-3
Duration: 9s
Result: FAILED

run-103
Parent: run-102
Retry index: 2
Worker: worker-2
Duration: 11s
Result: SUCCEEDED
```

Monitoring must distinguish:

```text
retry scheduled
retry waiting
retry run created
retry queued
retry executing
retry succeeded
retry exhausted
```

The UI/read model should make both the selected run and the complete execution chain visible.

Worker/delivery recovery attempts inside one run must remain separately inspectable and must not be presented as policy retries.

The monitoring system does not need to own the retry policy to observe these facts.

---

# 21. Duplicate Execution Detection

Distributed systems may deliver the same job more than once.

The platform should detect or highlight suspected duplicates using identifiers such as:

```text
run_id
attempt_id
message_id
idempotency_key
```

Example:

```text
run-123
   ├── worker-a RUNNING
   └── worker-b RUNNING

→ DUPLICATE_SUSPECTED
```

The product should not automatically assume that every duplicate delivery represents a correctness failure.

---

# 22. Stuck Job Detection

A running job must be considered potentially stuck when:

```text
current_time - last_execution_heartbeat > configured threshold
```

or when:

```text
execution duration > expected maximum duration
```

Example:

```text
Expected duration: < 5m
Current duration:  27m

Alert:
RUN_STUCK
```

---

# 23. Lost Job Detection

The system must distinguish a stuck job from a lost job.

Example:

```text
Job state: RUNNING
Worker: worker-17
Worker heartbeat: missing
Job heartbeat: missing
Redis claim: still pending
```

The monitoring platform may classify:

```text
RUN_LOST
```

A lost job means the monitoring system cannot find evidence that the execution is still alive.

---

# 24. Timeout Monitoring

Jobs may define:

```text
expected duration
warning threshold
hard timeout
```

Example:

```text
Expected:     30s
Warning:      60s
Hard timeout: 120s
```

Monitoring must expose execution approaching or exceeding these limits.

---

# 25. Scheduling Monitoring

The scheduling view must show:

```text
job
schedule
next execution
last execution
last result
expected start
actual start
schedule drift
```

Example:

| Job          | Schedule | Expected | Started  | Drift |
| ------------ | -------- | -------- | -------- | ----: |
| daily-report | 09:00    | 09:00    | 09:00:03 |    3s |
| settlement   | 10:00    | 10:00    | 10:07:14 | 7m14s |

---

# 26. Missed Schedule Detection

A run should be considered potentially missed when:

```text
current_time >
scheduled_at + allowed_schedule_delay
```

and no corresponding run has been created or started.

Example alert:

```text
SCHEDULE_MISSED
```

---

# 27. Job Timeline

Every run must expose a chronological event timeline, and the product must also expose a chain-level timeline across retries.

Example:

```text
10:00:00.000  run-101 CREATED                  chain-55 retry=0
10:00:00.015  run-101 QUEUED
10:00:00.220  run-101 CLAIMED worker-03
10:00:00.225  run-101 STARTED
10:00:02.104  run-101 PII_SCAN_COMPLETED
10:00:12.828  run-101 FAILED
10:00:12.830  run-102 RETRY_SCHEDULED          parent=run-101 retry=1
10:05:12.900  run-102 QUEUED
10:05:13.041  run-102 CLAIMED worker-08
10:05:13.050  run-102 STARTED
10:05:21.200  run-102 SUCCEEDED
```

Operators must be able to switch between the exact run timeline and the complete execution-chain timeline.

This timeline is a core product capability.

---

# 28. Event Model

Monitoring should be built around immutable lifecycle events.

Conceptual examples:

```text
run.created
run.scheduled
run.queued
attempt.claimed
attempt.started
attempt.heartbeat
attempt.succeeded
attempt.failed
run.succeeded
run.failed
run.retry_scheduled
run.retry_exhausted
run.cancelled

worker.registered
worker.heartbeat
worker.offline

pii.detected

alert.triggered
alert.resolved
```

Events should contain:

```text
event_id
event_type
timestamp
execution_chain_id
run_id
parent_run_id
attempt_id
worker_id
queue
metadata
```

where relevant.

---

# 29. PII Detection Objective

The system must identify sensitive information entering or leaving background jobs without unnecessarily exposing that information to operators.

PII detection should behave as a monitoring capability.

Example:

```text
Job created
   │
   ▼
Payload scanner
   │
   ├── No findings
   │
   └── PII detected
           │
           ▼
      pii.detected
           │
           ▼
       Monitoring
```

---

# 30. PII Scanning Locations

PII scanning must support independently configurable scanning of:

```text
JOB_PAYLOAD
JOB_METADATA
JOB_RESULT
ERROR_MESSAGE
STRUCTURED_LOG
```

Each location must be distinguishable in findings.

Example:

```text
Source: JOB_PAYLOAD
Path: customer.phone
Type: PHONE_NUMBER
```

---

# 31. Supported PII Types

The initial detector should support configurable detection categories such as:

```text
EMAIL_ADDRESS
PHONE_NUMBER
NATIONAL_ID
PASSPORT_NUMBER
CREDIT_CARD_NUMBER
BANK_ACCOUNT
IP_ADDRESS
PERSON_NAME
ADDRESS
DATE_OF_BIRTH
```

Country-specific detectors should be extensible.

For deployments involving Thai data, the detector should be capable of supporting patterns such as:

```text
Thai national identification number
Thai telephone number
Thai address patterns
```

Support should not require the system to treat all numeric values as PII.

---

# 32. PII Detection Methods

The PII framework should allow multiple detection strategies.

Possible strategies include:

```text
pattern matching
validation algorithms
dictionary/rule matching
field-name heuristics
context-aware detection
external detector/plugin
```

Example:

```text
13 digits alone
```

should not necessarily be enough to classify something as a national identification number.

Validation and contextual information should reduce false positives.

---

# 33. PII Finding Model

A PII finding should contain information similar to:

```json
{
  "findingId": "finding-001",
  "runId": "run-123",
  "attemptId": "attempt-1",
  "source": "JOB_PAYLOAD",
  "fieldPath": "customer.identity",
  "piiType": "NATIONAL_ID",
  "confidence": 0.98,
  "action": "REDACT",
  "detectedAt": "..."
}
```

The raw detected value should not be included by default.

---

# 34. PII Actions

Detection policies may define:

```text
OBSERVE
MASK
REDACT
BLOCK
```

### OBSERVE

Record that PII exists but do not modify execution.

### MASK

Display a masked representation.

Example:

```text
som***@example.com
```

### REDACT

Remove the value from monitoring-visible content.

```text
[REDACTED]
```

### BLOCK

Prevent the job from proceeding.

Blocking must be an explicit policy decision and must not be the default behavior of the monitoring platform.

---

# 35. PII Detection Status

A job should expose PII status separately from its execution state.

Example:

```text
Execution: SUCCEEDED
PII status: DETECTED
```

Possible values:

```text
NOT_SCANNED
SCANNING
CLEAN
DETECTED
SCAN_ERROR
```

A failure of the PII scanner must not automatically mean that the application job failed.

Example:

```text
Job status: SUCCEEDED
PII scan:   SCAN_ERROR
```

unless configuration explicitly requires scanning to succeed.

---

# 36. PII Findings View

Operators must be able to investigate PII detections without displaying raw sensitive values.

Example:

```text
Run: run-84932
Job: export_customer_report

PII findings: 3

1.
Type: EMAIL_ADDRESS
Location: payload.customer.email
Action: REDACT

2.
Type: PHONE_NUMBER
Location: payload.customer.mobile
Action: MASK

3.
Type: NATIONAL_ID
Location: result.customer.identity
Action: REDACT
```

---

# 37. Log Handling

Logs associated with jobs must carry correlation metadata.

At minimum:

```text
run_id
attempt_id
worker_id
job_type
```

Example:

```json
{
  "level": "info",
  "message": "report generation started",
  "runId": "run-123",
  "attemptId": "attempt-1",
  "workerId": "worker-8"
}
```

The monitoring product should encourage structured logs rather than attempting to parse arbitrary text whenever possible.

---

# 38. PII-Safe Logging

Before job logs are persisted into the monitoring platform, configured PII detection/redaction should be applied.

Example:

Input:

```text
Sending report to somchai@example.com
```

Stored monitoring log:

```text
Sending report to [EMAIL_REDACTED]
```

A detection event may additionally record:

```text
Type: EMAIL_ADDRESS
Source: STRUCTURED_LOG
```

---

# 39. Search

Operators must be able to search jobs using fields including:

```text
run_id
job type
queue
status
worker
schedule
creation time
execution time
failure type
PII status
alert type
```

Example queries:

```text
all failed report jobs today

all running jobs older than 10 minutes

jobs executed by worker-07

jobs with PII findings

jobs with national-ID findings

jobs with more than 3 attempts
```

Search must never expose raw detected PII values as a search index unless explicitly configured.

Search and filter state should be representable in a safe, shareable URL where practical so an investigation can be reproduced by another authorized operator. Sensitive values must not be encoded into URLs.

Time range, timezone context, filter state, sort order, and active scope should remain predictable while drilling from aggregate dashboards into detail views.

---

# 40. Job List View

The default job list should include:

| Field    | Description                 |
| -------- | --------------------------- |
| Run ID   | Unique execution identifier |
| Job      | Job type                    |
| Queue    | Queue used                  |
| Status   | Current execution state     |
| Health   | Derived monitoring health   |
| Worker   | Current/latest worker       |
| Attempts | Number of attempts          |
| Created  | Creation time               |
| Duration | Execution duration          |
| PII      | Detection status            |
| Alerts   | Active monitoring alerts    |

---

# 41. Job Detail View

The job detail view should provide:

```text
Job Summary
Execution Timeline
Attempts
Scheduling Information
Worker Information
Queue Information
PII Findings
Alerts and Evidence
Monitoring Completeness / Freshness
Sanitized Logs
Metadata
Correlation / Trace Information
```

Sensitive fields must be masked or redacted according to policy.

The first view of a run should make execution state, monitoring health, latest important event, and data freshness visible without requiring the operator to inspect the complete timeline.

---

# 42. Worker Dashboard

Required worker information:

```text
worker
status
queues
running jobs
capacity
last heartbeat
uptime
completed jobs
failure rate
```

Example:

| Worker   | Status  | Running | Capacity | Last Heartbeat |
| -------- | ------- | ------: | -------: | -------------- |
| worker-1 | Online  |       7 |       10 | 2s ago         |
| worker-2 | Online  |       4 |       10 | 4s ago         |
| worker-3 | Offline |     2\* |       10 | 3m ago         |

The `2*` executions should become candidates for lost-job detection.

---

# 43. Monitoring Dashboard

The main dashboard should answer:

```text
Is the distributed job system healthy?
```

Suggested sections:

```text
Jobs
Queues
Workers
Failures
Scheduling
PII Findings
Active Alerts
```

Example:

```text
Jobs last 24h
──────────────
Succeeded       18,204
Failed             182
Running             45
Waiting            318
Retrying            28

Workers
────────
Healthy              18
Degraded              1
Offline               2

PII
───
Jobs with findings   142

Alerts
──────
Queue backlog          1
Lost runs              3
SLA violations         8
```

The dashboard must also expose whether the monitoring system itself is healthy enough for the displayed information to be trusted.

Recommended monitoring-health signals include:

```text
event ingestion lag
projection lag
Redis queue observation age
alert evaluation lag
PII scan health / lag
known monitoring gaps
```

## 43.1 Dashboard Frontend Experience

The frontend must provide a consistent operator experience across:

```text
Overview
Runs
Queues
Workers
Schedules
Alerts
PII Findings
Monitoring Health
```

### Global Investigation Context

Where applicable, the frontend should expose:

```text
time range
timezone
auto-refresh state
last refresh / freshness
global search
active filters
```

Important investigation views should have stable, shareable URLs that preserve safe state such as time range, queue, job type, execution state, monitoring annotation, worker, alert severity, and sort order.

Raw PII, payload values, log content, tokens, and other sensitive data must never be placed in URLs.

### Freshness and Partial Data

Major views must distinguish:

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

When a refresh fails but previously loaded data remains useful, the frontend should retain the data and mark it stale instead of replacing the entire view with an error.

When one monitoring source is unavailable, the dashboard should preserve usable evidence from other sources and clearly identify the missing or stale subsystem.

### Investigation Navigation

Aggregate abnormalities must support drill-down to evidence.

Example:

```text
Overview
  ↓
Degraded Queue
  ↓
Affected Runs
  ↓
Run Detail
  ↓
Attempt / Timeline / Alert Evidence
```

Navigation should preserve the operator's prior search and filter context where practical.

### Visualization Requirements

Health and severity must not rely on color alone.

Charts and counters must distinguish missing data from a real zero value and make sampling gaps visible where relevant.

### Frontend Security

The frontend may use permissions to control navigation and presentation, but frontend checks are not security boundaries.

Unauthorized raw or masked PII must not be returned by the backend merely so the frontend can hide it.

### Device and Accessibility Scope

The initial dashboard is desktop-first. Primary investigation workflows should support keyboard navigation, visible focus states, semantic status labels, and tables that remain usable without hover-only interactions.

Detailed frontend behavior is defined in RFC-009.

## 43.2 Real-Time Whole-Platform Monitoring

The dashboard must support operational near-real-time observation of the whole project.

Live monitoring should cover:

```text
active job runs and retry chains
worker/delivery attempts
queue depth, oldest age, consumers, and health
worker health and capacity
schedule due/late/missed changes
alerts opening/updating/resolving
PII finding and scan-health summaries
monitoring ingestion/projection health
project component health
live-stream connection and delivery health
```

The dashboard should expose an explicit live state such as:

```text
CONNECTING
LIVE
RECONNECTING
RESYNCING
DEGRADED
POLLING_FALLBACK
STALE
PAUSED
```

A healthy browser connection must not be treated as proof that monitoring data is fresh. The UI must distinguish:

```text
transport freshness
monitoring/projection freshness
source observation freshness
```

Example:

```text
Live connection: LIVE
Projection updated: 1s ago
Redis queue sample: 47s ago — STALE
```

The initial live overview should include project component health for the platform's own runtime processes, such as API, scheduler, Redis delivery, workers, monitoring ingestor/projector, PII scanner, alert evaluator, query API, and real-time gateway.

Real-time updates must be recoverable. After a temporary disconnect the client should resume from a bounded cursor when possible; if continuity cannot be guaranteed, the system must require a fresh snapshot/resynchronization rather than silently skipping changes.

Live delivery is at-least-once. Clients must tolerate duplicate, late, and out-of-order updates. Durable monitoring history remains authoritative.

Detailed real-time semantics are defined in RFC-010.

---

# 44. Alert Framework

Alerts must represent abnormal conditions independently of job state.

Suggested alert types:

```text
RUN_FAILED
RUN_STUCK
RUN_LOST
RUN_TIMEOUT
RUN_RETRY_EXHAUSTED
RUN_DUPLICATE_SUSPECTED

QUEUE_BACKLOG
QUEUE_AGE_HIGH
QUEUE_NO_CONSUMER

WORKER_OFFLINE
WORKER_CAPACITY_HIGH

SCHEDULE_DELAYED
SCHEDULE_MISSED

PII_DETECTED
PII_SCAN_FAILED
PII_POLICY_VIOLATED
```

---

# 45. Alert Lifecycle

Alerts should have:

```text
OPEN
ACKNOWLEDGED
RESOLVED
```

Alerts should store:

```text
alert_id
alert_type
severity
created_at
resolved_at
run_id
worker_id
queue
description
evidence
```

where applicable.

---

# 46. Alert Severity

Suggested severity model:

```text
INFO
WARNING
CRITICAL
```

Severity should be configurable by alert rule.

Example:

```text
PII detected in permitted job payload
→ WARNING

PII detected where policy forbids PII
→ CRITICAL
```

---

# 47. Monitoring Rules

Rules must be configurable by job or queue.

Examples:

```text
report_job:
  expected_duration: 5m
  stuck_after: 10m

email_queue:
  backlog_warning: 1000
  oldest_job_warning: 5m

settlement_job:
  allowed_schedule_delay: 2m
```

Rules should support sensible defaults with per-job overrides.

---

# 48. Metrics

The platform should expose metrics including:

### Job metrics

```text
jobs_created_total
jobs_started_total
jobs_completed_total
jobs_failed_total
jobs_retried_total
```

### Latency

```text
job_queue_latency
job_execution_duration
job_total_duration
job_schedule_drift
```

### Redis / queue

```text
queue_depth
queue_oldest_message_age
queue_consumer_count
queue_processing_rate
```

### Worker

```text
worker_count
worker_active_jobs
worker_capacity
worker_heartbeat_age
```

### PII

```text
pii_findings_total
pii_scan_total
pii_scan_errors_total
```

### Real-time monitoring

```text
realtime_connections
realtime_delivery_lag
realtime_disconnect_total
realtime_resync_required_total
realtime_polling_fallback_total
```

### Project components

```text
component_instances
component_heartbeat_age
component_health
```

Labels must be designed carefully to avoid uncontrolled metric cardinality.

`run_id` must not be used as a general metrics label.

---

# 49. Tracing and Correlation

Where distributed tracing is available, a job should preserve trace correlation.

Example:

```text
HTTP request
    │
    trace_id = abc
    ▼
Create background job
    │
    run_id = xyz
    ▼
Redis
    │
    ▼
Worker
```

The relationship:

```text
trace_id ↔ run_id
```

should be available for investigation.

Monitoring must remain usable even when distributed tracing is not installed.

---

# 50. Data Persistence

Operational job history must be stored independently from Redis.

Conceptual persistent entities include:

```text
job_definitions
job_runs
job_attempts
job_events
workers
schedules
pii_findings
alerts
```

Redis should not be treated as the sole source for historical observability.

---

# 51. Event Retention

Lifecycle events should be append-oriented.

Instead of repeatedly rewriting:

```text
status = RUNNING
```

the system should be able to retain:

```text
run-101 run.created
run-101 run.queued
run-101 attempt.claimed
run-101 attempt.started
run-101 attempt.failed
run-101 run.failed
run-102 run.retry_scheduled parent=run-101
run-102 run.queued
run-102 attempt.started
run-102 run.succeeded
```

A current-state projection may be derived for efficient querying.

---

# 52. Redis Failure Handling

The monitoring model must account for Redis becoming temporarily unavailable.

The product must clearly distinguish:

```text
job failure
worker failure
Redis failure
monitoring failure
```

An infrastructure outage must not automatically be represented as an application failure.

The system should recover its monitoring view after connectivity returns where possible.

---

# 53. Monitoring-System Failure

The execution platform should not necessarily stop processing jobs simply because the monitoring platform becomes temporarily unavailable.

Where appropriate:

```text
Worker
 ├── execute business job
 └── emit monitoring event
```

Monitoring event delivery may use buffering or later reconciliation.

The product should make monitoring gaps visible instead of silently pretending that complete data exists.

The real-time channel is observational and must not become a job-execution dependency. If the live gateway or fanout transport is unavailable, the dashboard should fall back to bounded polling where possible and retain the last successful snapshot with a visible stale/degraded state.

---

# 54. Clock Considerations

Because workers operate on different machines, event timestamps may have small clock differences.

The product should not assume perfect clock synchronization.

Where ordering is important, the design should consider:

```text
event sequence
Redis ordering
server-side timestamp
worker timestamp
```

rather than relying only on worker wall clocks.

---

# 55. Security Requirements

Access to monitoring information must be protected.

The system must support authorization boundaries for capabilities such as:

```text
view jobs
view job metadata
view logs
view PII findings
view masked content
manage monitoring rules
manage PII policies
```

Raw sensitive values should require stronger authorization if raw-value storage is enabled at all.

---

# 56. PII Data-Minimization Requirements

The monitoring system should retain only information required to explain a finding.

Preferred:

```text
type = NATIONAL_ID
field = payload.customer.identity
confidence = 0.98
```

Avoid:

```text
raw_value = 1234567890123
```

Possible fingerprinting may be used where correlation is required without storing the actual value.

Example:

```text
value_fingerprint = HMAC(value)
```

The hashing/fingerprinting scheme must be designed so that common PII values cannot easily be recovered through dictionary attacks.

---

# 57. Configuration

Configuration should support scopes such as:

```text
Global
Queue
Job Type
Schedule
```

Precedence may conceptually follow:

```text
job-specific
    overrides
queue-specific
    overrides
global default
```

Configuration areas include:

```text
timeouts
stuck thresholds
schedule delay thresholds
worker heartbeat thresholds
PII detectors
PII actions
retention
alerting
real-time replay retention
live delivery latency thresholds
component heartbeat thresholds
polling fallback intervals
```

---

# 58. API Requirements

The product should expose APIs covering at least:

```text
Jobs
Runs
Attempts
Schedules
Workers
Queues
Alerts
PII Findings
Monitoring Rules
PII Policies
Project Components
Real-Time Observation
```

Illustrative resources:

```text
POST /jobs

GET /runs
GET /runs/{runId}
GET /runs/{runId}/events
GET /runs/{runId}/attempts

GET /queues
GET /queues/{queue}

GET /workers
GET /workers/{workerId}

GET /alerts

GET /pii/findings

GET /monitoring/health
GET /components
GET /components/{componentInstanceId}

GET /live  # streaming endpoint; exact transport/path defined by RFC-010
```

These resource names are illustrative rather than mandatory implementation contracts.

The dashboard frontend must consume supported query APIs rather than reading Redis or PostgreSQL directly. Backend APIs are responsible for authorization, PII-safe serialization, bounded filtering, pagination, aggregate calculations, and freshness metadata required by operator views.

Dashboard-oriented APIs should support bounded aggregate and health queries without requiring the browser to download high-cardinality run/event datasets and aggregate them locally.

Live-capable snapshot responses should expose an observation watermark/cursor that allows the dashboard to subscribe to subsequent changes without a silent handoff gap. Real-time subscriptions must be authenticated, authorization-aware, PII-safe, resumable within a bounded replay window, and able to force snapshot resynchronization when continuity cannot be guaranteed.

---

# 59. Runtime Interaction Model

A possible execution interaction is:

```text
Client
  │
  │ create job
  ▼
Job API
  │
  ├──── persist run
  │
  ▼
Redis
  │
  │ deliver
  ▼
Worker
  │
  ├──── job.claimed
  ├──── job.started
  ├──── heartbeat
  │
  ├──── execute
  │
  ├──── PII scan
  │
  └──── job.completed
             │
             ▼
      Monitoring Store
             │
             ▼
          Dashboard
```

The final implementation may use different boundaries as long as the observable behavior remains equivalent.

---

# 60. Example End-to-End Scenario

A producer requests:

```text
Generate monthly customer statement
```

The system creates:

```text
run_id = run-901
```

Timeline:

```text
10:00:00 job.created

10:00:00 payload scan starts

10:00:00 PII detected:
         EMAIL_ADDRESS
         payload.customer.email

10:00:00 job.queued
         queue=reports

10:00:01 worker-07 claims job

10:00:01 job.started

10:00:11 heartbeat

10:00:21 heartbeat

10:00:31 heartbeat

10:00:45 execution error

10:00:45 job.failed
         attempt=1

10:00:45 retry scheduled
         delay=5m

10:05:45 job.queued

10:05:46 worker-12 claims job

10:05:46 job.started
         attempt=2

10:06:13 job.completed
```

The UI should summarize this as:

```text
Status:
SUCCEEDED

Attempts:
2

Duration:
6m13s total

Execution:
27s final attempt

Workers:
worker-07
worker-12

PII:
1 finding

Alerts:
Previous failure
```

---

# 61. Abnormal Scenario — Worker Crash

```text
Worker A claims run-300
       │
       ▼
RUNNING
       │
       ▼
Worker A crashes
       │
       ├── heartbeat disappears
       │
       └── execution heartbeat disappears
```

Monitoring should eventually show:

```text
run-300

State:
RUNNING

Health:
LOST

Last worker:
worker-a

Last heartbeat:
2 minutes ago

Redis:
message remains pending
```

This distinction is important.

The execution record should not incorrectly become:

```text
FAILED
```

unless failure can actually be established.

---

# 62. Abnormal Scenario — Redis Backlog

```text
report queue

12:00
queue depth = 100

12:05
queue depth = 600

12:10
queue depth = 1,500

workers = 2
worker utilization = 100%
```

The system should raise:

```text
QUEUE_BACKLOG
```

and provide evidence:

```text
Enqueue rate: 80 jobs/min
Completion rate: 25 jobs/min
Oldest pending job: 14m
```

---

# 63. Abnormal Scenario — PII Leakage in Error

Worker produces:

```text
failed processing customer national ID 1234567890123
```

The monitoring pipeline detects sensitive content before persistence.

Stored log:

```text
failed processing customer national ID [REDACTED]
```

Finding:

```text
Type:
NATIONAL_ID

Source:
ERROR_MESSAGE

Path:
error.message

Run:
run-773
```

---

# 64. Product Acceptance Criteria

The initial product is considered functionally complete when an operator can:

- observe jobs across multiple independent workers
- determine where a job is in its lifecycle
- inspect every retry run in an execution chain
- trace each retry through `parent_run_id` and `execution_chain_id`
- inspect worker/delivery attempts within each run
- identify the worker responsible for each attempt
- identify jobs waiting in Redis
- inspect queue health
- detect queue backlog
- identify workers that have stopped heartbeating
- distinguish stuck jobs from potentially lost jobs
- observe schedule delay
- identify missed scheduled runs
- investigate execution failures
- search historical runs
- inspect a complete job timeline
- identify PII findings without exposing raw values
- distinguish PII scan failures from business-job failures
- trace a PII finding back to the corresponding job and field
- correlate jobs with distributed traces where tracing is available
- determine from the overview dashboard whether job execution and the monitoring pipeline appear healthy
- drill from an unhealthy queue, worker, schedule, alert, or PII aggregate to affected runs and evidence
- distinguish execution state from monitoring health in the UI
- see when dashboard data is stale, partial, or degraded instead of assuming it is current
- preserve safe investigation filters and time range during drill-down and sharing
- share a deep link that reproduces an authorized investigation without embedding sensitive values
- observe important run, queue, worker, schedule, alert, PII-summary, and monitoring-health changes without manual refresh
- see whether the dashboard is LIVE, reconnecting, resynchronizing, using polling fallback, or stale
- recover safely after a temporary live connection interruption without silently losing state changes
- inspect the health of the project's own runtime components
- distinguish a healthy live transport from stale monitoring/source observations
- use critical dashboard workflows without relying only on color

---

# 65. Suggested Product Phases

## Phase 1 — Job Execution Foundation

Support:

```text
Redis job delivery
multiple independent workers
run identity
attempt identity
basic lifecycle events
durable monitoring history
```

Goal:

Understand distributed execution.

---

## Phase 2 — Job Monitoring

Introduce:

```text
dashboard shell and navigation
system-health overview
job list
job detail
timeline
worker tracking
queue monitoring
execution duration
queue latency
global time range and filters
shareable investigation URLs
freshness / partial-data states
basic SSE live updates for active runs, workers, queues, and monitoring health
live connection state and polling fallback
```

Goal:

Make distributed job execution understandable.

---

## Phase 3 — Failure Intelligence

Introduce:

```text
retry visibility
worker heartbeat
stuck detection
lost detection
timeout detection
queue backlog detection
```

Goal:

Move from observability to operational diagnosis.

---

## Phase 4 — Scheduling Monitoring

Introduce:

```text
scheduled jobs
recurring jobs
schedule drift
missed schedule detection
schedule history
```

Goal:

Understand whether jobs execute when expected.

---

## Phase 5 — PII Detection

Introduce:

```text
payload scanning
result scanning
error scanning
log scanning
PII findings
masking
redaction
configurable policies
```

Goal:

Detect sensitive information flowing through background processing.

---

## Phase 6 — Operational Intelligence

Introduce:

```text
alert framework
monitoring rules
historical trends
queue-health indicators
worker-health indicators
cross-job analysis
monitoring-system health dashboard
saved investigation views where justified
whole-project component health
real-time activity feed
live replay/resume and resynchronization hardening
real-time backpressure/coalescing
real-time monitoring SLOs
```

Goal:

Allow operators to identify emerging issues before individual jobs fail.

---

# 66. Core Product Boundary

A useful boundary for the project is:

```text
              ┌─────────────────────┐
              │      Scheduler      │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │        Redis        │
              └──────────┬──────────┘
                         │
               ┌─────────┼─────────┐
               ▼         ▼         ▼
            Worker A  Worker B  Worker C
               │         │         │
               └─────────┼─────────┘
                         │
                         ▼
             ┌──────────────────────┐
             │ Monitoring Platform  │
             │                      │
             │ Job lifecycle        │
             │ Queue health         │
             │ Worker health        │
             │ Schedule health      │
             │ Failure detection    │
             │ PII detection        │
             │ Alerts               │
             │ Component health      │
             │ Real-time observation │
             └──────────┬───────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │ Query / Investigation│
             │ API + Read Models    │
             └──────────┬───────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │ Dashboard Frontend   │
             │ Overview + Drilldown │
             │ Freshness + Evidence │
             └──────────────────────┘
```

The scheduler and workers exist so that the product has a realistic distributed system to observe.

The **main engineering problem** is reconstructing reliable operational truth from many independently generated signals.

---
