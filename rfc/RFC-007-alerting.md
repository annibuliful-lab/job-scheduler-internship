# RFC-007: Alerting Context

- **Status:** Draft
- **Context:** Alerting
- **Depends On:** RFC-005, RFC-006
- **Consumers:** Query & Investigation, external notification adapters

## 1. Summary

This context converts monitoring conditions into manageable alert instances.

The Monitoring context determines facts and derived health signals. Alerting determines whether those signals require operator attention.

## 2. Goals

- configurable rules,
- alert deduplication,
- severity,
- alert lifecycle,
- acknowledgement,
- resolution,
- evidence linking,
- notification integration without coupling core alert state to one channel.

## 3. Non-Goals

- calculating raw queue depth,
- owning worker heartbeat,
- executing retries,
- performing PII detection,
- replacing an enterprise incident-management platform.

## 4. Alert Types

Initial categories:

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

MONITORING_GAP
```

## 5. Severity

Initial levels:

```text
INFO
WARNING
CRITICAL
```

Severity is policy, not intrinsic truth.

Example:

```text
PII_DETECTED in permitted payload → WARNING
PII_POLICY_VIOLATED              → CRITICAL
```

## 6. Alert Lifecycle

```text
OPEN
  │
  ├──► ACKNOWLEDGED
  │        │
  │        ▼
  └────► RESOLVED
```

A resolved condition may later create a new alert instance if it recurs, depending on deduplication/window rules.

## 7. Alert Model

```text
alert_id
alert_type
severity
status
opened_at
acknowledged_at?
resolved_at?
subject_type
subject_id
rule_id
rule_version
evidence_ref
summary
```

Possible subjects:

```text
RUN
ATTEMPT
QUEUE
WORKER
SCHEDULE
PII_FINDING
PLATFORM
```

## 8. Rule Model

Examples:

```text
if queue.oldest_age > 5m for 3m
then QUEUE_AGE_HIGH severity WARNING
```

```text
if worker.heartbeat_age > 60s
then WORKER_OFFLINE severity CRITICAL
```

```text
if pii.policy_action = BLOCK
then PII_POLICY_VIOLATED severity CRITICAL
```

Rules may scope by:

```text
global
queue
job type
schedule
PII category
```

## 9. Deduplication

Repeated evaluation must not create an unlimited number of identical active alerts.

A deduplication key may be conceptually:

```text
rule_id + subject_type + subject_id
```

The design must define what happens when severity changes while an alert is open.

## 10. Evidence

Every alert should explain why it exists.

Example:

```text
Alert: RUN_LOST
run_id: run-123

Evidence:
- attempt state = RUNNING
- last execution heartbeat = 10:01:10
- worker last heartbeat = 10:01:12
- evaluated at = 10:04:00
- threshold = 60s
```

The alert must not merely say "lost".

## 11. Resolution

Resolution should normally occur because the underlying condition is no longer true.

Example:

```text
WORKER_OFFLINE
```

may resolve when a heartbeat returns.

A historical resolution should not erase the previous alert.

## 12. Notification Adapters

Potential channels:

```text
email
webhook
Slack/Teams
PagerDuty-like integration
```

Alert state must survive notification delivery failure.

Example:

```text
alert.status = OPEN
notification.status = FAILED
```

## 13. PII Safety

Alert summaries must not include raw PII.

Use:

```text
PII detected: NATIONAL_ID
source: JOB_PAYLOAD
field: customer.identity
```

not the detected value.

## 14. Events

Suggested events:

```text
alert.opened
alert.updated
alert.acknowledged
alert.resolved
alert.notification_requested
alert.notification_failed
alert.notification_sent
```

## 15. Open Questions

1. Should acknowledgement require user identity?
2. Are alert silences/muting part of the first release?
3. How are flapping conditions handled?
4. Are notification channels implemented internally or through external tooling?
5. Should rules be declarative configuration, code, or both?
6. Is alert escalation needed initially?
