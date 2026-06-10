---
name: queue-race-fix-swarm
description: |
  3-stage pipeline (Concurrency Analyst → Fix Developer → Adversarial Tester) that fixes message queue race conditions with independent concurrent stress testing.
  Use when fixing known concurrency bugs in Python message queues where races cause message loss or crashes, and adversarial stress testing is critical.
  Do NOT use for greenfield queue implementation, single-threaded bugs, or undocumented race conditions.
version: "0.1"
kind: swarm-skill
roles:
  - id: concurrency-analyst
    kind: ai_agent
    purpose: "Confirms each listed race condition in source code, maps interleaving timelines, and produces a dependency-ordered fix checklist with ack/nack design."
    skills: []
    tools: []
  - id: fix-developer
    kind: ai_agent
    purpose: "Implements each fix from the analyst checklist as minimal locking changes, verifies all tests pass after every fix, and produces before/after docs."
    skills: []
    tools: [python3, pytest]
  - id: adversarial-tester
    kind: ai_agent
    purpose: "Designs and executes concurrent stress tests against each fix, simulates consumer crashes, and provides clear PASS/FAIL per race condition."
    skills: []
    tools: [python3, pytest]
---

# Queue Race Fix Swarm

A mixed C+A (pipeline + adversarial gate) team for fixing message queue race conditions. Solves the failure mode where a single developer fixing concurrency bugs cannot imagine all interleavings — the adversarial tester provides an independent worst-case scheduler perspective that finds synchronization gaps both the original author and the fixer missed.

## Workflow

0. **Pre-flight: check dependencies** — read [dependencies.yaml](dependencies.yaml) and verify `python3` and `pytest` are available. Report missing items; **user decides** go/no-go. If `pytest` missing, warn that tests cannot run.

1. **Stage 1: Concurrency Analysis** — dispatch `concurrency-analyst` with task spec ({TASK_SPEC}) and codebase ({CODEBASE_PATH}). The analyst reads all source and test files, confirms each race condition, maps interleaving timelines, and designs the ack/nack pattern contract. See [roles/concurrency-analyst.md](roles/concurrency-analyst.md). Quality gate: Analyst Verdict must be GO.

2. **Stage 2: Fix Implementation** — dispatch `fix-developer` with analyst's checklist ({ANALYST_CHECKLIST}) and codebase. The developer applies each fix in dependency order, runs `pytest` after each fix, verifies the 10K stress test passes, and produces FIXES_APPLIED.md. See [roles/fix-developer.md](roles/fix-developer.md). Quality gate: all tests pass + 10K stress test passes. Max 2 retries.

3. **Stage 3: Adversarial Stress Testing** — dispatch `adversarial-tester` with task spec ({TASK_SPEC}), FIXES_APPLIED.md, and fixed codebase. The tester designs concurrent stress tests per bug, executes them, simulates consumer crashes for nack/re-delivery, and runs tests independently. See [roles/adversarial-tester.md](roles/adversarial-tester.md). Quality gate: Verdict must be SHIP. Max 2 kick-back cycles.

4. **Final: emit Queue Race Fix Report** — Leader composes the Final Report from all three role outputs verbatim. Contradictions surfaced, never mediated. See [workflow.md](workflow.md) for the full protocol, mermaid diagram, and Final Report format.

## Roles

| id | Purpose | When dispatched | Input | Key dependencies | Role file |
|---|---|---|---|---|---|
| concurrency-analyst | Confirm 3 race conditions, design ack/nack pattern | Every run, Stage 1 | Task spec + full source + tests | none (inline persona) | [roles/concurrency-analyst.md](roles/concurrency-analyst.md) |
| fix-developer | Implement fixes with lock-discipline, keep tests green | After analyst GO, Stage 2 | Analyst checklist + source + tests | python3, pytest | [roles/fix-developer.md](roles/fix-developer.md) |
| adversarial-tester | Independently stress-test fixes under concurrent load | After developer READY, Stage 3 | Task spec + FIXES_APPLIED.md + fixed source + tests | python3, pytest | [roles/adversarial-tester.md](roles/adversarial-tester.md) |

> Before dispatching each teammate, read the corresponding role file and extract the `## Inline Persona for Teammate` section — paste it directly into the dispatch prompt. Most adopting agents do NOT auto-load role files for teammates.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid diagram, step-by-step protocol, quality gates, Final Report format | Before first dispatch |
| [bind.md](bind.md) | Resource limits, behavioral constraints, failure handling and degraded modes | When hitting limits or handling failures |
| [roles/*.md](roles/) | Per-role identity, success criteria, output schema, Inline Persona | Before dispatching each teammate |
| [dependencies.yaml](dependencies.yaml) | External tools required (python3, pytest) | Startup — verify deps, user decides go/no-go |
