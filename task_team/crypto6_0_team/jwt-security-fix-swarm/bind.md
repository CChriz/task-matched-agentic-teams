# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | C+A pipeline — all stages are sequential; only one role runs at a time |
| `total_wall_clock_budget` | 15 min | Upper bound for one full run: ~3 min analyst + ~5 min developer + ~5 min verifier + ~2 min integration |
| `total_token_budget` | 150k tokens | Budget across all 3 roles; codebase is small (~150 lines), analysis + attack scripts are the main consumers |
| `per_role_token_budget` | 60k per role | Symmetric: each role reads the full codebase and produces structured output |
| `max_kick_back_cycles` | 2 | Developer → Verifier → Developer → Verifier: 2 full cycles; on 3rd Verifier FAIL, halt and escalate |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates, enforces quality gates, and integrates outputs. The Leader does NOT analyze code, implement fixes, run attacks, or substitute any role's work.
- **C-pattern pipeline discipline**: each stage MUST NOT modify upstream stage outputs. The fix-developer implements from the analyst's checklist, not from a re-analysis. The verifier attacks the fixed code, not the analyst's report. No stage may redo or skip an upstream stage.
- **A-pattern adversarial isolation**: the adversarial-verifier MUST NOT see the analyst's fix recommendations before producing its independent attack verdict. The verifier receives the original task spec and the fix-developer's FIXES_APPLIED.md (what changed), but NOT the analyst's report (which would bias attack design). If the Leader accidentally includes the analyst report, the verifier MUST flag this and request clean inputs.
- **Contradiction handling**: when the analyst and verifier disagree (e.g., analyst marked a bug CONFIRMED but verifier cannot reproduce it, or verifier finds a new vulnerability the analyst missed), Leader surfaces contradictions verbatim in the Final Report. Leader NEVER mediates / picks a winner / averages opinions. Resolution is the human user's call.
- **No source modification by analyst or verifier**: only the fix-developer may edit source files. The analyst and verifier read code and produce reports; they MUST NOT write to any `.py` file.
- **Test-first discipline**: the fix-developer MUST run `pytest` after each individual fix, not just after all fixes. If a fix breaks any test, the developer MUST revert that fix and re-approach before proceeding to the next.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once (single retry only). On 2nd timeout, mark role's section in the Final Report as `[ROLE MISSING — teammate timed out]` and proceed with the remaining roles' outputs if at least 2 of 3 roles delivered. If only 1 of 3 roles delivered, the run is FAILED. |
| Malformed teammate output (does not match Output Schema) | Re-dispatch with the schema explicitly inlined and a "your previous output was malformed" preamble. Max 1 retry. On 2nd malformed output, mark as `[ROLE MISSING — malformed output]` and proceed. |
| Teammate refuses (e.g., adversarial-verifier claims it cannot design attacks) | Re-dispatch with the role's `Boundary > Mandatory` rule restated. If it still refuses, mark as `[ROLE INCONCLUSIVE]` and surface verbatim. The role's refusal itself is evidence the verification was not performed. |
| fix-developer cannot make tests pass | After 2 retries, escalate to user with the failing test output and the developer's FIXES_APPLIED.md. The user decides whether to proceed with the verifier anyway (acknowledging test failures). |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| Codebase > 1000 LOC | Warn user; reduce to 2-role pipeline (analyst + developer only, skip verifier). The analyst report becomes the final output. |
| Task spec lists > 10 bugs | Warn user; analyst prioritizes top 10 by severity. Remaining bugs noted as `[DEFERRED]`. |
| `pytest` not available in environment | Warn user before Step 2. fix-developer skips test runs and documents "tests not run — pytest unavailable." verifier skips independent test run. Both roles proceed with code-only changes. Final Report tagged `[WARNING: no test execution]`. |
| `python3` not available | Halt immediately — both fix implementation and attack execution require Python. Report to user and do not proceed. |

### Escalation rules

- If 2 of 3 roles return `[ROLE MISSING]`, the run is **FAILED** — emit a partial report with explicit "FAILED: insufficient role coverage" header and surface to user.
- If `total_wall_clock_budget` is exceeded, halt all in-flight teammates, emit whatever partial outputs exist, and tag the report `INCOMPLETE: budget exceeded`.
- If `total_token_budget` is exceeded mid-run, halt new dispatches, allow in-flight to complete, emit partial report tagged `INCOMPLETE: token budget exceeded`.
- If both kick-back cycles are exhausted and the verifier still reports FAIL, halt and escalate with the full chain of evidence: analyst report → developer FIXES_APPLIED.md → verifier attack failures. User decides whether to accept the risk or request manual intervention.
