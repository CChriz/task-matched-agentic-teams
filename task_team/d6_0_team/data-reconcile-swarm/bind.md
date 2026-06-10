# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | Sequential pipeline — only one stage runs at a time |
| `total_wall_clock_budget` | 15 min | Upper bound for one full run: ~3 min per stage + 2 retry cycles |
| `total_token_budget` | 150k tokens | Budget across all 3 roles; data files are ~18KB combined |
| `verifier_kickback_max` | 2 | Maximum Developer→Verifier kick-back cycles before escalation to user |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates and integrates outputs. The Leader does NOT write code, run analysis, or substitute any role's work.
- **Sequential stage discipline**: each stage MUST NOT modify upstream stage outputs — only consume them as input. The Developer fixes code based on the Analyzer's bug report; the Verifier verifies against the spec, not the Developer's claims.
- **Verifier independence**: the Verifier MUST complete its independent verification against the specification BEFORE reading the Developer's fix report. This prevents confirmation bias.
- **Contradiction handling**: if the Verifier's findings contradict the Developer's fix report, the Leader surfaces both verbatim in the Final Report. The Leader NEVER mediates which one is correct — the human user resolves the conflict.
- **Field ownership constants are immutable**: `A_OWNED_FIELDS`, `B_OWNED_FIELDS`, `SHARED_FIELDS`, `ALL_OUTPUT_FIELDS` in `reconcile.py` carry a "DO NOT CHANGE" directive in the source. No role may modify these constants.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once (single retry only). On 2nd timeout, mark role's section in the Final Report as `[ROLE MISSING — teammate timed out]` and proceed with remaining roles' outputs. |
| Malformed teammate output (does not match Output Schema) | Re-dispatch with the schema explicitly inlined and a "your previous output was malformed" preamble. Max 1 retry. On 2nd malformed output, mark as `[ROLE MISSING — malformed output]` and proceed. |
| Teammate refuses (e.g., Analyzer claims "code looks fine" without finding any bugs) | Re-dispatch with the role's `Boundary > Mandatory` rule restated. If it still refuses, mark as `[ROLE INCONCLUSIVE]` and surface verbatim. Proceed to Developer with a warning that analysis may be incomplete. |
| Developer produces script that errors on execution | Re-dispatch with the specific error traceback. Max 1 retry. On 2nd failure, mark as `[DEVELOPER FAILED]` and surface error to user. |
| Verifier produces REJECT with specific violations | Pipeline kicks back to Developer (Step 2). Max 2 kick-back cycles. On 3rd REJECT, surface both the Verifier's Violation Details and the Developer's fix history to the user. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| `reconcile.py` > 200 lines or data files > 50 records each | Warn user; still attempt pipeline but note that verification may be less thorough for edge cases. |
| Specification missing required sections (no field ownership table, no rules list) | Halt at Step 0. Report to user: "Specification incomplete — cannot reconcile without field ownership designations and conflict resolution rules." |
| `system_a.json` or `system_b.json` missing from workspace | Halt at Step 0. Report to user with missing file names. |

### Escalation rules

- If 50%+ of roles return `[ROLE MISSING]`, the run is **FAILED** — emit a partial report with explicit "FAILED: insufficient role coverage" header.
- If `total_wall_clock_budget` is exceeded, halt all in-flight teammates, emit whatever partial outputs exist, and tag the report `INCOMPLETE: budget exceeded`.
- If `verifier_kickback_max` is reached (2 cycles exhausted, 3rd REJECT), halt the pipeline, emit both the Verifier's Violation Details and the Developer's fix history, and surface to user with: "Pipeline could not resolve all violations after 2 fix cycles. Human review required."
