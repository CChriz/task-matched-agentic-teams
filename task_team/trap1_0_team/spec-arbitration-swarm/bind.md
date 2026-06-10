# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | C+A pipeline — sequential |
| `total_wall_clock_budget` | 12 min | ~3 min analyst + ~5 min developer + ~3 min verifier |
| `total_token_budget` | 100k tokens | Spec + CHANGELOG + code + tests |
| `per_role_token_budget` | 40k per role | |
| `max_kick_back_cycles` | 2 | |

## Behavioral Constraints

- **Leader-as-orchestrator only**: Leader dispatches, integrates. Does NOT read spec or implement validation.
- **C-pattern pipeline**: fix-developer implements from analyst's categorization. verifier audits from endpoint list.
- **A-pattern adversarial isolation**: compliance-verifier MUST NOT see analyst's report. Verifier receives only the STRICT/RELAXED endpoint list, then independently tests.
- **Authority rule**: CHANGELOG is authoritative over OpenAPI spec. RELAXED endpoints in CHANGELOG MUST stay relaxed.
- **422 error format**: all validation rejections use HTTP 422 + `{"error": "<reason>"}`.

## Failure Handling

### (a) Teammate failure
| Failure | Response |
|---|---|
| Timeout | Retry once. Mark `[ROLE MISSING]` on 2nd failure. |
| Malformed output | Re-dispatch. Max 1 retry. |
| Tests fail | Escalate to user with test output. |

### (b) Input over-scale
| Trigger | Degraded mode |
|---|---|
| >20 endpoints | Prioritize by CHANGELOG coverage; defer unlisted. |
| `pytest` missing | Warn; skip test execution. Tag `[WARNING: no tests]`. |
