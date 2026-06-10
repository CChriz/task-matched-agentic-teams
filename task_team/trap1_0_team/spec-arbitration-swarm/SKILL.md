---
name: spec-arbitration-swarm
description: |
  3-stage pipeline (Conflict Analyst → Fix Developer → Compliance Verifier) that resolves conflicting requirements across spec documents using priority rules with documented exceptions, and independently verifies every resolved value.
  Use when multiple specs conflict on configuration keys and a documented priority system with exceptions must produce a single resolved config.
  Do NOT use for single-spec configs, conflicts without documented rules, or already-verified output.
version: "0.1"
kind: swarm-skill
roles:
  - id: conflict-analyst
    kind: ai_agent
    purpose: "Maps every endpoint conflict between OpenAPI spec and CHANGELOG, categorizes 4 STRICT vs 3 RELAXED per the authoritative CHANGELOG."
    skills: []
    tools: []
  - id: fix-developer
    kind: ai_agent
    purpose: "Adds strict validation to STRICT endpoints only, preserves relaxed behavior on RELAXED endpoints, uses 422 error format."
    skills: []
    tools: [python3, pytest]
  - id: compliance-verifier
    kind: ai_agent
    purpose: "Independently tests every endpoint: STRICT endpoints reject malformed inputs with 422, RELAXED endpoints accept partner inputs."
    skills: []
    tools: [python3, pytest]
---

# Spec Arbitration Swarm

A mixed C+A team for resolving conflicting requirements across spec documents using documented priority rules. Solves the failure mode where a single agent blindly follows one spec (breaking backward compatibility) or incorrectly resolves which spec is authoritative — the compliance verifier independently tests every endpoint against the resolved rules.

## Workflow

0. **Pre-flight** — verify python3, pytest, and workspace files.

1. **Stage 1: Conflict Analysis** — dispatch `conflict-analyst` with api_spec.yaml + CHANGELOG.md. Maps all 7 endpoints, documents conflicts, categorizes STRICT vs RELAXED per CHANGELOG authority rule.

2. **Stage 2: Apply Validation** — dispatch `fix-developer` with categorization. Adds 422 validation to 4 STRICT endpoints, leaves 3 RELAXED untouched, runs pytest.

3. **Stage 3: Compliance Verification** — dispatch `compliance-verifier` with endpoint list. Independently tests every endpoint, verifies strict/relaxed behavior, runs pytest.

4. **Final Report** — Leader composes from all outputs.

## Roles

| id | Purpose | Input | Role file |
|---|---|---|---|
| conflict-analyst | Map spec-vs-changelog conflicts, categorize endpoints | api_spec.yaml + CHANGELOG.md | [roles/conflict-analyst.md](roles/conflict-analyst.md) |
| fix-developer | Apply validation per categorization, run tests | Analyst categorization + codebase | [roles/fix-developer.md](roles/fix-developer.md) |
| compliance-verifier | Independently test every endpoint | Endpoint list + codebase | [roles/compliance-verifier.md](roles/compliance-verifier.md) |

## Files

| File | What it contains |
|---|---|
| [workflow.md](workflow.md) | Mermaid, steps, gates, Final Report format |
| [bind.md](bind.md) | Resource limits, authority rule, failure handling |
| [roles/*.md](roles/) | Per-role identity, criteria, schema, Inline Persona |
| [dependencies.yaml](dependencies.yaml) | python3 + pytest |
