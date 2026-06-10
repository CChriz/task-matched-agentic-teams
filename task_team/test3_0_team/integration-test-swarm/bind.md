# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | C+A pipeline — all stages sequential |
| `total_wall_clock_budget` | 15 min | ~3 min analyst + ~7 min author (writing + running tests) + ~3 min validator + ~2 min integration |
| `total_token_budget` | 150k tokens | Budget across 3 roles; server code ~160 lines, spec ~190 lines |
| `per_role_token_budget` | 60k per role | Symmetric |
| `max_kick_back_cycles` | 2 | Author → Validator → Author → Validator: 2 full cycles max |

## Behavioral Constraints

- **Leader-as-orchestrator only**: Leader dispatches, enforces gates, integrates. Leader does NOT write tests, run servers, or audit.
- **C-pattern pipeline discipline**: each stage MUST NOT modify upstream outputs. Test-author implements from analyst's checklist (not re-analysis). Validator audits tests (not re-analyzes contract).
- **A-pattern adversarial isolation**: the adversarial-validator MUST NOT see the analyst's report before producing its independent audit. Validator receives grading criteria + test file + servers, NOT the analyst's checklist.
- **Contradiction handling**: when analyst and validator disagree on coverage, Leader surfaces verbatim. NEVER mediates.
- **No test modification by analyst or validator**: only test-author edits `tests/test_integration.py`.
- **Server isolation**: validator must start a fresh server instance for each test run (working vs broken). Do not leave stale servers running.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once. On 2nd timeout, mark `[ROLE MISSING — timed out]`. Proceed if ≥2 roles delivered. |
| Malformed output | Re-dispatch with schema inlined. Max 1 retry. On 2nd malformed, mark `[ROLE MISSING]`. |
| Test-author tests fail against working server | After 2 retries, escalate with test output. User decides whether tests are wrong or server is buggy. |
| Validator cannot start server | Mark as BLOCKED with specific error. Check that flask and server code are accessible. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| API spec has >10 endpoints | Warn user; analyst prioritizes by criticality. Remaining endpoints noted as `[DEFERRED]`. |
| `flask` or `requests` not available | Warn before Step 2. Tests cannot run. Test-author writes tests without execution. Final Report tagged `[WARNING: no test execution]`. |
| `python3` not available | Halt — tests and servers cannot run. Report to user. |

### Escalation rules

- If 2 of 3 roles return `[ROLE MISSING]`, run is **FAILED** — emit partial report.
- If `total_wall_clock_budget` exceeded, halt in-flight, emit partial, tag `INCOMPLETE: budget exceeded`.
- If both kick-back cycles exhausted and validator still reports FIX-REQUIRED, escalate with full chain.
