# RFC-006: PII Policy Engine

- **Status:** Draft
- **Context:** PII Policy Engine
- **Depends On:** RFC-000, RFC-005
- **Consumers:** Job API, Worker Runtime, Monitoring, Alerting, Query & Investigation, Dashboard Frontend, Real-Time Monitoring

## 1. Summary

This context provides a configurable policy engine for detecting, classifying, masking, redacting, observing, and optionally blocking PII flowing through distributed job processing.

The engine is configured through a versioned JSON policy document rather than requiring code changes for every new pattern or masking rule.

The primary principles are:

> Detect and transform PII through explicit, reproducible policy without making the monitoring platform a new PII repository.

> Detection answers **what matched**. Policy evaluation answers **what should happen**.

## 2. Goals

- define PII behavior through JSON configuration,
- support built-in and custom pattern detectors,
- support configurable masking/redaction behavior,
- scope rules by source, job type, queue, field path and PII type,
- support deterministic rule precedence,
- validate and compile configuration before activation,
- version every active policy for historical reproducibility,
- support safe runtime policy reload,
- support observe-only/dry-run rollout,
- record safe findings without requiring raw-value persistence,
- expose policy health and active version to monitoring,
- separate PII scan failure from job execution failure.

## 3. Non-Goals

- full enterprise DLP,
- identity verification,
- arbitrary code execution inside policy rules,
- unbounded scripting inside detectors or maskers,
- storing raw PII by default,
- using browser-side masking as a security boundary,
- deciding general job health.

## 4. Policy Document

A policy is an immutable versioned JSON document after activation.

Conceptual shape:

```json
{
  "apiVersion": "monitoring.job/v1alpha1",
  "kind": "PIIPolicy",
  "metadata": {
    "name": "default",
    "version": 7
  },
  "spec": {
    "evaluationMode": "FIRST_MATCH",
    "defaults": {
      "action": "OBSERVE",
      "onScanError": "FAIL_OPEN"
    },
    "detectors": [],
    "rules": []
  }
}
```

The policy must have a stable content checksum in addition to its human-facing version.

## 5. Detector Definitions

Detector configuration defines **how candidates are recognized**.

Supported detector families should include:

```text
BUILTIN
REGEX
FIELD_NAME
COMPOSITE
PLUGIN       # future/optional
MODEL        # future/optional
```

A detector definition may contain:

```text
id
piiType
type
enabled
pattern?
flags?
validator?
minimumConfidence?
normalization?
metadata?
```

Example custom pattern:

```json
{
  "id": "thai-phone-custom",
  "type": "REGEX",
  "piiType": "PHONE_NUMBER",
  "enabled": true,
  "pattern": "(?:\\+66|0)[689]\\d{8}",
  "minimumConfidence": 0.85
}
```

Custom regex must use a safe bounded-time regex implementation such as RE2 semantics. Arbitrary backtracking expressions must not be accepted.

## 6. Built-In Detectors

Initial built-in types may include:

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

Built-ins may combine:

```text
pattern
checksum validation
field-name evidence
context evidence
locale-specific validation
```

Thai deployments should support locally relevant formats such as Thai national identification numbers and Thai phone numbers.

## 7. Detector vs Policy Rule

The following must remain separate:

```text
Detector
  "This value appears to be NATIONAL_ID"

Policy Rule
  "For NATIONAL_ID in JOB_RESULT, redact it"
```

This allows the same detector to have different behavior by source or job type.

## 8. Scan Sources

Minimum supported sources:

```text
JOB_PAYLOAD
JOB_METADATA
JOB_RESULT
ERROR_MESSAGE
STRUCTURED_LOG
```

Each finding must identify its scan source.

## 9. Policy Match Conditions

A rule may match on one or more dimensions:

```text
sources
jobTypes
queues
fieldPaths
piiTypes
detectorIds
minimumConfidence
labels/tags
```

Example:

```json
{
  "id": "redact-national-id-in-results",
  "priority": 900,
  "match": {
    "sources": ["JOB_RESULT", "STRUCTURED_LOG"],
    "piiTypes": ["NATIONAL_ID"]
  },
  "action": {
    "type": "REDACT",
    "replacement": "[NATIONAL_ID_REDACTED]"
  }
}
```

## 10. Field Path Matching

Structured JSON payloads should support field-path scoping.

A bounded JSONPath-like subset is preferred, for example:

```text
$.customer.email
$.customers[*].phone
$.payment.cardNumber
```

The supported syntax must be explicitly documented. Policy evaluation must not require arbitrary user-provided expressions or executable scripts.

## 11. Actions

Supported policy actions:

```text
OBSERVE
MASK
REDACT
BLOCK
IGNORE
```

### OBSERVE

Record a safe finding and do not modify content.

### MASK

Transform a matched value according to a masking configuration.

### REDACT

