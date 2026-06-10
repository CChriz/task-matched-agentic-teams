# Execution Guardrails

## Resource Constraints

| Item | Limit | Reason |
|---|---|---|
| `max_parallel_teammates` | 1 | Sequential pipeline — only one stage runs at a time |
| `total_wall_clock_budget` | 12 min | Upper bound for one full run: ~2 min per stage + 2 retry cycles |
| `total_token_budget` | 100k tokens | Budget across all 3 roles; corpus documents are small |
| `verifier_kickback_max` | 2 | Maximum Arbiter→Verifier kick-back cycles before escalation to user |

## Behavioral Constraints

- **Leader-as-orchestrator only**: the Leader dispatches teammates and integrates outputs. The Leader does NOT read the corpus, resolve conflicts, or substitute any role's work.
- **Sequential stage discipline**: each stage MUST NOT modify upstream stage outputs — only consume them as input. The Arbiter resolves based on the Analyzer's matrix; the Verifier verifies against the corpus, not the Arbiter's resolution log.
- **Verifier independence**: the Verifier MUST complete its independent verification against the corpus and priority rules BEFORE reading the Arbiter's resolution log. This prevents confirmation bias and catches errors the Arbiter may have justified incorrectly.
- **Contradiction handling**: if the Verifier's findings contradict the Arbiter's resolution log, the Leader surfaces both verbatim in the Final Report. The Leader NEVER mediates which interpretation is correct — the human user resolves the conflict.
- **`compression` special case is mandatory**: every stage that touches the `compression` field MUST acknowledge and apply the legacy compatibility exception from `priority_rules.txt`. Ignoring the footnote is a spec violation.

## Failure Handling

### (a) Teammate failure

| Failure mode | Response |
|---|---|
| Teammate timeout | Retry once (single retry only). On 2nd timeout, mark role's section in the Final Report as `[ROLE MISSING — teammate timed out]` and proceed with remaining roles' outputs. |
| Malformed teammate output (does not match Output Schema) | Re-dispatch with the schema explicitly inlined and a "your previous output was malformed" preamble. Max 1 retry. On 2nd malformed output, mark as `[ROLE MISSING — malformed output]` and proceed. |
| Teammate refuses (e.g., Analyzer claims "no conflicts found") | Re-dispatch with the role's `Boundary > Mandatory` rule restated. If it still refuses, mark as `[ROLE INCONCLUSIVE]` and surface verbatim. Proceed to Arbiter with a warning that analysis may be incomplete. |
| Arbiter produces invalid JSON or wrong key count | Re-dispatch with the specific failure (parse error or key count mismatch). Max 1 retry. On 2nd failure, mark as `[ARBITER FAILED]`. |
| Verifier produces REJECT with specific violations | Pipeline kicks back to Arbiter (Step 2) with the Verifier's Violation Details table. Max 2 kick-back cycles. On 3rd REJECT, surface both reports to the user. |

### (b) Input over-scale degradation

| Trigger condition | Degraded mode |
|---|---|
| Corpus > 5 spec documents or > 20 keys | Warn user; still attempt pipeline but note that verification may be less thorough. |
| `priority_rules.txt` missing special case footnotes section | Warn user: "Priority rules file has no FOOTNOTES section — the `compression` special case may be undocumented. Proceed with normal priority resolution only." Proceed, but flag in Final Report. |
| `requirements.txt` missing one or more spec blocks | Halt at Step 0. Report to user: "Corpus incomplete — cannot resolve without all spec documents." |

### Escalation rules

- If 50%+ of roles return `[ROLE MISSING]`, the run is **FAILED** — emit a partial report with explicit "FAILED: insufficient role coverage" header.
- If `total_wall_clock_budget` is exceeded, halt all in-flight teammates, emit whatever partial outputs exist, and tag the report `INCOMPLETE: budget exceeded`.
- If `verifier_kickback_max` is reached (2 cycles exhausted, 3rd REJECT), halt the pipeline, emit both the Verifier's Violation Details and the Arbiter's resolution log, and surface to user with: "Pipeline could not resolve all violations after 2 arbitration cycles. Human review required."
