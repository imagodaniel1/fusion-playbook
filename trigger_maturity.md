# Trigger Maturity Model (T0–T5)

## Purpose
Align trigger and buffering choices to operational maturity. This is especially useful for comparing:
- inline instant triggers
- scheduled instant triggers (native queued webhook drain)
- polling/reconcile
- saga orchestration

## Maturity ladder

| Tier | Label | Trigger pattern | Buffering | Idempotency | Replay | Ops / Observability | Best for | Biggest risk |
|---:|---|---|---|---|---|---|---|---|
| T0 | Instant trigger, no special architecture | Inline webhook/watch events | None | None/ad hoc | Manual rerun | Minimal | Prototype/low-risk | Duplicates, brittle retries |
| T1 | Scheduled instant trigger (native queue drain) | Queued webhooks on schedule | Native queue | Record-level extRefID (basic) | Manual reschedule | Basic logs | Simple upserts | Queue is black box; partial success unclear |
| T2 | Scheduled drain + ops wrapper | Scheduled drain | Native queue / basic DS | Namespaced extRefID + basic rules | DLQ + manual replay | Teams alerts + error taxonomy | Production upserts | Multi-step/cross-system still weak |
| T3 | Ledger + replay + rate guards | Scheduled drain or polling | Queue + batching | Operation-level ledger (+ extRefID where useful) | Replay worker | Metrics + runbooks | High-volume feeds (hours) | Key design errors |
| T4 | Saga orchestration + reconcile | Hybrid push/poll | Queue + orchestration state | Step-level idempotency + locks | State-driven resume + replay | SLOs + state visibility | Low-volume, high-impact ERP setup | Partial success without state |
| T5 | Platform governance | Any | Any | Standardized platform-wide | Audited/controlled | Promotion gates + drills | Multi-team enterprise platform | Entropy without governance |

## Promotion gates
### T0 → T1
- Scheduled queued drain enabled
- Correlation ID logging
- Basic retry policy

### T1 → T2
- Standardized extRefID format
- Error taxonomy
- DLQ + Teams alerts + runbook

### T2 → T3
- Idempotency ledger
- Replay worker
- Batching/rate guard
- Watermark polling for feed-style routes

### T3 → T4
- Orchestration state + locks
- Step-level keys
- Reconciliation/backstop
- Replay-by-step

### T4 → T5
- Promotion and config governance
- SLOs and incident drills
- Standard profiles and review gates

---

# Drain Budget Controls (Maximum Returned Events + Number of Cycles)

## What these settings are
- **Maximum number of returned events** = intake cap / micro-batch size
- **Number of cycles** = loop budget per scheduled execution

## Capacity heuristic
```text
Max events per scheduled execution ≈ maxReturnedEvents × maxCycles
```

```text
Approx drain capacity per minute ≈ (maxReturnedEvents × maxCycles) / intervalMinutes
```

## Why they matter in the maturity model
- T1+: they become real architecture tuning knobs for queue drain behavior
- T3+: they must be governed with idempotency, retries, and rate limits
- T4: they are **secondary safety knobs** (blast radius and execution duration), not the primary reliability mechanism

## Safe tuning order
1. Idempotency + DLQ + replay first
2. Tune `maxReturnedEvents`
3. Tune `maxCycles`
4. Tune schedule interval
5. Tune rate guard/batch size

## Symptoms → likely adjustment
- Backlog grows, no failures → increase maxReturnedEvents or cycles gradually
- 429s rise → reduce drain budget and tighten rate guard
- Incomplete executions rise → reduce cycles first, then maxReturnedEvents
- DLQ spikes from poison payloads → lower maxReturnedEvents to reduce blast radius

---

# Workload-specific maturity profiles (5-minute minimum interval)

## 1) Actual Hours (Workfront → SAP) — T3 Feed
### Workload assumptions
- ~1,000 records/week, mostly entered/submitted on Fridays
- Edits/revisions may happen after initial entry

### Recommended target
- **Tier T3**
- Prefer polling with watermark for completeness (or scheduled queued trigger drain if already implemented and reliable)

### Starting profile: `FEED_THROUGHPUT_5MIN`
- `scheduleIntervalMin = 5`
- `maxReturnedEvents = 50`
- `maxCycles = 2`
- Drain budget = **100 events per 5 min**

### Safer profile: `FEED_SAFE_5MIN`
- `scheduleIntervalMin = 5`
- `maxReturnedEvents = 25`
- `maxCycles = 2`
- Drain budget = **50 events per 5 min**

### Catch-up profile (temporary): `FEED_CATCHUP_5MIN`
- `scheduleIntervalMin = 5`
- `maxReturnedEvents = 50`
- `maxCycles = 4`
- Drain budget = **200 events per 5 min**

### Idempotency guidance (critical)
Use **entry-level (and often revision-level)** keys:
- `SAP:ACTUAL_HOURS:<wfHourId>:<revisionOrHash>`

### Alerts (Teams)
- backlog/queue delay > 10 min during Friday peak
- DLQ count > 10 in 15 min
- consecutive failures ≥ 3
- any incomplete execution in PROD

---

## 2) SAP Project Setup Saga (Project + 3 WBS levels + planned allocations + financial plan) — T4
### Workload assumptions
- ~200/year; ~15–20/month (low volume, high consequence)
- One Workfront project triggers multiple dependent SAP side-effects

