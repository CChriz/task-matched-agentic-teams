---
name: config-system-swarm
description: |
  3-stage pipeline (Spec Analyst → Config Developer → Compliance Verifier) that implements a config system from a formal spec with validation, type coercion, and priority cascade.
  Use when a config system must be built from a spec with typed keys, validation rules, coercion, and multi-source priority — and correctness requires independent adversarial testing.
  Do NOT use for simple key-value configs, specs without formal rules, or already-verified implementations.
version: "0.1"
kind: swarm-skill
roles:
  - id: analyst
    kind: ai_agent
    purpose: "Extract all 10 config keys with exact types, defaults, validation bounds, coercion rules, and priority cascade into a requirements matrix."
    skills: []
    tools: []
  - id: developer
    kind: ai_agent
    purpose: "Implement config_system.py from the requirements matrix: ConfigValidationError, get_schema, validate_value, and load_config — all fully implemented."
    skills: []
    tools: [python]
  - id: verifier
    kind: ai_agent
    purpose: "Design and execute a test suite covering every validation boundary, bool coercion form, and priority cascade; produce ATTEST or REJECT."
    skills: []
    tools: [python]
---

# Configuration System Swarm

A 3-stage specialization pipeline that implements a configuration management system from a formal specification. The single-agent failure mode this solves: a solo developer reads the spec, implements the schema and validation, and runs a few manual tests — but misses a coercion edge case (e.g., "on"/"off" bool inputs), mis-handles the `""` vs `None` priority cascade override, or forgets a key. The Verifier independently designs tests from the spec and catches these errors.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify `python` is available (required). Verify spec and workspace files are present. Report missing items; user decides go/no-go.

1. **Specification Analysis** — the `analyst` extracts all 10 config keys with types, defaults, validation bounds, coercion rules, the 4-source priority cascade, and the API contract into a structured matrix. See [workflow.md](workflow.md) Step 1.

2. **Implementation** — the `developer` implements `config_system.py` from the Analyst's matrix, including `ConfigValidationError`, `get_schema()`, `validate_value()`, and `load_config()`. Runs 3 smoke tests before handoff. See [workflow.md](workflow.md) Step 2.

3. **Compliance Verification** — the `verifier` independently designs a test suite from the spec, covering boundary values, all 8 bool coercion forms, priority cascade scenarios, and error paths. Produces ATTEST or REJECT verdict. On REJECT, kicks back to Step 2 for up to 2 cycles. See [workflow.md](workflow.md) Step 3 and [bind.md](bind.md) § Failure Handling.

4. **Final: emit Configuration System Report** — the Leader composes all 3 stage outputs into a report with a binary ATTEST/REJECT verdict and per-key traceability. Contradictions surfaced verbatim, never mediated.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| analyst | Extract all 10 keys, coercion rules, priority cascade, API contract | Every run, first stage | Full spec (TASK.md) | none | [roles/analyst.md](roles/analyst.md) |
| developer | Implement config_system.py with all functions | After analyst gate passes | Analyst requirements matrix, workspace path | python | [roles/developer.md](roles/developer.md) |
| verifier | Independently test every rule, produce ATTEST/REJECT | After developer gate passes | Full spec, config_system.py | python | [roles/verifier.md](roles/verifier.md) |

> Before dispatching each teammate, read the corresponding role file and extract the `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol with gates, Final Report format | Before first dispatch — the complete playbook |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits or handling failures |
| [roles/\*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona for Teammate | Before dispatching each teammate — extract Inline Persona |
| [dependencies.yaml](dependencies.yaml) | External tools required to run | **Startup** — verify deps, report missing items, user decides go/no-go |
