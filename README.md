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
**TODOs**

- Evaluation on Eg Tasks - pass?
- Choose Subset of "top" TNI tasks
    - tasks from categories requiring different patterns
- In-depth Agent Metrics
- ...
- Better pipeline flow
