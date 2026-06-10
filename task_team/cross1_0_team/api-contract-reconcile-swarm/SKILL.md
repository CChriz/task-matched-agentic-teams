---
name: api-contract-reconcile-swarm
description: |
  3-stage pipeline (Analyze → Fix → Verify) that reconciles API contract mismatches between a Go server, Python client, and OpenAPI spec by cross-reading all three artifacts with the server as source of truth.
  Use when fixing polyglot API contract mismatches where a server and client disagree on field names, response shapes, or error formats, and the spec is also out of date.
  Do NOT use for single-language code review, greenfield API design, or tasks where the server can be modified.
version: "0.1"
kind: swarm-skill
roles:
  - id: contract-analyzer
    kind: ai_agent
    purpose: "Finds every contract mismatch by cross-reading Go server, Python client, and spec; server wire behavior is truth; outputs file:line evidence."
    skills: []
    tools: []
  - id: client-fixer
    kind: ai_agent
    purpose: "Applies prescribed fixes to Python client and api_spec.yaml from analyzer report; never touches Go; records before/after per change."
    skills: []
    tools: []
  - id: contract-verifier
    kind: ai_agent
    purpose: "Runs pytest and go build, audits every mismatch from the analyzer against the fixer's changes, and delivers a PASS/FAIL verdict with evidence."
    skills: []
    tools: [python3, go, git]
---

# API Contract Reconcile Swarm

A 3-stage specialization pipeline that fixes API contract mismatches in polyglot microservices. Solves the single-agent failure mode where a developer reads only one side of the contract, fixes the most visible mismatch, and misses the others — or skips verification entirely. By enforcing Analyze → Fix → Verify as sequential gates, every mismatch is found, fixed, and proven resolved before the pipeline completes.

## Workflow

0. **Pre-flight: workspace validation** — verify the workspace contains `service/`, `client/`, `api_spec.yaml`, `tests/`. Report missing components. See [dependencies.yaml](dependencies.yaml) for tool checks; user decides go/no-go.

1. **Stage 1: Analyze** — dispatch `contract-analyzer` with the Go server source, Python client source, and `api_spec.yaml`. The analyzer cross-reads all three, treating the server's struct tags and response construction as source of truth. Outputs a structured mismatch report with file:line evidence and fix prescriptions. See [workflow.md](workflow.md) Step 1 for the quality gate.

2. **Stage 2: Fix** — dispatch `client-fixer` with the analyzer's mismatch report and the workspace files. Applies each prescribed fix to the Python client and `api_spec.yaml`. Never touches `.go` files. Records every change with before/after snippets. See [workflow.md](workflow.md) Step 2 for the quality gate.

3. **Stage 3: Verify** — dispatch `contract-verifier` with the analyzer's report, fixer's report, and the workspace path. Runs `pytest tests/` and `go build ./...`, then audits every mismatch to confirm resolution. Outputs a PASS/FAIL verdict. See [workflow.md](workflow.md) Step 3.

4. **Final: emit Reconciliation Report** — the Leader composes a unified report from all 3 stage outputs. The Leader never performs analysis, fixing, or verification — only integration and formatting. See [workflow.md](workflow.md) Step 4 for the report template.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| contract-analyzer | Cross-read server, client, spec to find mismatches with file:line evidence | Stage 1 — always first | Go server source, Python client source, api_spec.yaml | None (LLM-native) | [roles/contract-analyzer.md](roles/contract-analyzer.md) |
| client-fixer | Apply prescribed fixes from analyzer report to Python files and spec | Stage 2 — after analyzer passes quality gate | Analyzer mismatch report + workspace files | None (LLM-native) | [roles/client-fixer.md](roles/client-fixer.md) |
| contract-verifier | Run tests, check compilation, audit fix completeness | Stage 3 — after fixer passes quality gate | Analyzer report + fixer report + workspace path | python3 (optional), go (optional), git (optional) | [roles/contract-verifier.md](roles/contract-verifier.md) |

> Before dispatching each teammate, read the corresponding role file and extract the
> `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt.
> Most adopting agents do NOT auto-load role files for teammates.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol with quality gates, Final Report format | Before first dispatch — the complete playbook |
| [bind.md](bind.md) | Resource limits, behavioral constraints (server immutability, stage boundaries), failure handling and degraded modes | When hitting limits, handling failures, or needing degraded-mode rules |
| [roles/\*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona for Teammate | Before dispatching each teammate — extract Inline Persona |
| [dependencies.yaml](dependencies.yaml) | External tools required to run (python3, go, git — all optional) | **Startup** — verify deps, report missing items, user decides go/no-go |
