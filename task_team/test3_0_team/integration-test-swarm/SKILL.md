---
name: integration-test-swarm
description: |
  3-stage pipeline (Contract Analyst → Test Author → Adversarial Validator) that writes comprehensive integration tests from an API contract with independent mutation-detection validation.
  Use when generating integration tests against a documented REST API where tests must verify contract compliance and catch server-side regressions.
  Do NOT use for unit tests, load tests, or APIs without a formal contract document.
version: "0.1"
kind: swarm-skill
roles:
  - id: contract-analyst
    kind: ai_agent
    purpose: "Extracts complete test-case matrix from the API spec covering all endpoints, status codes, schema fields, auth, and boundary conditions."
    skills: []
    tools: []
  - id: test-author
    kind: ai_agent
    purpose: "Writes complete pytest integration tests from the analyst checklist, verifies all pass against the working server, ensures schema field assertions."
    skills: []
    tools: [python3, pytest, flask, requests]
  - id: adversarial-validator
    kind: ai_agent
    purpose: "Runs tests against working and broken servers for mutation detection, audits grading check coverage, identifies structural weaknesses."
    skills: []
    tools: [python3, pytest, flask, requests]
---

# Integration Test Swarm

A mixed C+A (pipeline + adversarial gate) team for writing integration tests from an API contract. Solves the failure mode where a single developer writing tests shares the same blind spots as the API designer — the adversarial validator independently verifies that tests catch contract violations, not just that they pass against today's implementation.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify `python3`, `pytest`, `flask`, and `requests` are available. Verify `server.py` can start. Report missing items; **user decides** go/no-go.

1. **Stage 1: Contract Analysis** — dispatch `contract-analyst` with API spec ({TASK_SPEC}) and server code ({CODEBASE_PATH}). The analyst reads the full spec and server, produces a test-case matrix covering all 5 endpoints with status codes, schema fields, auth, and boundaries. See [roles/contract-analyst.md](roles/contract-analyst.md). Quality gate: Analyst Verdict must be GO with ≥9 test functions.

2. **Stage 2: Test Authoring** — dispatch `test-author` with analyst's checklist ({ANALYST_CHECKLIST}) and server code. The author writes `tests/test_integration.py`, runs `pytest` against the working server, and confirms all tests pass. See [roles/test-author.md](roles/test-author.md). Quality gate: all tests pass against working server. Max 2 retries.

3. **Stage 3: Adversarial Validation** — dispatch `adversarial-validator` with grading criteria ({TASK_SPEC}), test file, and both servers. The validator runs tests against working AND broken servers, audits all 12 grading checks, and identifies structural weaknesses. See [roles/adversarial-validator.md](roles/adversarial-validator.md). Quality gate: Verdict must be SHIP (mutation detected, grading covered). Max 2 kick-back cycles.

4. **Final: emit Integration Test Report** — Leader composes the Final Report from all three role outputs verbatim. Contradictions surfaced, never mediated. See [workflow.md](workflow.md) for full protocol.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| contract-analyst | Map API spec to complete test-case matrix | Every run, Stage 1 | API spec + server code + test skeleton | none (inline persona) | [roles/contract-analyst.md](roles/contract-analyst.md) |
| test-author | Write pytest integration tests, verify against working server | After analyst GO, Stage 2 | Analyst checklist + server code + skeleton | python3, pytest, flask, requests | [roles/test-author.md](roles/test-author.md) |
| adversarial-validator | Audit tests for mutation detection and grading coverage | After author READY, Stage 3 | Grading criteria + test file + both servers | python3, pytest, flask, requests | [roles/adversarial-validator.md](roles/adversarial-validator.md) |

> Before dispatching each teammate, read the corresponding role file and extract the `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol, quality gates, Final Report format | Before first dispatch |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits or handling failures |
| [roles/*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona | Before dispatching each teammate |
| [dependencies.yaml](dependencies.yaml) | External tools required (python3, pytest, flask, requests) | Startup — verify deps, user decides go/no-go |
