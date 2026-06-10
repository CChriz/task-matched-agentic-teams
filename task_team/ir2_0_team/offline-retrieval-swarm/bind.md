# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 2 | Both researchers run in parallel (B-pattern fan-out) |
| `total_wall_clock_budget` | 8 min | ~3 min parallel research + ~3 min verifier + integration |
| `total_token_budget` | 80k tokens | 3 small text documents + analysis |
| `per_role_token_budget` | 30k per role | Symmetric |
| `max_retries` | 1 per researcher, 2 for verifier | |

## Behavioral Constraints

- **Leader-as-orchestrator only**: Leader dispatches, integrates. Leader does NOT read corpus files or produce answers.
- **B-pattern isolation (Stage 1)**: corpus-researcher-1 and corpus-researcher-2 MUST NOT see each other's outputs. Isolation is the value — if they converge from different document sets, the answer is trustworthy.
- **A-pattern cross-check (Stage 2)**: evidence-verifier receives both reports and cross-references against source documents. Verifier NEVER picks a winner without re-reading the documents itself.
- **Trap document rule**: doc_trap.txt is NEVER valid evidence. Even if it contains the correct answer, citing it = automatic failure. The verifier MUST enforce this.
- **Offline constraint**: no web search, no API calls, no external data. All answers come from the provided corpus only.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Researcher timeout | Retry once. On 2nd timeout, mark `[ROLE MISSING]` — verifier proceeds with single researcher output but escalates. |
| Malformed output | Re-dispatch. Max 1 retry. |
| Researchers disagree on answer | Verifier re-reads source documents to resolve. Marks conflict in report. |
| Verifier cannot confirm answer | Escalate with both research reports + corpus. User resolves. |

### (b) Input over-scale degradation

| Trigger | Degraded mode |
|---|---|
| Corpus > 10 documents | Analyst prioritizes doc_A + doc_B + doc_trap; others noted as `[DEFERRED]`. |
| Document > 5000 words | Read first 2000 + last 500 words; note `[TRUNCATED]`. |
| Corpus files missing | Halt — cannot complete without documents. |
