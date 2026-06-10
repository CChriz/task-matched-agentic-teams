---
name: jwt-security-fix-swarm
description: |
  3-stage pipeline (Security Analyst → Fix Developer → Adversarial Verifier) that fixes JWT authentication vulnerabilities with independent adversarial verification to prevent fixer blind spots.
  Use when fixing known security bugs in JWT token validation or authentication code where the fixes must survive adversarial attack, and an independent "attacker" perspective is critical.
  Do NOT use for greenfield JWT implementation, feature additions, or bugs without a documented vulnerability list.
version: "0.1"
kind: swarm-skill
roles:
  - id: security-analyst
    kind: ai_agent
    purpose: "Confirms each listed security bug in source code, maps to code locations, produces a severity-ordered fix checklist with additional findings."
    skills: []
    tools: []
  - id: fix-developer
    kind: ai_agent
    purpose: "Implements each fix from the analyst checklist as minimal surgical changes, verifies tests pass after every fix, and produces before/after docs."
    skills: []
    tools: [python3, pytest]
  - id: adversarial-verifier
    kind: ai_agent
    purpose: "Designs and executes concrete attacks against each fix to independently verify they hold, plus extended edge-case attacks, with PASS/FAIL per bug."
    skills: []
    tools: [python3, pytest]
---

# JWT Security Fix Swarm

A mixed C+A (pipeline + adversarial gate) team for fixing JWT token validation vulnerabilities. Solves the failure mode where a single developer fixing their own bugs shares the same blind spots as the original author — the adversarial verifier provides an independent "attacker" perspective that catches what both the original developer and the fixer miss.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify `python3` and `pytest` are available. Report missing items: `required: true` = likely fails without it. **User decides** whether to proceed. If `pytest` is missing, warn that test execution will be skipped and the Final Report will be tagged accordingly.

1. **Stage 1: Security Analysis** — dispatch `security-analyst` with the task spec ({TASK_SPEC}) and codebase path ({CODEBASE_PATH}). The analyst reads all source files, confirms each listed bug, identifies additional vulnerabilities, and produces a severity-ordered fix checklist. See [roles/security-analyst.md](roles/security-analyst.md). Quality gate: Analyst Verdict must be GO. Max 1 retry on malformed output.

2. **Stage 2: Fix Implementation** — dispatch `fix-developer` with the analyst's checklist ({ANALYST_CHECKLIST}) and the codebase. The developer applies each fix in severity order, runs `pytest` after each fix, and produces FIXES_APPLIED.md. See [roles/fix-developer.md](roles/fix-developer.md). Quality gate: all tests pass. Max 2 retries; escalate on 3rd failure.

3. **Stage 3: Adversarial Verification** — dispatch `adversarial-verifier` with the original task spec ({TASK_SPEC}), the developer's FIXES_APPLIED.md, and the fixed codebase. The verifier designs and executes attacks against each fix, attempts extended edge-case attacks, and runs `pytest` independently. See [roles/adversarial-verifier.md](roles/adversarial-verifier.md). Quality gate: Verdict must be SHIP. Max 2 full kick-back cycles (verifier → developer → verifier → developer). On 3rd verifier FAIL, escalate with full evidence chain.

4. **Final: emit JWT Security Fix Report** — Leader composes the Final Report from all three role outputs verbatim. Contradictions between stages are surfaced, never mediated. See [workflow.md](workflow.md) for the full step protocol, mermaid diagram, and Final Report format.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| security-analyst | Confirm all 6 bugs and find additional vulnerabilities | Every run, Stage 1 | Task spec + full source code | none (inline persona) | [roles/security-analyst.md](roles/security-analyst.md) |
| fix-developer | Implement fixes as minimal surgical changes, keep tests green | After analyst GO verdict, Stage 2 | Analyst fix checklist + source code + tests | python3, pytest | [roles/fix-developer.md](roles/fix-developer.md) |
| adversarial-verifier | Independently attack fixes to verify they hold | After developer READY-FOR-VERIFICATION, Stage 3 | Task spec + FIXES_APPLIED.md + fixed source + tests | python3, pytest | [roles/adversarial-verifier.md](roles/adversarial-verifier.md) |

> Before dispatching each teammate, read the corresponding role file and extract the
> `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt.
> Most adopting agents do NOT auto-load role files for teammates.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol, quality gates, Final Report format | Before first dispatch — the complete playbook |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits, handling failures, or needing degraded-mode rules |
| [roles/*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona for Teammate | Before dispatching each teammate — extract Inline Persona |
| [dependencies.yaml](dependencies.yaml) | External tools required to run (python3, pytest) | **Startup** — verify deps, report missing items, user decides go/no-go |