Replace the matched value with a fixed safe replacement.

### BLOCK

Prevent the protected content from entering the next execution/persistence boundary.

`BLOCK` must be explicit because it can change business execution semantics.

### IGNORE

Suppress a detector match intentionally. This supports known-safe fields or expected false positives without disabling the detector globally.

## 12. Masking Strategies

The engine should support reusable masking strategies without requiring code changes per field.

Minimum useful strategies:

```text
FULL
KEEP_PREFIX
KEEP_SUFFIX
KEEP_PREFIX_SUFFIX
PRESERVE_FORMAT
EMAIL
FIXED
```

Example:

```json
{
  "type": "MASK",
  "mask": {
    "strategy": "KEEP_SUFFIX",
    "visibleCharacters": 4,
    "maskCharacter": "*"
  }
}
```

Input:

```text
1234567890123
```

Output:

```text
*********0123
```

Masking must never be considered equivalent to encryption.

## 13. Pattern-Specific Transformation

A rule may override the default transformation for a specific detector or field.

Example email masking:

```json
{
  "id": "mask-email-in-metadata",
  "priority": 500,
  "match": {
    "sources": ["JOB_METADATA"],
    "piiTypes": ["EMAIL_ADDRESS"]
  },
  "action": {
    "type": "MASK",
    "mask": {
      "strategy": "EMAIL",
      "localVisiblePrefix": 2,
      "domainMode": "PRESERVE"
    }
  }
}
```

Example result:

```text
john.smith@example.com
→ jo******@example.com
```

## 14. Rule Precedence

Policy evaluation must be deterministic.

Recommended ordering:

1. higher numeric `priority`,
2. more specific scope,
3. stable rule identifier as final deterministic tie-breaker.

For the first version, use:

```text
evaluationMode = FIRST_MATCH
```

Once a rule applies to a candidate, no lower-priority rule changes the action.

A future `MOST_RESTRICTIVE` mode may be considered, but combining transformations should not be implicit.

## 15. Default Behavior

Every policy must define defaults for unmatched findings and scan errors.

Example:

```json
{
  "defaults": {
    "action": "OBSERVE",
    "onScanError": "FAIL_OPEN"
  }
}
```

Supported scan-error behavior:

```text
FAIL_OPEN
FAIL_CLOSED
```

`FAIL_CLOSED` should only be accepted for scan points where enforcement is explicitly supported.

## 16. Scan Pipeline

```text
Content
   │
   ▼
Normalizer
   │
   ▼
Detector Registry
   │
   ▼
Candidate Findings
   │
   ▼
Policy Matcher
   │
   ▼
Action Executor
   │
   ├── OBSERVE
   ├── MASK
   ├── REDACT
   ├── BLOCK
   └── IGNORE
   │
   ▼
Safe Findings + Safe Output
```

## 17. Finding Model

A finding should include enough policy evidence to reproduce the decision:

```text
finding_id
run_id
attempt_id?
source
field_path?
pii_type
confidence
detector_id
detector_version
policy_name
policy_version
policy_checksum
rule_id
policy_action
mask_strategy?
detected_at
value_fingerprint?
```

Raw PII is excluded by default.

## 18. Raw Value Handling

Default:

```text
store_raw_value = false
```

If correlation across findings is required:

```text
HMAC(versioned_secret_key, normalized_value)
```

A plain unsalted hash is discouraged.

The finding should store `fingerprint_key_version` when fingerprints are enabled.

## 19. Policy Lifecycle

A policy lifecycle should be explicit:

```text
DRAFT
  ↓ validate
VALIDATED
  ↓ activate
ACTIVE
  ↓ replace
SUPERSEDED
```

Invalid policy documents must never become active.

Activation should be atomic: scanners observe either the complete old version or the complete new version, never a partial configuration.

## 20. Policy Validation

Before activation, validate at least:

- JSON/schema validity,
- unique detector IDs,
- unique rule IDs,
- supported PII types,
- safe regex compilation,
- field-path syntax,
- masking parameter constraints,
- rule references to known detector IDs,
- supported source/action combinations,
- impossible or contradictory values,
- policy size and detector-count limits.

## 21. Policy Test / Dry Run

The engine should support validation against test inputs before activation.

Useful operations:

```text
validate policy
compile policy
test policy against sample input
compare candidate policy with active policy
activate policy
rollback to previous policy version
```

Dry run must not persist supplied raw test content unless explicitly designed for a protected test environment.

## 22. Runtime Reload

Policy changes should not require process restart.

Expected behavior:

```text
new policy uploaded
      ↓
validate + compile
      ↓
store immutable revision
      ↓
activate pointer atomically
      ↓
scanner instances receive/invalidate cache
      ↓
new scans use new revision
```

In-flight scans should finish under the policy revision they started with.

## 23. Pre-Execution Scanning

