# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | Sequential pipeline — only one stage runs at a time |
| `total_wall_clock_budget` | 15 min | Upper bound for one full run: ~3 min analysis + ~5 min implementation + ~5 min testing + retry cycles |
| `total_token_budget` | 180k tokens | Budget across all 3 roles; spec is ~6KB, implementation is new code |
| `verifier_kickback_max` | 2 | Maximum Developer→Verifier kick-back cycles before escalation to user |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates and integrates outputs. The Leader does NOT write code, run tests, or substitute any role's work.
- **Sequential stage discipline**: each stage MUST NOT modify upstream stage outputs. The Developer implements from the Analyst's matrix; the Verifier tests against the spec, not the Developer's claims.
- **Verifier independence**: the Verifier MUST design its test plan from the specification BEFORE reading the Developer's implementation report. This prevents test cases from being biased by what the Developer claims to have implemented.
- **Contradiction handling**: if the Verifier's test results contradict the Developer's smoke tests, the Leader surfaces both verbatim. The Leader NEVER mediates — the human user resolves.
- **Schema is source of truth**: `get_schema()` must return the full schema defined in the spec. The Developer must not use `config_schema.json` as the authoritative schema — the spec document is authoritative per the task notes.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once. On 2nd timeout, mark role as `[ROLE MISSING — timed out]` and proceed. |
| Malformed teammate output (does not match Output Schema) | Re-dispatch with schema inlined. Max 1 retry. On 2nd malformed output, mark as `[ROLE MISSING — malformed output]`. |
| Teammate refuses (e.g., Analyst claims "spec is already clear") | Re-dispatch with `Boundary > Mandatory` rule restated. If still refuses, mark as `[ROLE INCONCLUSIVE]` and proceed with warning. |
| Developer produces code that fails to import | Re-dispatch with the specific ImportError traceback. Max 1 retry. |
| Developer produces code that passes smoke tests but fails Verifier tests | Pipeline kicks back with Violation Details. Max 2 kick-back cycles. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| Spec defines > 20 config keys | Warn user; still attempt pipeline but note that tests may not cover all combinations exhaustively. |
| `config_skeleton.py` missing from workspace | Proceed — the skeleton is a reference only; the Developer writes `config_system.py` from scratch. |
| `config_schema.json` present but the spec warns it is partial | Apply the behavioral constraint: use the spec, not the JSON file, as authoritative. Flag in pre-flight report. |

### Escalation rules

- If 50%+ of roles return `[ROLE MISSING]`, the run is **FAILED** — emit partial report with "FAILED: insufficient role coverage".
- If `total_wall_clock_budget` is exceeded, halt in-flight, emit partial outputs, tag `INCOMPLETE: budget exceeded`.
- If `verifier_kickback_max` is reached (2 cycles exhausted, 3rd REJECT), surface both the Verifier's Violation Details and the Developer's fix history. Ask user: "Pipeline could not resolve all failures after 2 fix cycles. Human review required."
