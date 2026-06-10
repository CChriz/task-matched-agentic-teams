---
name: data-reconcile-swarm
description: |
  3-stage pipeline (Analyzer → Developer → Verifier) that fixes broken data reconciliation scripts by tracing spec rules to code, implementing fixes, and adversarially verifying correctness.
  Use when a reconciliation script must be fixed against a spec with field ownership and conflict resolution rules, and correctness requires independent verification.
  Do NOT use for greenfield scripts, tasks without a formal spec, or already-verified fixes.
version: "0.1"
kind: swarm-skill
roles:
  - id: analyzer
    kind: ai_agent
    purpose: "Trace all 6 reconciliation rules from spec to code, produce a severity-ranked bug report with exact line references and output field coverage gaps."
    skills: []
    tools: []
  - id: developer
    kind: ai_agent
    purpose: "Fix every bug from the Analyzer's report, run reconcile.py, and produce valid reconciled.json — no scope creep, no field-ownership constant changes."
    skills: []
    tools: [python]
  - id: verifier
    kind: ai_agent
    purpose: "Adversarially verify all 6 rules against reconciled.json with field-level walkthroughs; produce binary ATTEST or REJECT verdict."
    skills: []
    tools: [python, jq]
---

# Data Reconciliation Swarm

A 3-stage specialization pipeline that fixes broken data reconciliation scripts by enforcing a disciplined handoff: the Analyzer finds what's wrong, the Developer fixes what's wrong, and the Verifier independently confirms nothing is still wrong. The single-agent failure mode this solves: a solo developer writes a fix, tests with their own mental model of the spec, and ships bugs that an adversarial second set of eyes would catch.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify `python` is available (required). `jq` is optional for enhanced verification. Report missing items; user decides go/no-go.

1. **Requirements Analysis** — the `analyzer` reads the spec, `reconcile.py`, and data files, producing a bug report with rule traceability, severity-ranked bugs, and output field coverage. See [workflow.md](workflow.md) Step 1 for the full gate criteria.

2. **Code Fix and Execution** — the `developer` takes the Analyzer's bug report, fixes `reconcile.py`, runs the script, and produces `reconciled.json`. See [workflow.md](workflow.md) Step 2.

3. **Compliance Verification** — the `verifier` independently checks every record in `reconciled.json` against all 6 rules. Verdict is ATTEST (all pass) or REJECT (with violations). On REJECT, pipeline kicks back to Step 2 for up to 2 cycles. See [workflow.md](workflow.md) Step 3 and [bind.md](bind.md) § Failure Handling.

4. **Final: emit Reconciliation Report** — the Leader composes the Analyzer's findings, Developer's fixes, and Verifier's attestation into a single report with a binary ATTEST/REJECT verdict. Contradictions between Verifier and Developer are surfaced verbatim, never mediated.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| analyzer | Trace all 6 rules from spec to code, produce severity-ranked bug report | Every run, first stage | Spec, reconcile.py, system_a.json, system_b.json | none | [roles/analyzer.md](roles/analyzer.md) |
| developer | Fix bugs, run script, produce reconciled.json | After analyzer gate passes | Analyzer bug report, reconcile.py, workspace path | python | [roles/developer.md](roles/developer.md) |
| verifier | Adversarially verify all 6 rules, produce ATTEST/REJECT | After developer gate passes | Spec, reconciled.json, system_a.json, system_b.json | python, jq (optional) | [roles/verifier.md](roles/verifier.md) |

> Before dispatching each teammate, read the corresponding role file and extract the `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt. Most adopting agents do NOT auto-load role files for teammates.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol with gates, Final Report format | Before first dispatch — the complete playbook |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits, handling failures, or needing degraded-mode rules |
| [roles/\*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona for Teammate | Before dispatching each teammate — extract Inline Persona |
| [dependencies.yaml](dependencies.yaml) | External tools required to run | **Startup** — verify deps, report missing items, user decides go/no-go |
