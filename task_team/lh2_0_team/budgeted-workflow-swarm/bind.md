# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | C+A pipeline — sequential stages |
| `total_wall_clock_budget` | 10 min | ~2 min analyst + ~5 min developer + ~2 min auditor + integration |
| `total_token_budget` | 80k tokens | Small codebase — ~130 lines Python + 4 small JSON files |
| `per_role_token_budget` | 30k per role | Symmetric |
| `command_budget` | 28 | Hard limit from task spec — each read/write/validate/fix counts |
| `max_kick_back_cycles` | 2 | Developer → Auditor → Developer → Auditor |

## Behavioral Constraints

- **Leader-as-orchestrator only**: Leader dispatches, enforces gates, integrates. Leader does NOT read files, fix code, or count budget.
- **C-pattern pipeline discipline**: fix-developer implements from analyst's strategy, not re-analysis. auditor verifies from clean state, not re-analysis.
- **A-pattern adversarial isolation**: budget-auditor MUST NOT see analyst's strategy. Auditor receives fixed workspace only — counts independently.
- **Budget discipline**: fix-developer must delete stale budget_log.jsonl before each test run. auditor must start from clean state (delete all logs, output, tmp).
- **Contradiction handling**: if auditor's budget count doesn't match fixer's report, surface verbatim. NEVER mediate.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Timeout | Retry once. On 2nd timeout, mark `[ROLE MISSING]`. |
| Malformed output | Re-dispatch. Max 1 retry. |
| fix-developer exceeds budget | Escalate with budget_log.jsonl. User decides whether to add budget or accept partial fix. |
| auditor cannot reproduce fixer's results | Report discrepancy verbatim. User investigates. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| >10 data files | Warn user; fix only the 3 specified files. |
| `python3` not available | Halt — cannot run scripts. |

### Escalation rules

- If 2 of 3 roles return `[ROLE MISSING]`, run FAILED.
- If budget exceeded mid-fix, halt, emit partial, escalate.
