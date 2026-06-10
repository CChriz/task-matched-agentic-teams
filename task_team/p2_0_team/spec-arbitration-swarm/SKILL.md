---
name: spec-arbitration-swarm
description: |
  3-stage pipeline (Conflict Analyst → Priority Arbiter → Arbitration Verifier) that resolves conflicting requirements across spec documents using priority rules with documented exceptions, and independently verifies every resolved value.
  Use when multiple specs conflict on configuration keys and a documented priority system with exceptions must produce a single resolved config.
  Do NOT use for single-spec configs, conflicts without documented rules, or already-verified output.
version: "0.1"
kind: swarm-skill
roles:
  - id: analyzer
    kind: ai_agent
    purpose: "Catalogue every config key from all specs, build a complete conflict matrix with priority classes, and quote all rules and special cases verbatim."
    skills: []
    tools: []
  - id: arbiter
    kind: ai_agent
    purpose: "Apply priority rules to resolve every conflict, handle documented exceptions like the compression special case, and produce resolved_config.json."
    skills: []
    tools: [python]
  - id: verifier
    kind: ai_agent
    purpose: "Independently re-derive every expected value from the corpus, verify resolved_config.json key by key, and produce binary ATTEST or REJECT verdict."
    skills: []
    tools: [python, jq]
---

# Spec Arbitration Swarm

A 3-stage specialization pipeline that resolves conflicting requirements across multiple specification documents. The single-agent failure mode this solves: a solo resolver reads the specs, mentally applies priority rules, and produces a config — but misses a conflict, misapplies the priority chain on a same-class tie, or overlooks a documented exception like the `compression` legacy-compatibility footnote. The Verifier independently re-derives every value to catch these errors.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify corpus directory contains `requirements.txt` and `priority_rules.txt`. `python` and `jq` are optional for validation. Report missing items; user decides go/no-go.

1. **Conflict Analysis** — the `analyzer` reads all spec documents in the corpus, catalogues every key with its value and priority class, builds a conflict matrix, and quotes all priority rules and special cases verbatim. See [workflow.md](workflow.md) Step 1.

2. **Priority Arbitration** — the `arbiter` takes the Analyzer's conflict matrix, applies priority rules to every conflict, handles the `compression` special case, includes pass-through keys, and writes `output/resolved_config.json` with exactly 8 keys. See [workflow.md](workflow.md) Step 2.

3. **Arbitration Verification** — the `verifier` independently re-reads the corpus, re-derives every expected value, and verifies `resolved_config.json` key by key. Verdict is ATTEST (all pass) or REJECT (with violations). On REJECT, pipeline kicks back to Step 2 for up to 2 cycles. See [workflow.md](workflow.md) Step 3 and [bind.md](bind.md) § Failure Handling.

4. **Final: emit Arbitration Report** — the Leader composes the Analyzer's conflict matrix, Arbiter's resolution log, and Verifier's attestation into a single report with a binary ATTEST/REJECT verdict and per-key evidence. Contradictions are surfaced verbatim, never mediated.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| analyzer | Catalogue all keys, build conflict matrix, quote rules | Every run, first stage | Corpus (requirements.txt, priority_rules.txt), spec | none | [roles/analyzer.md](roles/analyzer.md) |
| arbiter | Apply rules, resolve conflicts, produce config | After analyzer gate passes | Analyzer report, workspace path | python (optional) | [roles/arbiter.md](roles/arbiter.md) |
| verifier | Independently verify every key, produce ATTEST/REJECT | After arbiter gate passes | Corpus, resolved_config.json, spec | python, jq (optional) | [roles/verifier.md](roles/verifier.md) |

> Before dispatching each teammate, read the corresponding role file and extract the `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol with gates, Final Report format | Before first dispatch — the complete playbook |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits, handling failures, or needing degraded-mode rules |
| [roles/\*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona for Teammate | Before dispatching each teammate — extract Inline Persona |
| [dependencies.yaml](dependencies.yaml) | External tools required to run | **Startup** — verify deps, report missing items, user decides go/no-go |
