# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | Pipeline is strictly sequential (Pattern C) — only one stage runs at a time |
| `total_wall_clock_budget` | 15 min | Upper bound: ~3 min analysis + ~5 min fixing + ~5 min verification + buffer |
| `total_token_budget` | 150k tokens | Budget across all 3 stages; codebases are small (~500 LOC total) so this is generous |
| `per_stage_token_budget` | 50k per stage | Symmetric — each stage reads similar input volume |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates in sequence and integrates outputs into the Final Reconciliation Report. The Leader does NOT perform analysis, fixing, or verification — those are the exclusive domains of the 3 stage roles.
- **Sequential pipeline discipline**: each stage MUST complete and pass its quality gate before the next stage starts. Stages cannot be skipped or reordered.
- **Server immutability**: NO `.go` files may be modified by any stage or the Leader. The Go server is the source of truth. If a stage attempts to modify a `.go` file, the Leader must refuse the output and re-dispatch.
- **Stage boundary enforcement**: the contract-analyzer may only read files. The client-fixer may read and write Python/YAML files. The contract-verifier may only read files and run commands. Cross-boundary actions are rejected.
- **Contradiction handling**: not applicable in this pipeline (stages are sequential and non-overlapping). If the analyzer and verifier disagree on resolution status, the Leader surfaces the disagreement verbatim in the Final Report and lets the user decide.
- **Prescription compliance**: the client-fixer MUST follow the analyzer's fix prescriptions exactly. Creative deviations ("I also refactored X while I was there") are forbidden.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once. On 2nd timeout, mark the stage's section in the Final Report as `[STAGE MISSING — timeout]` and proceed with remaining stages. If Stage 1 fails, the pipeline halts entirely (no input for Stage 2). |
| Malformed teammate output (does not match Output Schema) | Re-dispatch with the schema explicitly inlined and a "your previous output was malformed" preamble. Max 1 retry. On 2nd malformed output, mark as `[STAGE MALFORMED]` and proceed if possible. |
| Analyzer finds <3 mismatches | Re-dispatch with explicit instruction to recheck all server struct tags. If still <3, surface to user: "Only N mismatches found. Proceed or abort?" |
| Fixer fails to address a mismatch | Re-dispatch with the specific unaddressed mismatch highlighted. On 2nd failure, surface to user with the partial fix report. |
| Verifier reports test failures | Do NOT re-dispatch the fixer automatically. Surface the test failures in the Final Report and let the user decide whether to re-run the pipeline. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| Workspace missing `service/` or `client/` directory | Halt in Step 0. Report missing components. Do not start the pipeline. |
| Workspace has >20 source files or >5000 LOC | Warn the user: "Large codebase detected. The analyzer may produce incomplete results. Consider narrowing scope to specific endpoints." Proceed if user confirms. |
| `api_spec.yaml` is missing or malformed | Proceed without spec analysis; note in the analyzer report that spec validation was skipped. |
| Python environment lacks `pytest` | Verifier skips `pytest tests/` and reports "pytest not available — manual test verification required." |
| Go toolchain not available | Verifier skips `go build ./...` and reports "go not available — manual compilation check required." |

### Escalation rules

- If Stage 1 (analyzer) fails after retry, the run is **FAILED** — no analysis to fix from.
- If Stage 2 (fixer) fails after retry, surface the analyzer's report with the partial fix report and ask the user how to proceed.
- If Stage 3 (verifier) reports test failures, surface in Final Report — do NOT auto-retry the fixer.
- If `total_wall_clock_budget` is exceeded, halt all in-flight stages, emit whatever partial outputs exist, and tag the report `INCOMPLETE: budget exceeded`.
- If `total_token_budget` is exceeded mid-run, halt new dispatches, allow in-flight to complete, emit partial report tagged `INCOMPLETE: token budget exceeded`.
