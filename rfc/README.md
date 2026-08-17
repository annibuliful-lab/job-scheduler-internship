# Distributed Job Scheduling Monitoring — RFC Set

## Purpose

This RFC set defines the domain boundaries for a monitoring-first distributed job scheduling platform using Redis as the runtime delivery/coordination substrate and PII detection as a first-class monitoring capability.

The documents intentionally define **domain ownership, contracts, invariants, and failure semantics** without forcing a one-domain-per-service architecture.

A bounded context may be implemented as:
- a module in a modular monolith,
- an independently deployable service,
- multiple internal components,
- or a mixture of the above.

The implementation architecture should be justified separately.

## Product Boundary

```text
                    ┌────────────────────┐
                    │ Scheduling Context │
                    └─────────┬──────────┘
                              │ creates due runs
                              ▼
                    ┌────────────────────┐
                    │ Job Run Context    │
                    └─────────┬──────────┘
                              │ dispatch request
                              ▼
                    ┌────────────────────┐
                    │ Redis Delivery     │
                    │ Context            │
                    └─────────┬──────────┘
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              Worker A     Worker B     Worker C
                 │            │            │
                 └────────────┼────────────┘
                              │ lifecycle signals
                              ▼
                    ┌────────────────────┐
                    │ Monitoring Context │
                    └──────┬───────┬─────┘
                           │       │
                    ┌──────▼───┐   ▼
                    │ Alerting │  PII Detection
                    │ Context  │  Context
                    └──────────┘
```

## RFC Index

| RFC | Context | Primary Responsibility |
|---|---|---|
| [RFC-000](RFC-000-domain-boundaries.md) | Domain Boundaries | Shared language, context ownership, integration rules |
| [RFC-001](RFC-001-job-run-lifecycle.md) | Job Run Lifecycle | Logical runs, attempts, state transitions, execution identity |
| [RFC-002](RFC-002-scheduling.md) | Scheduling | One-time and recurring schedule definitions and due-run creation |
| [RFC-003](RFC-003-redis-delivery.md) | Redis Delivery | Dispatch, claiming, acknowledgement and recovery semantics |
| [RFC-004](RFC-004-worker-runtime.md) | Worker Runtime | Worker identity, capacity, heartbeats and execution protocol |
| [RFC-005](RFC-005-monitoring-observability.md) | Monitoring & Observability | Durable lifecycle events, projections, timelines and health interpretation |
| [RFC-006](RFC-006-pii-detection.md) | PII Detection | Detection, classification, masking/redaction and findings |
| [RFC-007](RFC-007-alerting.md) | Alerting | Rules, incidents/alerts, severity and lifecycle |
| [RFC-008](RFC-008-query-and-investigation.md) | Query & Investigation | Search, investigation APIs, access-safe read models and evidence composition |
| [RFC-009](RFC-009-dashboard-frontend.md) | Dashboard Frontend | Operator navigation, visualization, drill-down, freshness and safe frontend behavior |
| [RFC-010](RFC-010-real-time-monitoring.md) | Real-Time Monitoring | Low-latency whole-project observation, replay/resume, component health and live delivery semantics |

## Important Architectural Principles

1. **Monitoring is the product focus.** Scheduling and execution features exist to provide a realistic distributed system to observe.
2. **Redis is runtime infrastructure, not historical truth.** Durable monitoring history must survive Redis trimming, acknowledgement and restart.
3. **Job state and monitoring health are different concepts.**
   - Example: a run can be `RUNNING` while also having `SLA_VIOLATED`.
4. **A policy retry creates a new `run_id` in the same `execution_chain_id`, linked through `parent_run_id`; worker/delivery recovery may create a new `attempt_id` within one run.**
5. **A missing worker does not automatically mean a failed job.**
6. **PII findings must not require storage of raw PII values.**
7. **Cross-context communication should use stable contracts, not direct access to another context's internal tables.**
8. **At-least-once delivery must be assumed.**
9. **Events and commands must be idempotent where duplicate delivery is possible.**
10. **Monitoring gaps must be visible rather than silently hidden.**
11. **Real-time delivery is not a new source of truth.** Live updates use snapshot reconciliation and explicit resync when continuity is uncertain.
12. **The project observes itself.** Component and monitoring-pipeline health are visible alongside job health.

## Suggested Reading Order

```text
RFC-000
   │
   ├── RFC-001 Job Run Lifecycle
   ├── RFC-002 Scheduling
   ├── RFC-003 Redis Delivery
   └── RFC-004 Worker Runtime
            │
            ▼
      RFC-005 Monitoring
        ┌────┴─────┐
        ▼          ▼
     RFC-006    RFC-007
       PII       Alerts
        └────┬─────┘
             ▼
         RFC-008
       Investigation
             │
             ▼
         RFC-009
        Dashboard UI
             │
             ▼
         RFC-010
       Real-Time View
```

## RFC Status Convention

Each RFC uses one of:
- `Draft`
- `Proposed`
- `Accepted`
- `Superseded`
- `Rejected`

Open questions are intentional. They are areas where implementation teams should evaluate alternatives and record a decision rather than treating this RFC set as a complete implementation specification.

## Technical Implementation & System Design

See [`architecture/README.md`](architecture/README.md) for implementation and system-design companions to every RFC.
