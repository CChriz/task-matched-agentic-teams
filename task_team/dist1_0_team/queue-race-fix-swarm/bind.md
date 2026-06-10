# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | C+A pipeline — all stages sequential; only one role runs at a time |
| `total_wall_clock_budget` | 20 min | ~4 min analyst + ~8 min developer (per-fix test runs) + ~6 min tester (stress tests) + ~2 min integration. Concurrency stress tests may need extra time |
| `total_token_budget` | 180k tokens | Budget across 3 roles; codebase is ~200 lines, test files ~250 lines. Stress test design and execution consume the most tokens |
| `per_role_token_budget` | 70k per role | Slightly asymmetric — developer and tester may need more for test output |
| `max_kick_back_cycles` | 2 | Developer → Tester → Developer → Tester: 2 full cycles max |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates, enforces gates, integrates outputs. The Leader does NOT analyze code, implement fixes, run stress tests, or substitute any role's work.
- **C-pattern pipeline discipline**: each stage MUST NOT modify upstream outputs. Fix-developer implements from analyst's checklist (not re-analysis). Tester attacks fixed code (not analyst's report).
- **A-pattern adversarial isolation**: the adversarial-tester MUST NOT see the analyst's fix recommendations before producing its independent stress tests. The tester receives task spec + FIXES_APPLIED.md (what changed), NOT the analyst's report (which would bias attack design).
- **Contradiction handling**: when analyst and tester disagree, Leader surfaces verbatim. Leader NEVER mediates.
- **No source modification by analyst or tester**: only fix-developer edits `.py` files.
- **Test-first discipline**: fix-developer MUST run full `pytest` after EACH individual fix. If any fix breaks a test, revert and re-approach before proceeding.
- **Ack/nack contract awareness**: tests check `isinstance(result, tuple)` and call `q.ack(receipt)`. The developer's get()/ack() signatures MUST match this contract.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once. On 2nd timeout, mark as `[ROLE MISSING — timed out]` and proceed if ≥2 of 3 roles delivered. If only 1 delivered, run is FAILED. |
| Malformed output (does not match schema) | Re-dispatch with schema inlined. Max 1 retry. On 2nd malformed, mark as `[ROLE MISSING — malformed output]`. |
| Teammate refuses (e.g., tester claims cannot design stress tests) | Re-dispatch with Mandatory rule restated. If still refuses, mark `[ROLE INCONCLUSIVE]` and surface. |
| Developer cannot make tests pass | After 2 retries, escalate to user with failing test output + FIXES_APPLIED.md. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| Codebase > 2000 LOC | Warn user; reduce to 2-role pipeline (analyst + developer only, skip tester). |
| Task spec lists > 6 bugs | Warn user; analyst prioritizes top 6 by severity. Remaining noted as `[DEFERRED]`. |
| `pytest` not available | Warn before Step 2. Developer skips test runs, documents "tests not run — pytest unavailable." Tester skips independent run. Final Report tagged `[WARNING: no test execution]`. |
| `python3` not available | Halt — both fix implementation and stress tests require Python. Report to user. |

### Escalation rules

- If 2 of 3 roles return `[ROLE MISSING]`, run is **FAILED** — emit partial report with "FAILED: insufficient role coverage."
- If `total_wall_clock_budget` exceeded, halt in-flight, emit partial outputs, tag `INCOMPLETE: budget exceeded`.
- If `total_token_budget` exceeded mid-run, halt new dispatches, allow in-flight to complete, tag `INCOMPLETE: token budget exceeded`.
- If both kick-back cycles exhausted and tester still reports FAIL, halt and escalate with full chain: analyst → developer FIXES_APPLIED.md → tester attack failures.
