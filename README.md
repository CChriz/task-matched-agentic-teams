# Task-Matched Agentic Teams

Task-Matched Agentic Teams is an experimental artefact repository for evaluating how agent-team configurations perform on standalone task instances exported from [TeamBench](https://github.com/ybkim95/TeamBench).

The tasks in this repository are extracted from TeamBench using an extended standalone task export pipeline, specifically [`export_standalone_tasks.py`](https://github.com/CChriz/TeamBench-task-export/blob/main/scripts/export_standalone_tasks.py) from [CChriz/TeamBench-task-export](https://github.com/CChriz/TeamBench-task-export).


This repository is intended to make TeamBench-derived task instances easier to inspect, run, and compare against task-matched multi-agent-team outputs.

## Provenance

- **Tasks:** [ybkim95/TeamBench](https://github.com/ybkim95/TeamBench)
- **Export script:** [`scripts/export_standalone_tasks.py`](https://github.com/CChriz/TeamBench-task-export/blob/main/scripts/export_standalone_tasks.py)

The exported tasks are standalone tasks, prepared so they can be run, inspected, and evaluated outside the original TeamBench execution environment.

## Repo Structure (tbu)

```text
.
├── evals/
|   |
│   ├── d6_0_eval/
│   │   └── _eval/
│   │       ├── grade.sh
│   │       ├── task.yaml
│   │       └── task_dir/
│   └── ...
|
├── task_team/
|   |
│   ├── d6_0_team/
│   │   ├── D6_data_reconcile__seed_0/
│   │   └── data-reconcile-swarm/
|   |   
│   ├── cross1_0_team/
│   │   ├── CROSS1_api_contract_seed_0/
│   │   └── api-contract-reconcile-swarm/
|   |
│   └── ...
└── README.md
```

## `task_team/`

The `task_team/` directory contains task-specific team artefacts and generated team workspaces.

Each task family has a `*_team/` directory. Inside that, the standalone exported task and the generated team workspace sit side by side.

Example layout:

```text
task_team/
|
└── d6_0_team/
    |
    ├── D6_data_reconcile__seed_0/
    │   ├── task.yaml
    │   ├── input files...
    │   └── expected task workspace...
    |   
    └── data-reconcile-swarm/
        ├── team configuration / agent workspace files...
        ├── generated outputs...
        └── evaluation or run artefacts...

```

## Envisioned Pipeline

### Task Extraction & Team Creation

1. **Specify task ID and seed**  
   Select the task instance to prepare.

2. **Extract as standalone task folder**  
   Generate / copy the task into a self-contained folder.

3. **Separate visible task files from hidden eval files**  
   `<standalone_task>/` → `<task_visible>/` + `evals/`

4. **Create a task-specific team**  
   Pass `TASK.md` and `workspace/` to the team-creator.

5. **Bundle task and team together**  
   `<task>/` + `<team>.json` → `task_team/`

### Task Execution

1. Start from clean task package
   `task_team/`

2. Copy into isolated agent workspace
   `task_team/` → `agent_modified/`

3. Run agent team on `agent_modified/`
   agents modify files in-place

4. Freeze final workspace
   no more agent edits

### Evaluation

1. Combine final workspace with hidden evals
   `agent_modified/` + `evals/` → `combined_grading/`

2. Run grader
   `combined_grading/grade.sh`

3. Save score + traces
   `results.json`

4. Separate evaluation script for Collaboration/team metrics on traces
   `results_extended.json`

---

**TODOs**

- Clean Up Pipeline for batch once working
- metrics