### Recommended target
- **Tier T4**
- Prioritize correctness/resume over throughput

### Default profile: `SAGA_CONSERVATIVE_5MIN`
- `scheduleIntervalMin = 5`
- `maxReturnedEvents = 5`
- `maxCycles = 1`
- Drain budget = **5 projects per 5 min** (more than enough for typical volume)

### Burst profile (optional): `SAGA_BALANCED_5MIN`
- `scheduleIntervalMin = 5`
- `maxReturnedEvents = 5`
- `maxCycles = 2`
- Drain budget = **10 projects per 5 min**

### Required controls (not optional)
- `Orchestration_State` DS keyed by `wfProjectId`
- `Route_Locks` DS by `wfProjectId`
- Step-level idempotency keys for:
  - create project
  - WBS L1/L2/L3
  - allocation plan post
  - financial plan post
- Replay-by-step (resume), not blind full rerun

### Alerts (Teams)
- Any DLQ on this route (low-volume, high-impact)
- Any incomplete execution in PROD
- Saga stuck IN_PROGRESS > threshold (e.g., 15 min)
- Consecutive failures ≥ 2

---

# Incomplete Executions — How they fit the solution

## Position in the operating model
- Incomplete executions are a **safety net / symptom**, not the primary replay mechanism.
- In a mature production design, incomplete executions should be **rare**.

## What they are good for
- Operator forensics (inspect where the run failed)
- Manual reschedule for transient issues (only if idempotency boundary is solid)
- Signal that a failure escaped your classification logic

## What they are not good for
- Governed replay at scale
- Data-quality quarantine
- Multi-step transactional recovery

## Design rule
> Use incomplete executions as an **unexpected/systemic-failure signal** and route them to Teams. Use **DLQ + replay worker + state** for normal recovery.

## Practical implications
- **Actual hours (T3):** rescheduling can be safe if entry-level idempotency is implemented
- **SAP project setup saga (T4):** rescheduling is only safe if step-level idempotency and orchestration state exist

---

# IT Domains that inform these patterns (architectural rationale)

These Fusion decisions are informed by multiple IT domains:

1. **Distributed Systems**
   - at-least-once delivery, duplicates, partial failure, out-of-order events
   - informs idempotency, retries, replay, saga safety

2. **Enterprise Integration Architecture**
   - routing, queueing, polling consumers, orchestration patterns
   - informs trigger choice, queue/buffer patterns, routers, DLQ

3. **SRE / Production Operations**
   - observability, alerts, runbooks, incident response
   - informs logging schema, Teams routing, incomplete execution handling, SLOs

4. **Data Engineering**
   - watermarks, pagination, incremental processing, late-arriving data
   - informs polling, overlap windows, checkpoint rollback

5. **API Engineering**
   - 429/5xx/4xx semantics, auth models, quotas, contract drift
   - informs error taxonomy and retry policies

6. **Workflow Orchestration / Transaction Processing**
   - stateful sequencing, step resume, compensation/forward recovery
   - informs saga state, locks, replay-by-step for SAP project setup

7. **Security / IAM / Compliance**
   - least privilege, secret management, redaction, auditability
   - informs connection usage, payload hashing, environment separation

8. **Software Architecture / Config Management**
   - modularity, config-driven routing, versioning, promotion discipline
   - informs `Integration_Config`, shared subflows, platform standards

9. **ITSM / Ops Governance**
   - ownership, escalation, incident/problem/change workflows
   - informs Teams alert routing, runbooks, promotion gates

10. **Performance / Capacity Management**
   - throughput vs latency, quota-aware tuning, drain budgets
   - informs `maxReturnedEvents`, `maxCycles`, schedule interval, rate guards

---

# Quick Start — Building a production-ready Fusion scenario (one page)

## Design
- [ ] Classify route: simple upsert (T1/T2), feed (T3), or saga (T4)
- [ ] Choose trigger pattern: webhook/watch events/polling/hybrid
- [ ] Define `routeName`, `correlationId`, `objectId`, `idempotencyKey` formula
- [ ] Choose buffering: native queue / inbox / debounce / batch
- [ ] Define error taxonomy + retry policy + DLQ policy
- [ ] Decide whether reconcile/backstop is required

## Build
- [ ] Create Data Stores (as needed): `Integration_Config`, `DLQ`, `Idempotency_Ledger`, `Watermarks`, `Orchestration_State`, `Route_Locks`
- [ ] Implement normalization + correlation
- [ ] Implement config-driven route resolution
- [ ] Implement idempotent side-effects
- [ ] Implement error handler (classify + retry + DLQ)
- [ ] Implement replay worker
- [ ] Implement structured logs + Teams alert routing

## Test
- [ ] Happy path
- [ ] Duplicate/replay test
- [ ] Transient error retry test (429/500)
- [ ] Permanent/data-quality failure to DLQ
- [ ] Incomplete execution handling (unexpected failure) and runbook path
- [ ] Burst/load/quota test (as applicable)

## Release
- [ ] Promotion checklist complete
- [ ] Runbook published
- [ ] Alerts validated in Teams
- [ ] Config/profile values documented (`maxReturnedEvents`, `maxCycles`, interval)
- [ ] Go-live monitoring window scheduled

---
