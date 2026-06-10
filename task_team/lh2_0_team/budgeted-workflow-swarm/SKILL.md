---
name: budgeted-workflow-swarm
description: |
  3-stage pipeline (Workflow Analyzer → Fix Developer → Budget Auditor) that fixes data-processing scripts and data files within a strict command budget, with independent budget attestation.
  Use when fixing bugs in a script that processes data files under a hard command-execution budget, where independent budget counting is required.
  Do NOT use for simple script fixes without budget constraints, or tasks where the validator/auditor role is unnecessary.
version: "0.1"
kind: swarm-skill
roles:
  - id: workflow-analyzer
    kind: ai_agent
    purpose: "Traces code vs spec to map every defect, identifies staging bugs, and proposes a minimal-command fix strategy under the budget."
    skills: []
    tools: []
  - id: fix-developer
    kind: ai_agent
    purpose: "Fixes all bugs in one pass — corrects fix_file logic and data files, runs workflow, confirms budget ≤28 and validation passes."
    skills: []
    tools: [python3]
  - id: budget-auditor
    kind: ai_agent
    purpose: "Runs fixed workflow from clean state, independently counts every command, verifies validation, and produces signed attestation."
    skills: []
    tools: [python3]
---

# Budgeted Workflow Swarm

A mixed C+A team for fixing data-processing scripts under a hard command-execution budget. Solves the failure mode where a single agent fixing bugs under a budget constraint may miscount or skip independent verification — the budget auditor provides clean-room attestation.

## Workflow

0. **Pre-flight** — verify `python3` and workspace. User decides go/no-go.

1. **Stage 1: Workflow Analysis** — dispatch `workflow-analyzer` with task spec and workspace. Analyzes code vs data, maps every defect, proposes minimal-command fix strategy. See [roles/workflow-analyzer.md](roles/workflow-analyzer.md).

2. **Stage 2: Fix Implementation** — dispatch `fix-developer` with analyst's strategy. Fixes all bugs in one pass, deletes stale state, runs workflow, confirms budget ≤28 and validation passes. See [roles/fix-developer.md](roles/fix-developer.md).

3. **Stage 3: Budget Audit** — dispatch `budget-auditor` with fixed workspace. Clean-room start, independent budget count, validation verification, attestation. See [roles/budget-auditor.md](roles/budget-auditor.md).

4. **Final: Budgeted Workflow Report** — Leader composes from all three outputs. See [workflow.md](workflow.md).

## Roles

| id | Purpose | When dispatched | Input | Key deps | Role file |
|---|---|---|---|---|---|
| workflow-analyzer | Map defects, propose budget-conscious fix strategy | Stage 1 | Task spec + workspace | none | [roles/workflow-analyzer.md](roles/workflow-analyzer.md) |
| fix-developer | Fix all bugs in one pass, confirm budget | Stage 2 | Analyst strategy + workspace | python3 | [roles/fix-developer.md](roles/fix-developer.md) |
| budget-auditor | Clean-room audit: count budget, verify validation, attest | Stage 3 | Fixed workspace | python3 | [roles/budget-auditor.md](roles/budget-auditor.md) |

> Before dispatching, extract `## Inline Persona for Teammate` from the role file.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid, steps, gates, Final Report format | Before first dispatch |
| [bind.md](bind.md) | Resource limits, budget discipline, failure handling | When hitting limits |
| [roles/*.md](roles/) | Per-role identity, criteria, schema, Inline Persona | Before dispatching |
| [dependencies.yaml](dependencies.yaml) | python3 requirement | Startup |
