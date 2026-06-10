---
name: offline-retrieval-swarm
description: |
  3-role adversarial retrieval team (2 parallel researchers + evidence verifier) that cross-verifies answers from an offline corpus and detects misinformation traps by isolating one researcher from the trap document.
  Use when answering questions from a static document corpus that may contain deliberately misleading sources, and cross-verification against independent documents is required.
  Do NOT use for live web research, single-document Q&A, or corpora without known misinformation traps.
version: "0.1"
kind: swarm-skill
roles:
  - id: corpus-researcher-1
    kind: ai_agent
    purpose: "Reads only trusted documents (doc_A, doc_B) to provide an uncontaminated baseline answer with verbatim evidence."
    skills: []
    tools: []
  - id: corpus-researcher-2
    kind: ai_agent
    purpose: "Reads all documents including the trap to detect misinformation, finds truth from trusted sources, flags conflicting claims."
    skills: []
    tools: []
  - id: evidence-verifier
    kind: ai_agent
    purpose: "Cross-references both researchers' findings against source documents, enforces evidence rules, and produces answer.json + attestation.json."
    skills: []
    tools: []
---

# Offline Retrieval Swarm

A mixed B+A (parallel research + adversarial cross-check) team for answering questions from an offline document corpus with misinformation traps. Solves the failure mode where a single agent trusts the first source it finds — including deliberately misleading documents — because it has no independent cross-check. By isolating one researcher from the trap document, the verifier gets an uncontaminated baseline to detect deception.

## Workflow

0. **Pre-flight** — verify corpus files exist at {CORPUS_PATH}. Confirm offline-only execution.

1. **Stage 1: Parallel Research (isolated)** — dispatch `corpus-researcher-1` (reads doc_A + doc_B only) and `corpus-researcher-2` (reads all 3 docs including trap) **in parallel**. Both produce independent findings with verbatim evidence. See role files.

2. **Stage 2: Evidence Verification** — dispatch `evidence-verifier` with both research reports. Cross-references findings against source documents, enforces rules (2 evidence items, 2 different docs, no doc_trap.txt), produces answer.json and attestation.json. See [roles/evidence-verifier.md](roles/evidence-verifier.md).

3. **Final: Offline Retrieval Report** — Leader composes from all outputs. See [workflow.md](workflow.md).

## Roles

| id | Purpose | When dispatched | Input | Role file |
|---|---|---|---|---|
| corpus-researcher-1 | Uncontaminated baseline from trusted docs only | Parallel, Stage 1 | doc_A, doc_B, question | [roles/corpus-researcher-1.md](roles/corpus-researcher-1.md) |
| corpus-researcher-2 | Trap-aware reading of all docs, flags deception | Parallel, Stage 1 | all 3 docs, question | [roles/corpus-researcher-2.md](roles/corpus-researcher-2.md) |
| evidence-verifier | Cross-check + enforce rules + produce outputs | Stage 2 | both reports, corpus, question | [roles/evidence-verifier.md](roles/evidence-verifier.md) |

> Before dispatching, extract `## Inline Persona for Teammate` from the role file.

## Files

| File | What it contains | When to read |
|---|---|---|
| [workflow.md](workflow.md) | Mermaid, steps, gates, Final Report format | Before first dispatch |
| [bind.md](bind.md) | Resource limits, isolation rules, trap enforcement | When hitting limits |
| [roles/*.md](roles/) | Per-role identity, criteria, schema, Inline Persona | Before dispatching |
| [dependencies.yaml](dependencies.yaml) | Empty — no external tools needed | Startup |
