# RFC-006: PII Detection Context

- **Status:** Draft
- **Context:** PII Detection
- **Depends On:** RFC-000, RFC-005
- **Consumers:** Monitoring, Alerting, Query & Investigation

## 1. Summary

This context detects and classifies personally identifiable information flowing through distributed job processing.

The primary principle is:

> Detect PII without making the monitoring platform a new PII repository.

## 2. Goals

- detect configured PII categories,
- scan multiple job-related sources,
- record safe findings,
- support confidence and detector evidence,
- support mask/redact/block policies,
- separate scan failure from job failure,
- make PII findings searchable without indexing raw PII values.

## 3. Non-Goals

- full enterprise DLP,
- identity verification,
- broad document classification,
- storing raw PII by default,
- deciding job health.

## 4. Scan Sources

Minimum supported locations:

```text
JOB_PAYLOAD
JOB_METADATA
JOB_RESULT
ERROR_MESSAGE
STRUCTURED_LOG
```

Each finding must identify its source.

## 5. PII Categories

Initial categories may include:

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

Detectors should be extensible by locale.

Thai deployments should be able to add detectors for locally relevant formats such as Thai national identification numbers and Thai phone numbers.

## 6. Detector Strategies

A detector may combine:

```text
regex/pattern
checksum/validation
field-name heuristic
context heuristic
dictionary
statistical/model-based detector
external plugin
```

A match is not automatically a finding if validation indicates it is impossible.

## 7. Scan Model

```text
Content
   │
   ▼
Normalizer
   │
   ▼
Detector Set
   │
   ▼
Candidate Findings
   │
   ▼
Policy Evaluation
   │
 ┌─┼─────────┬───────┐
 ▼ ▼         ▼       ▼
OBSERVE     MASK    REDACT   BLOCK
```

## 8. Finding Model

A safe finding should include:

```text
finding_id
run_id
attempt_id?
source
field_path?
pii_type
confidence
detector_id
policy_action
detected_at
value_fingerprint?
```

Raw PII is excluded by default.

## 9. Raw Value Handling

Default:

```text
store_raw_value = false
```

If correlation across findings is required, a keyed fingerprint may be used:

```text
HMAC(secret, normalized_value)
```

A plain unsalted hash is discouraged because common values may be dictionary attacked.

## 10. Scan Status

Per scan target:

```text
NOT_SCANNED
SCANNING
CLEAN
DETECTED
SCAN_ERROR
```

Important:

```text
Job execution: SUCCEEDED
PII scan:      SCAN_ERROR
```

is valid.

PII scan failure does not automatically turn the job into `FAILED`.

## 11. Policy Actions

### OBSERVE

Record finding, leave data unchanged.

### MASK

Return/store only a partial representation.

### REDACT

Replace sensitive content with a marker.

### BLOCK

Prevent execution or output propagation.

`BLOCK` must be explicit because it changes business execution semantics.

## 12. Pre-Execution Scanning

For payload scanning, an implementation may scan:

- before publishing to Redis,
- before worker execution,
- or both.

The chosen boundary affects whether raw PII enters Redis.

The architecture must document this explicitly.

## 13. Post-Execution Scanning

Results, errors and logs may be scanned before monitoring persistence.

Preferred sequence:

```text
worker output
   │
   ▼
PII scan/redaction
   │
   ▼
monitoring persistence
```

This reduces accidental sensitive-data retention.

## 14. PII-Safe Logging

Given:

```text
failed to process 1234567890123
```

the monitoring copy should become:

```text
failed to process [NATIONAL_ID_REDACTED]
```

and create a finding without persisting the raw number.

## 15. Search

Searchable dimensions:

```text
pii_type
source
job_type
queue
run_id
time range
policy_action
scan_status
confidence range
```

Raw PII value lookup is out of scope by default.

## 16. False Positives and False Negatives

The system must expose detector metadata and confidence where applicable.

Operators should be able to understand:

- which detector matched,
- what source/path matched,
- why a policy action occurred.

The product should support future feedback mechanisms without storing raw values unnecessarily.

## 17. Security

PII findings require stricter authorization than general job health.

Possible permissions:

```text
pii.findings.list
pii.findings.view
pii.policy.manage
pii.masked_value.view
pii.raw_value.view   # only if raw storage is enabled
```

## 18. Events

Suggested events:

```text
pii.scan_started
pii.scan_completed
pii.scan_failed
pii.detected
pii.policy_applied
```

Example safe event:

```json
{
  "event_type": "pii.detected",
  "run_id": "run-123",
  "source": "JOB_PAYLOAD",
  "field_path": "customer.email",
  "pii_type": "EMAIL_ADDRESS",
  "confidence": 0.99,
  "policy_action": "REDACT"
}
```

## 19. Open Questions

1. Which PII types are mandatory for the first release?
2. Should scanning be synchronous or asynchronous per source?
3. Which sources can block execution?
4. How are nested/binary payloads handled?
5. Will model-based detection be allowed in the first version?
6. How are detector versions recorded for reproducibility?
7. What is the retention policy for PII findings and fingerprints?