For payloads, the engine may scan:

- before durable job creation,
- before publishing to Redis,
- before worker execution,
- or at multiple explicitly configured boundaries.

The chosen scan point determines whether raw PII can enter Redis.

A `BLOCK` or transformation policy must identify the boundary it protects.

## 24. Post-Execution Scanning

Preferred sequence:

```text
worker output/error/log
      ↓
PII engine
      ↓
policy action
      ↓
safe monitoring persistence
```

This reduces accidental sensitive-data retention.

## 25. Structured JSON Processing

For JSON sources:

1. parse the tree,
2. traverse scalar fields,
3. preserve safe field path,
4. evaluate field-name detectors,
5. evaluate value detectors,
6. resolve candidates,
7. evaluate rules,
8. transform a reconstructed safe copy when required.

Binary/unparseable content requires an explicit policy rather than silent best-effort interpretation.

## 26. Text Processing

For unstructured text:

- normalize Unicode according to configured detector requirements,
- execute bounded-time detectors,
- validate candidates,
- resolve overlapping matches deterministically,
- apply transformations from end-to-start by offsets.

## 27. Overlapping Matches

If multiple detectors match overlapping text, the engine must not apply inconsistent transformations.

Recommended resolution order:

1. higher policy priority,
2. higher detector confidence,
3. longer match,
4. stable detector ID tie-breaker.

The exact candidate resolution must be observable in test mode.

## 28. Scan Status

Per scan target:

```text
NOT_SCANNED
SCANNING
CLEAN
DETECTED
TRANSFORMED
BLOCKED
SCAN_ERROR
```

Example:

```text
Job execution: SUCCEEDED
PII scan:      SCAN_ERROR
```

is valid.

PII scan failure does not automatically make the run `FAILED` unless enforcement at that scan boundary is explicitly fail-closed.

## 29. Search and Investigation

Searchable dimensions should include:

```text
pii_type
source
job_type
queue
run_id
execution_chain_id
time range
policy_action
policy_name
policy_version
rule_id
detector_id
scan_status
confidence range
```

Raw PII lookup is out of scope by default.

## 30. Policy Observability

Monitoring must expose at least:

```text
active policy name/version/checksum
policy reload status
policy compile failures
scanner instances by policy version
scan latency
scan error rate
findings by type/action
blocked count
masked/redacted count
policy evaluation latency
rule match count
```

A scanner instance running an unexpectedly old policy revision should be visible as configuration drift.

## 31. Dashboard Requirements

Authorized operators should be able to inspect:

- active policy version and status,
- policy revision history,
- safe rule summaries,
- detector inventory,
- validation/compile failures,
- finding counts by rule/action/type,
- scanner policy-version drift.

If policy editing is later exposed in the dashboard, write access must be separately authorized and audited.

## 32. Real-Time Events

Suggested safe events:

```text
pii.scan_started
pii.scan_completed
pii.scan_failed
pii.detected
pii.policy_applied
pii.policy_validated
pii.policy_validation_failed
pii.policy_activated
pii.policy_reload_failed
pii.policy_drift_detected
```

Example:

```json
{
  "event_type": "pii.detected",
  "run_id": "run-123",
  "source": "JOB_PAYLOAD",
  "field_path": "$.customer.email",
  "pii_type": "EMAIL_ADDRESS",
  "confidence": 0.99,
  "detector_id": "builtin-email",
  "policy_name": "default",
  "policy_version": 7,
  "rule_id": "mask-email-payload",
  "policy_action": "MASK"
}
```

Raw or reconstructable matched values must not be included in general monitoring events.

## 33. Security

Possible permissions:

```text
pii.findings.list
pii.findings.view
pii.policy.read
pii.policy.validate
pii.policy.manage
pii.policy.activate
pii.masked_value.view
pii.raw_value.view   # only if explicitly supported
```

Policy mutations must be audited with actor, revision, checksum and activation result.

Secrets such as HMAC keys must not be embedded in the JSON policy. The policy may reference a secret/key identifier managed outside the document.

## 34. Example Policy

A complete example is provided at:

```text
rfc/examples/pii-policy.example.json
```

A machine-readable validation schema is provided at:

```text
rfc/schemas/pii-policy.schema.json
```

## 35. Open Questions

1. Which built-in PII types are mandatory for the first release?
2. Which policy store owns immutable revisions and the active pointer?
3. Is `FIRST_MATCH` sufficient for v1, or is a restrictive merge mode required?
4. Which scan sources permit `BLOCK` and `FAIL_CLOSED`?
5. Which JSONPath subset will be supported?
6. Should custom detector definitions support only regex initially?
7. Will policy management be API/config-file only or dashboard-editable?
8. What is the maximum allowed policy size, rule count and detector count?
9. How are policy rollbacks approved and audited?
10. What is the retention policy for findings and fingerprints?
