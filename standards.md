# Standards Appendix

## Naming conventions
- Scenario: `WF_<Domain>_<Purpose>_<TriggerType>_<ENV>`
- `routeName`: lowercase snake_case (e.g., `actual_hours_export`, `sap_project_setup`)
- Data stores: `Integration_Config`, `DLQ`, `Idempotency_Ledger`, `Watermarks`, `Orchestration_State`, `Route_Locks`
- Required variable names: `correlationId`, `runId`, `objectId`, `objectType`, `scenarioName`, `routeName`

## Logging schema (required fields)
```json
{
  "timestamp":"ISO-8601",
  "env":"DEV|QA|PROD",
  "scenarioName":"string",
  "routeName":"string",
  "runId":"string",
  "correlationId":"string",
  "sourceSystem":"string",
  "targetSystem":"string",
  "objectType":"string",
  "objectId":"string",
  "eventId":"string|null",
  "idempotencyKey":"string",
  "attempt":0,
  "classification":"TRANSIENT|PERMANENT|DATA_QUALITY|N/A",
  "action":"string",
  "durationMs":0,
  "httpStatus":0,
  "errorCode":"string|null",
  "message":"string",
  "payloadHash":"sha256:...",
  "piiRedacted":true
}
```

## Error taxonomy
- **TRANSIENT**: 429, 5xx, timeouts, network/transient connector errors
- **PERMANENT**: auth/permissions, endpoint/config errors, non-recoverable request errors
- **DATA_QUALITY**: missing/invalid business data, invalid transitions, malformed payloads

## Promotion checklist
- [ ] Config-driven routing in place
- [ ] Idempotency strategy documented and tested
- [ ] DLQ + replay worker implemented and tested
- [ ] Watermark rollback tested (if polling)
- [ ] Logs schema + Teams alerts validated
- [ ] Runbook includes disable/rollback/replay/validate
- [ ] Performance profile documented (`interval`, `maxReturnedEvents`, `maxCycles`)
- [ ] Secrets in connections; least privilege reviewed
- [ ] Blueprint export + version tag + release notes

## DoD rubric for new scenarios
A scenario is production-ready only when it is:
- **Correct**: deterministic keys, clear contracts, business trigger state defined
- **Resilient**: retries, DLQ, replay, safe rollback/disable
- **Operable**: logs, metrics, alerts, runbook, clear ownership
- **Secure**: least privilege, redaction, secret hygiene
- **Promotable**: configuration/version discipline and test evidence

---

# Reuse & Scale
- Package standard blueprints:
  - `T1 Scheduled Drain Starter`
  - `T2 DLQ + Alerts Starter`
  - `T3 Ledger + Replay Feed Starter`
  - `T4 Saga State + Step Resume Starter`
- Maintain standard performance profiles:
  - `UPSERT_SAFE_5MIN`
  - `FEED_SAFE_5MIN`
  - `FEED_THROUGHPUT_5MIN`
  - `FEED_CATCHUP_5MIN`
  - `SAGA_CONSERVATIVE_5MIN`
  - `SAGA_BALANCED_5MIN`
- Teach one non-negotiable rule:
  - **No production side-effects without idempotency, DLQ, replay, logs, and alerts.**
